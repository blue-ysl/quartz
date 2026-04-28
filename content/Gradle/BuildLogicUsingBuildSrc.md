---
aliases:
title: buildSrc를 통한 멀티 프로젝트 공통 빌드로직 관리
---
`마지막 수정 일시: 2026년 4월 29일 수요일, 01시 (KST)`

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

##### Java 소스 빌드 관련 팁

> 제 로컬 환경에서의 테스트 결과, `buildSrc`에 Java 소스코드만 있으면 `build.gradle.kts` 파일에 아무것도 작성하지 않아도 해당 로직의 컴파일과 다른 프로젝트 내 사용이 가능했습니다.

## 실습: 공통 함수의 구현과 사용

`buildSrc` 폴더에 공통 함수를 구현하고 사용하는 예시를 보여 드리겠습니다. 현재 일자를 String 타입으로 반환하는 Java 함수를 추가해 보겠습니다.

우선 `gradle init --dsl kotlin`으로 Gradle 프로젝트를 생성해 줍니다. (IntelliJ 같은 IDE에서 지원하는 Gradle Java/Kotlin 프로젝트를 사용하셔도 상관 없습니다)

```
blue@Bluebook:~/Gradle/my-project$ gradle init --dsl kotlin

Select type of build to generate:
  1: Application
  2: Library
  3: Gradle plugin
  4: Basic (build structure only)
Enter selection (default: Application) [1..4] 4

Project name (default: my-project): my-project

Generate build using new APIs and behavior (some features may change in the next minor release)? (default: no) [yes, no] no


> Task :init
Learn more about Gradle by exploring our Samples at https://docs.gradle.org/9.4.1/samples

BUILD SUCCESSFUL in 11s
1 actionable task: 1 executed
blue@Bluebook:~/Gradle/my-project$ l
total 32
-rw-r--r-- 1 blue blue  200 Apr 27 22:04 build.gradle.kts
drwxr-xr-x 3 blue blue 4096 Apr 27 22:04 gradle
-rw-r--r-- 1 blue blue  194 Apr 27 22:04 gradle.properties
-rwxr-xr-x 1 blue blue 8654 Apr 27 22:04 gradlew
-rw-r--r-- 1 blue blue 2896 Apr 27 22:04 gradlew.bat
-rw-r--r-- 1 blue blue  347 Apr 27 22:04 settings.gradle.kts
```

`buildSrc` 폴더를 만들고 그 안에 `build.gradle.kts` 파일, `src` 폴더를 추가해 줍니다.

```
blue@Bluebook:~/Gradle/my-project$ mkdir buildSrc
blue@Bluebook:~/Gradle/my-project$ touch buildSrc/build.gradle.kts
blue@Bluebook:~/Gradle/my-project$ mkdir -p buildSrc/src/main/java
blue@Bluebook:~/Gradle/my-project$ mkdir -p buildSrc/src/main/kotlin
```

그렇게 하고 나면 이런 구조가 되어 있을 거예요.

```
blue@Bluebook:~/Gradle/my-project$ tree buildSrc/
buildSrc/
├── build.gradle.kts
└── src
    └── main
        ├── java
        └── kotlin
        
blue@Bluebook:~/Gradle/my-project$ tree
.
├── build.gradle.kts
├── buildSrc
│   ├── build.gradle.kts
│   └── src
│       └── main
│           ├── java
│           └── kotlin
├── gradle
│   ├── libs.versions.toml
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradle.properties
├── gradlew
├── gradlew.bat
└── settings.gradle.kts

8 directories, 9 files       
```

이제 공통 함수를 추가해 봅시다.

```java
// my-project/buildSrc/src/main/java/JavaUtil.java
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class JavaUtil
{
    public static String javaCurrentDate()
    {
        LocalDateTime current = LocalDateTime.now();
        return "Java date: " + current.format(DateTimeFormatter.ofPattern("yyyy-MM-dd"));
    }
}
```

```kotlin
// my-project/buildSrc/src/main/kotlin/KotlinUtil.kt
import java.time.LocalDateTime
import java.time.format.DateTimeFormatter

fun kotlinCurrentDate(): String {
    val current = LocalDateTime.now()
    val formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd")
    return "Kotlin date: " + current.format(formatter)
}
```

`buildSrc` 폴더의 `build.gradle.kts` 파일도 추가합니다.

```kotlin
// my-project/buildSrc/build.gradle.kts
plugins {
    `kotlin-dsl`
}

repositories {
    gradlePluginPortal()
}

```

그 다음, `buildSrc` 프로젝트를 빌드합니다.

