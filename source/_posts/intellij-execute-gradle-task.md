---
layout: post
title: 인텔리제이 Execute Gradle task로 Gradle를 쉽게 사용하자
catalog: true
header-img: 'https://i.imgur.com/avC1Xor.jpg'
tags:
  - IntelliJ
  - Gradle
date: 2019-10-17 00:16:46
subtitle:
---

IntellIj를 이용하면 Gradle Task의 명령어 assistant 해주는 기능을 통해서 보다 쉽게 Gradle을 사용할 수 있습니다. 

## 설정 방법
![](https://github.com/cheese10yun/IntelliJ/raw/master/assets/execute-gradle-task.png)

Find Action에서 `Execute Gradle Task`을 열어 볼 수 있습니다.

![](https://github.com/cheese10yun/IntelliJ/raw/master/assets/gradle-tasks-hot-key.png)

단축키를 지정해서 사용하는 것을 권장드립니다. `Keymap` -> `Execute Gradle Task` -> `Hot Key` 지정
저같은 경우에는 `CMD + 0`으로 지정해서 사용하고 있습니다.

## 사용법

![](https://github.com/cheese10yun/IntelliJ/raw/master/assets/gradle-task-run-1.gif)

`Execute Gradle Task`를 실행하면 위 화면 처럼 해당 Gradle 명령어를 assistant 해줍니다.

![](https://github.com/cheese10yun/IntelliJ/raw/master/assets/costom-build.png)

![](https://github.com/cheese10yun/IntelliJ/raw/master/assets/gradle-task-run-2.gif)
위 그림처럼 커스텀하게 등록한 task도 assistant 해줍니다.