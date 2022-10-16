---
title: "[DeliveryFood] Spring Security 적용(2)"
excerpt: "Spring Security를 적용해보기"

categories:
  -  DeliveryFood
tags:
  - [Project, DeliveryFood]

toc: true
toc_sticky: true
 
date: 2022-10-16
last_modified_at: 2022-10-16
---

Spring Security를 적용하며 가장 중요하다고 느낀 키워드 두가지는 `UsernamePasswordAuthenticationFilter와 AuthenticationProvider` 였다.

## UsernamePasswordAuthenticationFilter
스프링에서 Filter는 DispatcherServlet에 전달되기 전에 주로 '인증, 권한 체크'등을 하는데에 사용한다. 요청을 채가서 먼저 처리를 한다는 의미에서 Filter 비슷한 역할을 하는 Interceptor도 있지만, Interceptor와 Filter는 실행 시점이 다르다. 
Web Context 단에서 동작하는 Filter가 Sping Context 단에서 동작하는 Interceptor보다 먼저 실행된다. 그래서 어떤 api를 타기전에 가장 먼저 실행되는 Filter에서 인증, 권한 체크를 하는 것 같다.

UsernamePasswordAuthenticationFilter는 앞에서 알아본 것처럼 Spring Security가 제공하는 특별한 Filter중 하나이다. ID/PW form 기반 인증에 사용하는 URL 요청을 감시하고, 요청이 감지되면 인증을 진행한다.<br>

AuthenticationManager를 통해 인증을 실행하고<br>
(1) 인증 성공시, 얻은 Authentication객체를 SecurityContext에 저장 후 AuthenticationSuccessHandler가 실행된다.<br>
(2) 인증 실패시, AuthenticationFailureHandler가 샐행된다.<br>

``` java
@RequiredArgsConstructor
public class CustomAuthenticationFilter extends UsernamePasswordAuthenticationFilter {

    private final AuthenticationManager authenticationManager;

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
        if(request.getParameter("username") == null || request.getParameter("password") == null)
            return null;

        UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(
                        request.getParameter("username"),
                        request.getParameter("password"));

        setDetails(request, authenticationToken);
        return authenticationManager.authenticate(authenticationToken);
    }

    @Override
    protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain,
                                            Authentication authResult) throws IOException, ServletException {
        System.out.println("successfulAuthentication");

        CustomUserDetails userDetails = (CustomUserDetails) authResult.getPrincipal();
        if(userDetails.getAuthority().equals("ROLE_NOT_AUTH")) {
            response.setStatus(HttpStatus.TEMPORARY_REDIRECT.value());
            // response에 리다이렉트 URI 추가 가능
        }
    }
}
```

### attemptAuthentication
attemptAuthentication 메소드는 request로 들어온 username과 password를 가져와서 UsernamePasswordAuthenticationToken을 생성한다.
생성된 UsernamePasswordAuthenticationToken은 AuthenticationManager를 통해 인증을 진행하도록 한다.

AuthenticationManager는 토큰을 받아 인증하고, Authentication을 리턴하도록 하는 메소드를 가지고 있는 인터페이스이다.
그리고 이 `AuthenticationManager는 구현체인 ProviderManager를 통해 인증이 진행`된다. 실제 인증은 ProviderManager의 AuthenticationProvider인터페이스를 통해 처리되게 되는데, ProviderManager의 생성자에서 List<AuthenticationProvider>를 받아와 리스트를 순회하게 된다. 
AuthenticationProvider의 support()가 true라면, 해당 `AuthenticationProvider의 authenticate()를 실행`하여 실제 인증을 진행하게 된다.<br>
즉, 실제 인증은 AuthenticationProvider에서 진행되므로 AuthenticationProvider를 프로젝트에 맞게 적절히 구현하는 과정이 필요하다.

## AuthenticationProvider
AuthenticationProvider는 DB에서 가져온 정보와 Request로 넘어온 정보를 비교하는 로직이 포함되어있는 인터페이스다.<br>
AuthenticationProvider의 authenticate 메소드는 AuthenticationManager에서 넘겨준 Authentication객체를 가지고 ID와 PW를 조회하고 비밀번호를 비교한다.
authenticate 메소드의 리턴타입은 UsernamePasswordAuthenticationToken으로, Authentication객체를 가지고 토큰을 생성해 리턴한다.