```
blue@Bluebook:~/Gradle/my-project$ ./gradlew buildSrc:build
Calculating task graph as configuration cache cannot be reused because file 'build.gradle.kts' has changed.

> Task :buildSrc:jar
:jar: No valid plugin descriptors were found in META-INF/gradle-plugins

BUILD SUCCESSFUL in 1s
6 actionable tasks: 6 executed
Configuration cache entry stored.
```

이후 `buildSrc` 폴더를 보면, 매우 많은 빌드 결과물이 생성된 것을 확인할 수 있습니다.
이제 이 함수를 본 프로젝트의 빌드 스크립트에서 활용해 봅시다.

루트 디렉토리에 있는 `build.gradle.kts` 파일에, 'hello'라는 이름의 task를 추가하는 내용을 작성합니다. 내용은 아주 단순한데, 위에서 작성한 `buildSrc` 함수를 호출해서 그 결과물을 출력하는 것이 전부입니다.

```kotlin
// my-project/build.gradle.kts
tasks.register("hello") {
    doLast {
        val nowKotlin = kotlinCurrentDate();
        println("The current date: $nowKotlin")

        val nowJava = JavaUtil.javaCurrentDate();

        println("The current date: $nowJava")
    }
}
```

그 다음, `hello` 태스크를 실행해 봅시다.

```
blue@Bluebook:~/Gradle/my-project$ ./gradlew hello
Calculating task graph as no cached configuration is available for tasks: hello

> Task :hello
The current date: Kotlin date: 2026-04-27
The current date: Java date: 2026-04-27

BUILD SUCCESSFUL in 523ms
5 actionable tasks: 1 executed, 4 up-to-date
Configuration cache entry stored.
```

`buildSrc`에서 만든 함수들이 빌드스크립트에서 호출되어 동작한 것을 확인할 수 있습니다.

## 실습: 컨벤션 플러그인의 구현과 적용

다음으로는 `buildSrc`를 이용해서 컨벤션 플러그인을 만들어 보겠습니다. 

> 컨벤션 플러그인(convention plugin)이란, 다른 프로젝트에서 마치 플러그인처럼 가져다 쓸 수 있는 빌드스크립트로 생각해 주셔도 무방합니다. 

Java 프로젝트에 적용될 수 있는 간단한 플러그인을 만들어 보겠습니다.

> **컨벤션 플러그인의 확장자**는 `gradle(.kts)`여야 한다는 점에 주의해 주세요.

우선 `my-java-task-conventions.gradle.kts` 파일을 추가합니다. 이 플러그인에는 다음과 같은 내용이 정의되어 있습니다.
- 이 플러그인을 사용하는 프로젝트는 Java 17 버전의 툴체인을 사용함.
- 이 플러그인은 Java 플러그인의 'jar'  task를 확장하며, 각 프로젝트에 정의된 프로젝트명, 메인 클래스, 프로젝트 버전 값을 가지고 jar 파일을 작명하고 MANIFEST 파일을 작성함.

```kotlin
// my-project/buildSrc/src/main/kotlin/my-java-task-conventions.gradle.kts
plugins {  
    java  
}  
  
repositories {  
    mavenCentral()  
}  

// Java 버전을 정의합니다
java {  
    toolchain {  
        languageVersion = JavaLanguageVersion.of(17)  
    }  
}  
  
// java 플러그인의 'jar' task 내용을 확장합니다.
tasks.named<Jar>("jar") {  
    // 이 플러그인을 적용하는 프로젝트의 build.gradle.kts에 정의되어 있는 값을 가져옵니다.
    val projectJarName: String by project  
    val mainClass: String by project  
    val projectVersion: String by project  
  
    // JAR 파일의 이름을 지정합니다.
    archiveFileName.set("${projectJarName}.jar")  
  
    // MANIFEST.MF 파일에 적을 내용을 정의합니다.
    manifest {  
        attributes("Main-Class" to mainClass)  
        attributes("Module-Version" to projectVersion)  
        attributes("Build-Released" to kotlinCurrentDate())  
    }
}
```

이제, 실제로 이 플러그인을 활용할 프로젝트가 필요하겠죠. 루트 디렉토리의 `src` 디렉토리에 Java 소스코드를 추가합니다.

```java
// my-project/src/main/java/com/blue/Main.java
package com.blue;  
  
public class Main  
{  
    public static void main(String[] args)  
    {  
        System.out.println("Hello World, this is an example Gradle project!");    
    }  
}
```

