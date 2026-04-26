---
aliases:
title: buildSrc를 통한 멀티 프로젝트 공통 빌드로직 관리
---
`마지막 수정 일시: 2026년 4월 26일 토요일, 14시 (KST)`

## 공통 로직이 필요할 때

여러 프로젝트에 대한 `build.gradle.kts` 파일을 작성하다 보면 중복되는 부분이 생깁니다. 예를 들어 jar 파일을 생성하거나 manifest 파일을 쓰는 작업은 프로젝트별로 큰 차이가 없습니다. 같은 task 로직을 프로젝트마다 추가해야 한다면 귀찮은 것은 물론, 변경이 있을 때 해야 할 일도 많습니다. 

유틸리티 함수를 만들어서 여러 프로젝트에서 쓰고 싶은 경우도 있습니다. 빌드 과정에서 생기는 에러를 로그로 남기는 함수, Git 관련된 함수, 간단한 계산을 수행하는 함수 등, 여러 프로젝트에서 공통으로 쓸만한 함수를 어느 한 곳에 구현해 놓고 가져다 쓸 수 있으면 유용하겠지요?

이러한 공통 로직이 필요할 때, `buildSrc` 디렉토리를 사용해 보는 것은 어떨까요?
- [Gradle 공식 가이드 링크](https://docs.gradle.org/current/userguide/sharing_build_logic_between_subprojects.html)

## `buildSrc`: 공통 로직을 위한 특별한 디렉토리 이름

`buildSrc`는 공통 로직을 구현하기 위해 Gradle에서 특별히 예약된 디렉토리 이름입니다.

루트 폴더에 `buildSrc`라는 이름으로 폴더를 만들면, Gradle 실행 과정에서 이를 인식하고 다른 프로젝트를 빌드하기 전 `buildSrc` 프로젝트의 내용을 먼저 컴파일/빌드합니다. 이러한 점 덕분에 다른 프로젝트에서 공통 로직을 가져다 쓸 수 있게 되는 것이지요.

한 가지 유의할 점은 Gradle에 있어 `buildSrc` 자체도 하나의 프로젝트로 취급된다는 점입니다. 따라서 `buildSrc` 폴더 안에 `build.gradle(.kts)` 파일이 정의되어 있어야 하고, `src` 폴더가 있거나 빌드 스크립트에 `sourceSets`가 정의되어 있어야 합니다. 

#### 디렉토리 구성 예시

```
my-project              // 루트 디렉토리
|_ buildSrc
   |_ src 
   |  |_ main
   |     |_ java        // Java 소스코드
   |     |_ kotlin      // Kotlin 소스코드 또는 gradle.kts 컨벤션 플러그인 파일
   |_ build.gradle.kts  // buildSrc 빌드스크립트
```    

#### `build.gradle.kts` 예시

만약 Kotlin 소스코드와 플러그인을 작성하신다면 다음과 같이 `build.gradle.kts` 파일을 작성해 주시면 됩니다. `kotlin-dsl` 플러그인이 정확히 어떻게 동작하는지에 대해서는 저도 잘 알지 못하고 이 글에서는 다루지 않습니다.

```kotlin
// my-project/buildSrc/build.gradle.kts
plugins {
    `kotlin-dsl`
}

repositories {
    gradlePluginPortal()
}
```

제 로컬 환경에서 테스트 한 결과, `buildSrc`에 Java 소스코드만 있으면 `build.gradle.kts` 파일에 아무것도 작성하지 않아도 해당 로직의 컴파일과 다른 프로젝트 내 사용이 가능했습니다.

## 커스텀 플러그인의 구현과 적용


## 공통 함수의 구현과 사용


## 나가며