```java
@RequiredArgsConstructor
public class CustomAuthenticationProvider implements AuthenticationProvider {

    private final MemberService memberService;
    private final BCryptPasswordEncoder passwordEncoder;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String username = (String) authentication.getPrincipal();
        String password = (String) authentication.getCredentials();
        CustomUserDetails user = memberService.loadUserByUsername(username);

        if (!passwordEncoder.matches(password, user.getPassword())) {
            throw new BadCredentialsException(user.getUsername() + " Invalid password");
        }

        return new UsernamePasswordAuthenticationToken(user, password, user.getAuthorities());
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return authentication.equals(UsernamePasswordAuthenticationToken.class);
    }
}
```
이전에 attemptAuthentication()에서 토큰을 만들어 AuthenticationManager에게 넘겨줬다. 그리고 이 AuthenticationManager를 통해 AuthenticationProvider에서 실제 인증 과정이 이루어 진다는 것까지 알아봤었다.
코드를 살펴보면 넘어온 토큰에서 username과 password를 가져와 DB에 있는 유저 인지 확인하고, DB에 있는 유저의 password와 비교하는 과정을 거친다. 이 과정을 위해 또 `UserDetailsService의 loadUserByUsername()`를 구현해서 생성되는 토큰의 principal에 세팅해주게 해야 한다. 그래야 추후 Spring Security의 세션 기능을 사용할 때, pricipal을 가져와 로그인되어 있는 유저의 정보를 알 수 있게 된다.

```java
    @Override
    public CustomUserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        MemberDto memberDto = memberDao.findByEmail(username);
        if(memberDto == null) {
            throw new NullPointerException("Member DB에 member가 존재하지 않음 : " + username);
        }

        CustomUserDetails userDetails = CustomUserDetails.builder()
                .userId(memberDto.getUserId())
                .email(memberDto.getEmail())
                .password(memberDto.getPassword())
                .status(memberDto.getStatus())
                .authority(memberDto.getRole())
                .build();

        return userDetails;
    }
```
loadUserByUsername에서는 파라미터로 넘어온 username을 가지고 DB에서 유저를 찾아오게 된다. 여기서 DB에 유저의 정보가 없다는 것은, 회원 가입을 하지 않았다는 것으로 FailHandler를 타게 될 것이고 Handler에서 리다이렉트를 통해 회원가입으로 넘기게 할 수 있을 것이다.<br>
다시 돌아와서, username을 가지고 DB에서 찾아온 MemberDto를 기반으로 CustomUserDetails를 생성한다. `CustomUserDetails는 Spring Security에서 사용자의 정보를 담는 용도로 사용되는데, 이 인터페이스를 구현하게 되면 Security에서의 인증, 인가 작업에서 구현한 클래스를 기반으로 작업할 수 있다.` 나는 username과 password만 필요한게 아니어서 다양한 유저의 정보를 담을 수 있도록 따로 CustomUserDetails를 구현해주었다.

해당 과정까지 구현해 주었다면 Spring Security의 인증을 사용할 수 있는 상태가 된다.

## 보완해야할 점
일단 Spring Security의 인증을 사용할 수 있게 해주었지만, 아직 테스트가 부족하다. 대부분의 테스트가 로그인한 유저의 정보를 기반으로 작성되는데, 테스트를 할 때마다 로그인을 한 채로(?) 진행하기에는 어려움이 있다. 때문에 로그인한 유저로 테스트할 수 있는 환경을 만들어 줘야 한다. 
또한 아직 디테일한 작업이 남아있다. 로그라던지, 실패시 unsuccessfulAuthentication를 구현해주지 않았다. 구현에만 급급했기 때문에 가장 기본적으로 되어야 할 로그를 작성해주지 않았는데, 추후 문제가 될만한 포인트나 확인이 되어야할 포인트에 로그를 추가할 예정이다.

Spring Security를 구현해보며 Spring의 Filter가 어떤식으로 동작하고, 어떻게 컨트롤러에 들어오게 되는지 좀 더 자세히 알게 되었다. 처음 접해보는 개념들이 대부분이라 이해하는데 시간이 좀 걸렸지만 결국 중점이 되는 것은 UsernamePasswordAuthenticationFilter와 AuthenticationProvider라는 걸 알고나서는 쉽게 구현할 수 있었던 것 같다. 앞으로 시간이 나는대로 인증 관련해 부족한 부분을 보완해야겠다.