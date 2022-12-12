---
title: "[DeliveryFood] 부하 분산을 위한 데이터베이스 이중화 작업 - MySQL Replication"
excerpt: "프록시 객체와 지연화를 통해 데이터베이스 이중화하기"

categories:
  -  DeliveryFood
tags:
  - [Project, DeliveryFood, DB, Replication]

toc: true
toc_sticky: true
 
date: 2022-12-11
last_modified_at: 2022-12-11
---

# 🚀 Replication을 적용해야 하는 이유
---
대용량 트래픽이 요청되는 상황에서 안정적인 운영을 하기 위해서는 다중화, 부하분산, 튜닝과 같은 방법들을 사용할 수 있다. 그 중 데이터베이스의 최적화는 아주 중요한데 OS나 웹서버를 튜닝해도 결국 데이터베이스에서 병목이 발생하면 시스템 전체의 성능 저하가 발생할 수도 있다.

데이터베이스의 성능을 최적화하는 방법에는 인덱싱이나 N+1 문제 해결등 쿼리 튜닝을 할 수 있지만, 데이터베이스를 다중화하여 읽기와 쓰기를 분리한다면 데이터베이스의 병목을 없애며 성능을 향상시킬 수 있다.

## 📝 Scale-out(수평 확장)
읽기와 쓰기, 수정(CUD)과 같은 모든 연산이 하나의 DB에서 일어나게 되면 트래픽이 늘어남에 따라 병목 현상이 생길 수 있다. 이 병목 현상을 해결하기 위해 한대의 source(master) 서버에서 쓰기를 담당하고, 여러대의 replica(slave)에서 읽기를 담당하게하여 부하를 분산할 수 있다. 

여러대의 replica(slave)를 둔다는 점에서 Scale-out 방식을 적용한 것이라고 볼 수 있는데, 서버의 성능을 향상시키기 위해 성능을 증가 시키는 것이 아니라 서버의 개수를 증가 시키는 것을 말한다. 일반적인 데이터베이스 레플리케이션을 통해 애플리케이션 서버뿐만 아니라 데이터베이스의 서버도 수평적으로 확장할 수 있다.

## 📝 데이터 백업
Replication은 데이터를 동기화하기 때문에 백업에도 활용이 가능하다. 하지만 데이터베이스가 애플리케이션의 요청을 처리하면서, 백업 과정을 동시에 수행하기 때문에 백업 프로그램을 통해 정기적으로 데이터베이스를 백업하는 방식을 사용하면 서비스에 영향을 줄 수 있다.

백업을 위해서는 레플리카 서버는 실시간으로 동기화 되도록 해주고 레플리카 서버에 스케줄러를 실행하여 간격을 두고 주기적으로 백업하는 구조를 구축하도록 해야 한다. 레플리케이션으로 백업된 데이터베이스 서버는 원본 서버에 문제가 생겼을 때 사용될수도 있다.

## 📝 데이터 분석
데이터 분석을 위한 쿼리는 많은 양의 조회 연산과 집계 연산을 사용하기 때문에 데이터베이스 서버에 부하를 준다. 실제 서비스의 데이터베이스에서 이런 쿼리를 수행하게 되면 실 서비스 성능에 영향을 미칠 수 있다.

따라서 Replication으로 데이터베이스를 이중화한 후, 복제 서버에서 분석 쿼리를 수행하게 하면 실 서비스에 영향을 끼치지 않을 수 있다.

## 📝 데이터의 지리적 분산
전세계적으로 운영되는 서비스의 경우 레이턴시와 같이 지리적 이유로 발생하는 이슈들이 많다. 따라서 각 국가별로 복제 서버를 둔다면 Replication을 통한 동기화로 문제를 해결할 수 있다. CDN과 비슷하다.