이제 마지막으로 `my-java-task-conventions`를 프로젝트의 `build.gradle.kts`에 적용할 일만 남았는데요, 다음과 같은 내용을 추가해 주시면 됩니다.

```kotlin
// my-project/build.gradle.kts

/* 프로젝트 외부에 노출할 변수를 정의합니다. 이렇게 해야 플러그인에서 해당 변수 내용을
   참조할 수 있다고 생각해 주시면 됩니다. */
val projectJarName by extra("myGradleProject")
val mainClass: String by extra("com.blue.Main")
val projectVersion: String by extra("1.2")

/* 방금 만든 'my-java-task-conventions'를 적용합니다. */
plugins {
    id("my-java-task-conventions")
}
```

모든 준비가 끝났으니, Gradle을 실행해 볼까요? (최초 실행 여부 등에 따라 메시지는 다를 수 있음)

```
blue@Bluebook:~/Gradle/my-project$ ./gradlew clean jar
Calculating task graph as no cached configuration is available for tasks: clean jar

BUILD SUCCESSFUL in 3s
13 actionable tasks: 8 executed, 5 up-to-date
Configuration cache entry stored.
```

그렇게 하면, `build/libs` 폴더에 `myGradleProject.jar` 파일이 생성된 것을 확인할 수 있고,

```
blue@Bluebook:~/Gradle/my-project$ ls -al build/libs
total 12
drwxr-xr-x 2 blue blue 4096 Apr 29 00:50 .
drwxr-xr-x 7 blue blue 4096 Apr 29 00:50 ..
-rw-r--r-- 1 blue blue  991 Apr 29 00:50 myGradleProject.jar
```

MANIFEST.MF 파일에도 우리 플러그인이 의도한대로 내용이 작성된 것을 확인할 수 있습니다.

```
blue@Bluebook:~/Gradle/my-project$ unzip -p build/libs/myGradleProject.jar META-INF/MANIFEST.MF
Manifest-Version: 1.0
Main-Class: com.blue.Main
Module-Version: 1.2
Build-Released: Kotlin date: 2026-04-29
```

> `unzip` 커맨드의 `p` 옵션(파이프 옵션)을 활용하면 파일을 풀어헤치지 않고 내용을 확인할 수 있습니다.

## 컨벤션 플러그인을 활용하면 좋은 점

여러 프로젝트에 공통적으로 적용할 수 있는 컨벤션 플러그인을 사용하면, 다음과 같은 이점이 있습니다.
- 프로젝트별 빌드스크립트에 중복되어 들어가는 내용을 줄일 수 있다.
	- 똑같은 내용의 JAR 태스트가 각 Java 프로젝트마다 있을 필요는 없겠죠?
- 각 프로젝트가 전체 프로젝트의 규약/관습(convention)을 어기는 상황을 도구로 관리할 수 있다.
	- MANIFEST 파일 형식, Java 버전, 의존성 버전 등

아래 예시에서는 두 번째 특징에 조금 더 주목해보고 싶네요.

Java 21에서 처음 소개된 `Math.clamp` 메서드를 사용하는 상황을 보겠습니다.
코드를 아래와 같이 작성하고, 다시 `jar` 태스크를 실행하면 어떻게 될까요?

```java
package com.blue;  
  
public class Main  
{  
    public static void main(String[] args)  
    {  
        System.out.println("Hello World, this is an example Gradle project!");  
        int result = Math.clamp(15, 0, 10);  
    }  
}
```

결과는 다음과 같습니다. `compileJava` 단계에서 실패했는데요!

```
blue@Bluebook:~/Gradle/my-project$./gradlew clean jar 
> Task :compileJava FAILED
...(생략)/src/main/java/com/blue/Main.java:8: error: cannot find symbol
        int result = Math.clamp(15, 0, 10);
```

우리가 정의한 플러그인에서 Java 툴체인을 17버전으로 사용하기로 했기 때문에, 소스코드를 컴파일하는 작업인 `compileJava` 역시 17 버전을 기준으로 수행되었지만, 소스코드에 Java 17에는 없는 내용이 있어 컴파일에 실패하게 된 것입니다.


## 나가며

`buildSrc`를 활용해서 여러 프로젝트에서 활용 가능한 공통 로직을 작성하는 방법을 알아보았습니다.

특히 예시에서 본 것처럼, 컨벤션 플러그인을 이용하면 여러 프로젝트의 빌드스크립트에 중복으로 작성하는 내용을 압축할 수 있고, 다수 프로젝트에 공통적인 규약을 보다 쉽게 적용할 수 있으니 활용해 보기를 권해드립니다. 

읽어주셔서 감사합니다.