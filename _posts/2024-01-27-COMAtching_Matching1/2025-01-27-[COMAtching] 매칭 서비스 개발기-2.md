---
layout: post
title: (COMAtching) 매칭 서비스 개발기-2
date: 2025-01-01 13:35
category: [COMAtching, Troubleshooting]
author: [greensnapback0229]
tags: []
summary: 코매칭 서비스 개발기
---

> 코매칭 프로젝트를 하면서 매칭 기능을 담당해 계속 고도화 시킨 경험을 남기고자 합니다.
> Ver1부터 3까지 고민했던 비즈니스 로직과 기술적 고민들에 관한 내용입니다.

# Ver3 매칭

`Ver1`, `Ver2`에서 받은 사용자들의 피드백을 바탕으로 매칭시 관리자를 거치지 않고 사용자가 직접 매칭할 수 있는 flow로 서비스를 고도화 시켰습니다.
`Ver3`로 오면서 AI의 과중화로 인해 EC2 프리티어에 내장할 수 없었고 `AI`, `BE`서버를 분리하게 되었습니다.

<a href="https://greensnapback0229.github.io/posts/COMAtching_architecture/"> > 서버 분리 관련 글 </a>

## 기능 복잡도..

서버가 분리되면서 구현도 경량화 되었을까?라고 한다면 절대 `NO` 였습니다.
CSV의 Read/Write를 하지 않아도 됐지만 결국 원천 DB를 다루는 사이드에서 유저 정보의 동기화 요청을 수시로 보내야 했습니다.
1건에 매칭에 대해서 다음과 같은 비즈니스 로직을 생각했습니다.

1. 사용자가 매칭 요청  
   1-1. 사용자의 잔여 포인트 체크

2. 사용자 정보를 담아 AI 서버로 요청 및 응답 확인
   2-1. 학교에서 얻을 수 있는 많은 사람들의 데이터를 받을
3.

![ver3_matching](/assets/comatching3_matching_sequence.png)

## RabbitMQ 정책 및 확장성 고려

## RabbitMQ 도입

User Crud Queue RPC에서 얻을 수

## CorrelationId
