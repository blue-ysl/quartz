---
aliases:
title: 자주 쓰는 정규식 모음
---
`마지막 수정 일시: 2026년 4월 25일 토요일, 14시 (KST)`

정규식은 레퍼런스도 많고 LLM의 시대에 더 이상 공부할 필요는 없지만, 매번 검색하는 것보다 자주 쓰는 것들은 손에 익히는 것이 효율적이지 않을까 생각합니다.

이 문서는 지속적으로 업데이트 됩니다.

## 특정 문자열을 포함하는 줄 전체를 선택

```
^.*KEYWORD.*$
```

 `^`와 `$`를 포함하는 거은 줄의 시작과 끝을 명시적으로 나타내기 위해서입니다. 만약 Visual Studio Code의 탐색기나 `grep`처럼 줄 단위로 정규식을 평가하는 도구라면, `.*KEYWORD.*`만 검색해도 결과는 같게 나올 것입니다.

`grep` 사용 예시는 아래와 같습니다. 우선, 샘플 로그입니다.

```text
[2026-04-25 08:01:12] INFO  App started successfully
[2026-04-25 08:01:13] DEBUG Initializing myArray[0] = 0
[2026-04-25 08:01:13] DEBUG Initializing myArray[1] = 0
[2026-04-25 08:01:14] INFO  Loading config from config.json
[2026-04-25 08:01:15] DEBUG myArray[2] = "hello"
[2026-04-25 08:01:16] WARN  myArray[3] is undefined, using default
[2026-04-25 08:01:17] INFO  User 'admin' logged in from 192.168.1.10
[2026-04-25 08:01:18] DEBUG Reading myArray [4] = 42
[2026-04-25 08:01:19] ERROR Null reference at myArray[5]
[2026-04-25 08:01:20] INFO  Retrying...
[2026-04-25 08:01:21] DEBUG myArray[5] = "fallback"
[2026-04-25 08:01:22] INFO  Processing batch of 10 items
[2026-04-25 08:01:23] DEBUG Loop index i=0: myArray[i] = "apple"
[2026-04-25 08:01:24] DEBUG Loop index i=1: myArray[i] = "banana"
[2026-04-25 08:01:25] DEBUG Loop index i=2: myArray[i] = "cherry"
[2026-04-25 08:01:26] WARN  Disk usage at 85%
[2026-04-25 08:01:27] DEBUG anotherArray[0] = 99
[2026-04-25 08:01:28] DEBUG myArrayExtra[0] = "should not match word boundary"
[2026-04-25 08:01:29] DEBUG myArray[i + 1] = "computed index"
[2026-04-25 08:01:30] ERROR Connection timeout after 30s
[2026-04-25 08:01:31] INFO  Reconnecting to db host: db.internal:5432
[2026-04-25 08:01:32] DEBUG myArray[
  index
] = "multiline index"
[2026-04-25 08:01:33] INFO  Reconnection successful
[2026-04-25 08:01:34] DEBUG myArray[row][col] = "matrix value"
[2026-04-25 08:01:35] INFO  Cache cleared
[2026-04-25 08:01:36] DEBUG myArray[] declared but not assigned
[2026-04-25 08:01:37] WARN  myArray[999] out of bounds
[2026-04-25 08:01:38] INFO  Shutdown initiated
[2026-04-25 08:01:39] INFO  App stopped
```

다음과 같이 사용할 수 있습니다.
- `grep "^.*DEBUG.*$" sample.log`

![[tricks_regex_1.png]]
