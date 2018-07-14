---
layout: post
title: Github로 프로젝트 관리하기 Part2
subtitle: Part2 - CI & Test Coverage
catalog: true
header-img: 'https://i.imgur.com/vpKKHrD.jpg'
tags:
  - Github
  - Issue
  - Proejct Management
date: 2018-07-14 00:00:00
---


![](https://i.imgur.com/G5jo0Ty.png)

[GitHub Marketplace](https://github.com/marketplace/category/continuous-integration) Public Repository를 이용하면 대부분 무료로 이용 가능합니다. **본 포스팅에서는  CI는 Travis CI, Test Coverage는 Coveralls를 이용해서 진행하겠습니다.**

전체적인 플로우를 설명하는 것이 목적 이리서 특정 툴에 대한 직접적인 사용법을 다루지는 않겠습니다. 언어의 특성 및 개인에 기호에 맞는 제품을 사용하시면 됩니다.

## 전체 플로우
1. Pull Request 요청 -> Code Review 진행
2. Code Review 완료 -> 특정 Branch에 반영
3. 특정 Branch 수정시 자동 CI Build 작업 진행 -> 테스트 코드 실행
4. 테스트 커버지리 표시

## Pull Request & Code Review
![](https://i.imgur.com/q6HmT7o.png)

별다른 설정을 하지 않았다면 Pull Request를 요청할 경우 Travis에서 자동으로 해당 요청한 코드 기반으로 Build 작업이 진행됩니다. Build가 실패했을 경우는 Pull Request 요청자는 코드를 수정해서 최소한 Build가 된 코드 기반으로 Code Review를 진행하게 해야 됩니다(Build도 안 되는 코드를 리뷰할 이유는 없을 거 같습니다.)

요청받은 Pull Request에 대해서 Code Review 작업을 진행하게 됩니다. Code Review가 완료되면 Merge pull request를 통해서 해당 작업(issue)을 반영합니다.

## 테스트 커버지리 표시
![](https://i.imgur.com/U1ROYeE.png)

위에서 Merge pull request를 통해서 해당 작업(issue)을 반영했다면 Travis가 Build 할 때 작성된 Test Code 기반으로 Coverage 정보를 위처럼 자동으로 코멘드를 추가해줍니다.

누군가가 테스트 코드를 작성하지 않았다면 `Change from base` 항목에서 - 표시가 됩니다. **이렇게 해당 작업마다 커버리지를 표시하는 것이 전체 커버리지를 높이고 그 값을 유지하는 좋은 방법이라고 생각합니다.**