<br>
# 🚀 Replication 구조
---
![image](https://user-images.githubusercontent.com/85219306/207068093-9c072a86-e3b9-4293-8c10-76a6178f35c2.png)
1. 클라이언트가 Commit을 누르면 먼저 Master 서버에 존재하는 Binary log에 변경사항을 모두 기록한다.
2. Master Thread는 비동기적으로 Binary log를 읽어 Slave 서버로 전송한다.
3. Slave의 I/O Thread는 Master로부터 받은 변경 데이터들을 Relay log에 기록한다.
4. Slave의 SQL Thread는 Relay log의 기록들을 읽어 자신의 스토리지 엔진에 최종 적용한다.

## 📝 바이너리 로그 덤프 쓰레드 (Binary Log Dump Thread)
바이너리 로그 덤프 쓰레드는 레플리케이션 작업이 레플리카 서버로부터 요청 되었을 때 소스 서버로부터 생성된다. 바이너리 로그 덤프 쓰레드는 바이너리 로그의 이벤트를 읽고, 레플리카 서버로 전송하는 역할을 수행한다. 이벤트를 읽을 때 바이너리 로그 파일에 대해 잠금을 수행하고, 읽기 작업이 끝나면 잠금을 즉시 해제한다.

## 📝 레플리케이션 I/O 쓰레드 (Replication I/O Thread)
레플리카 서버에서 레플리케이션 작업이 시작되면, 레플리케이션 I/O 쓰레드를 생성하며, 소스 서버로부터 바이너리 로그를 가져온다. 가져온 이벤트들은 레플리카 서버의 릴레이 로그(relay log)에 저장한다. Relay는 중계라는 의미이다. 즉, 소스 서버와 레플리카 서버를 중계하는 로그라는 의미이다. 레플리케이션 I/O 쓰레드는 오직 바이너리 로그를 읽고(input), 릴레이 로그를 쓰는 작업(output)만 수행한다.

## 📝 레플리케이션 SQL 쓰레드 (Replication SQL Thread)
레플리케이션 I/O 쓰레드가 릴레이 로그를 작성하면, 레플리케이션 SQL 쓰레드는 릴레이 로그에 기록된 이벤트를 읽고 실행하는 역할을 수행한다.

<br>
# 🚀 적용하기
---
## 📝 Master DataSource, Slave DataSource 빈 등록
```java
@Bean(name = "masterDataSource")
@ConfigurationProperties(prefix = "spring.datasource.master")
public DataSource masterDataSource() {
    return DataSourceBuilder.create()
            .type(HikariDataSource.class)
            .build();
}

@Bean(name = "slaveDataSource")
@ConfigurationProperties(prefix = "spring.datasource.slave")
public DataSource slaveDataSource() {
    return DataSourceBuilder.create()
            .type(HikariDataSource.class)
            .build();
}
```

## 📝 RoutingDataSource 설정
```java
@Bean(name = "routingDataSource")
public DataSource routingDataSource(@Qualifier(value = "masterDataSource") DataSource masterDataSource,
                                    @Qualifier(value = "slaveDataSource") DataSource slaveDataSource) {
    ReplicationRoutingDataSource routingDataSource = new ReplicationRoutingDataSource();
    Map<Object, Object> targetDataSourcesMap = new HashMap<>();

    targetDataSourcesMap.put("master",masterDataSource);
    targetDataSourcesMap.put("slave", slaveDataSource);

    routingDataSource.setTargetDataSources(targetDataSourcesMap);
    routingDataSource.setDefaultTargetDataSource(masterDataSource);

    return routingDataSource;
}
```

## 📝 determineCurrnetLookupKey() 오버라이딩 (master, slave 분기)
```java
@Slf4j
public class ReplicationRoutingDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        log.trace("[ReplicationRoutingDataSource] determineCurrentLookupKey=" + 
        (TransactionSynchronizationManager.isCurrentTransactionReadOnly() ? "slave" : "master"));

        return TransactionSynchronizationManager.isCurrentTransactionReadOnly() ? "slave" : "master";
    }
}
```

## 📝 ProxyDataSource 설정
### 문제 발생
트랜잭션이 readOnly인지를 판단하고 분기해서 얻은 DataSource로부터 connection을 얻어야 하는데 readOnly인지를 판단하기 전에 connection을 얻으려고 한다.  
이를 해결하기 위해 connection 시점을 조정할 필요가 있다. connection을 트랜잭션 시작할 때가 아닌 쿼리를 보낼 때 얻을 수 있도록 시점을 바꿔야 한다.

![image](https://user-images.githubusercontent.com/85219306/207074459-3e987d1a-98d3-4cfc-bf7f-bf49c9579e0e.png)

### 문제 해결
```java
@Bean(name = "dataSource")
public DataSource dataSource(@Qualifier("routingDataSource") DataSource routingDataSource) {
    return new LazyConnectionDataSourceProxy(routingDataSource);
}
```
위와 같이 TransactionManager에게 DataSource를 보낼 때 먼저 프록시로 한번 더 감싸서 LazyConnection으로 바꿔서 전달해준다.

```java
@Bean
public PlatformTransactionManager transactionManager(@Qualifier("dataSource") DataSource dataSource) {
    return new DataSourceTransactionManager(dataSource);
}
```

마지막으로 SqlSessionFactory에 dataSource를 전달해주고 작업을 마무리해준다.

```java
@Bean(name = "sqlSessionFactory")
@Primary
public SqlSessionFactory sqlSession(
        @Qualifier("dataSource") DataSource dataSource,
        ApplicationContext applicationContext
) throws Exception {
    SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
    sqlSessionFactoryBean.setDataSource(dataSource);
    sqlSessionFactoryBean.setMapperLocations(applicationContext.getResources("classpath:/mybatis/**/*.xml"));
    return sqlSessionFactoryBean.getObject();
}
```

