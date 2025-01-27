---
layout: post
title: "[COMAtching] 매칭 서비스 개발기-1"
date: 2025-01-01 13:35
category: [COMAtching, TIL]
author: [greensnapback0229]
tags: []
summary: 코매칭 서비스 개발기
---


> 코매칭 프로젝트를 하면서 매칭 기능을 담당해 계속 고도화 시킨 경험을 남기고자 합니다.
Ver1부터 3까지 고민했던 비즈니스 로직과 기술적 고민들에 관한 내용입니다.


# Ver1 매칭
Ver1에서는 정말 단순하게 crud 밖에 할줄 모르던 시기였고 그냥 이렇게 귀엽게 개발했었다는 내용입니다.

## Ver1 매칭 로직
Ver1은 MBTI 기반의 매칭이었습니다. 원하는 상대의 MBTI 중 E/I와 P/J를 골라서 해당하는 MBTI를 가진 유저를 랜덤으로 매칭해주었습니다.

```java
if (ei.equals("Z") && jp.equals("X")) {
			candidate = userInfoRepository.findByGenderAndChooseGreaterThan(gender, 0);
		} else if (jp.equals("X")) {
			candidate = userInfoRepository.findByMbtiStartingWithAndGenderAndChooseGreaterThan(ei, gender, 0);
		} else if (ei.equals("Z")) {
			candidate = userInfoRepository.findByMbtiEndingWithAndGenderAndChooseGreaterThan(jp, gender, 0);
		} else {
			candidate = userInfoRepository.findByMbtiStartingWithAndMbtiEndingWithAndGenderAndChooseGreaterThan(ei, jp,
				gender, 0);
		}
```
DB 설계도 부족했고 ORM을 부턱대고 쓰다보니 이렇게 되어버렸다.. 

## Ver2 매칭

