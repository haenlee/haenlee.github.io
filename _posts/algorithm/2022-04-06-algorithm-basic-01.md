---
title:  "[알고리즘] 모두의 약수"
excerpt: ""

categories:
  - Algorithm
tags:
  - [Algorithm]

toc: true
toc_sticky: true
 
date: 2022-04-06
last_modified_at: 2022-04-06
---

## 🔎 문제
자연수 N이 입력되면 1부터 N까지의 각 숫자들의 약수의 개수를 출력하는 프로그램을 작성하세요. 
만약 N이 8이 입력된다면 1(1개), 2(2개), 3(2개), 4(3개), 5(2개), 6(4개), 7(2개), 8(4개) 와 같이 각 숫자의 약수의 개수가 구해집니다.
출력은 다음과 같이 1부터 차례대로 약수의 개수만 출력하면 됩니다. 
1 2 2 3 2 4 2 4 와 같이 출력한다.

### 입력 설명
첫 번째 줄에 자연수 N(5<=N<=50,000)가 주어진다.

### 출력 설명
첫 번째 줄에 1부터 N까지 약수의 개수를 순서대로 출력한다.

## 💡 풀이
N제한을 유의해야 한다.
아래 소스코드를 적용하면, i가 30,000이고 j가 1,000이라면 j는 30번만 돌면 된다.

### 📍 j의 배수 활용
j가 2라면, 2를 약수로 가지고 있는 숫자는 2의 배수이다.

## 📃 소스코드
```c++
#include<iostream>
#include<vector>
using namespace std;

int main()
{
	int n;
	cin >> n;

	vector<int> a(n + 1, 1);
	for(int i = 2; i <= n; i++)
	{
		for(int j = i; j <= n; j += i)
		{
		  a[j]++;
		}
	}

	for(int i = 1; i <= n; i++)
		cout << a[i] << ' ';

	return 0;
}
```