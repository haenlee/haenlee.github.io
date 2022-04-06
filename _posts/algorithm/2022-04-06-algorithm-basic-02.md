---
title:  "[알고리즘] 숫자의 총 개수"
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
자연수 N이 입력되면 1부터 N까지의 자연수를 종이에 적을 때 각 숫자는 몇 개 쓰였을까요?
예를 들어 1부터 15까지는 1, 2, 3, 4, 5, 6, 7, 8, 9, 1, 0, 1, 1, 1, 2, 1, 3, 1, 4, 1, 5으로 총 21개가 쓰였음을 알 수 있습니다. 
자연수 N이 입력되면 1부터 N까지 각 숫자는 몇 개가 사용되었는지를 구하는 프로그램을 작성하세요.

### 입력 설명
첫 번째 줄에는 자연수 N(3<=N<=100,000,000)이 주어진다.

### 출력 설명
첫 번째 줄에 숫자의 총개수를 출력한다.

## 💡 풀이
N제한을 유의해야 한다.

### 📍 자릿수
1 ~ 9 : 1<br>
10 ~ 99 : 2<br>
100 ~ 999 : 3<br>

이렇게 자릿수가 증가하는 규칙을 찾으면 각 자리의 숫자를 나누면서 더하지 않아도 쉽게 개수를 찾을 수 있다.<br>
start는 1부터, end는 1 * 10 - 1 이라는 것을 파악하면 그 후 연산은 쉽게 할 수 있다.<br>
꼭 자릿수가 1, 2, 3 순으로 증가한다는 것을 알아두자!

## 📃 소스코드
```c++
#include<iostream>
using namespace std;

int main()
{
	int n;
	cin >> n;

	int start = 0, end = 0;
	int sum = 0;
	for(int i = 1, digit = 1; end < n; i *= 10, digit++)
	{
		start = i;
		end = (i * 10) - 1;
		if(end > n)
			end = n;

		sum += (end - start + 1) * digit;
	}

	cout << sum;

	return 0;
}
```