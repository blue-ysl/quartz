---
aliases:
title: 잘못된 ConcurrentHashMap 사용과 putIfAbsent 활용
---
`마지막 수정 일시: 2026년 6월 5일 금요일, 01시 (KST)`

## `ConcurrentHashMap`

하나의 `HashMap`에 여러 스레드에서 접근하는 경우, 동시성을 생각하여 `ConcurrentHashMap`을 쓰는 경우가 있습니다. 그동안은 'Java에서 어련히 잘 해주겠지'라고 생각하다가, 동작 방식을 명확하게 알아야 할 필요가 생겨서 공식 문서를 찾아 보았습니다.
- [ConcurrentHashMap (Java Platform SE 8 ) - Oracle Help Center](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentHashMap.html "ConcurrentHashMap (Java Platform SE 8 ) - Oracle Help Center")

첫 줄의 내용은 이렇습니다.

```
A hash table supporting full concurrency of retrievals and high expected concurrency for updates.
```

영어가 모국어가 아닌지라 해석에 시간이 걸렸습니다. 오해했을 수도 있지만, 우선 이렇게 결론 내렸습니다.

- `full concurrency`: 완전한 동시성이라고 번역할 수 있을 것 같습니다. 동시성이 '완전하면' 무엇일까요? 저는 '여러 곳에서 동시에 접근해도 전혀 제약이 없는 상태'로 이해했습니다. 즉, 탐색 작업 대해서는 block이 전혀 없는 것이지요.
- `high expected concurrency`: 갱신에 대해서는 높은 수준의 동시성을 제공한다는 정도로 이해했습니다. 다른 무언가에 비해서 더 높은 수준의 동시성을 제공한다는 뜻일 것으로 보입니다.   

이어지는 내용에서는 ConcurrentHashMap을 잘못 사용하는 사례와 이에 대한 개선 방안을 서술합니다.

## 잘못된 사용 패턴: 따로 노는 읽기와 쓰기기

예시 코드입니다.
- `MyThread` 클래스
	- `num`과 `key`를 멤버변수로 갖습니다.
	- 3초마다 `HashMapTest`의 `myFunc`를 실행합니다. 
- `HashMapTest`:
	- 시작점에서 `MyThread` 스레드 2개 인스턴스를 생성하여 실행합니다.
	- 스레드들의 키 값을 `myKey`로 동일하게 지정합니다. 
	- `resource`라는 이름의 `ConcurrentHashMap`을 상수로 가지고 있습니다.
	- 스레드들은 `myFunc` 함수 호출을 통해 이 맵에 동시에 접근하게 됩니다다.

```java
// HashMapTest.java
package test;  
  
import util.MyThread;  
  
import java.util.concurrent.ConcurrentHashMap;  
  
public class HashMapTest  
{  
    public static final ConcurrentHashMap<String, String> resource = new ConcurrentHashMap<>();  
  
    public static void main(String[] args)  
    {  
        String key = "myKey";  
        for (int i = 0; i < 2; i++)  
        {  
            Thread th = new Thread(new MyThread(i + 1, key));  
            th.start();  
        }  
    }  
  
    public static void myFunc(String threadKey, int threadNum) throws Exception  
    {  
        if (resource.containsKey(threadKey))  
        {  
            return;  
        }  
        
        System.out.printf("PointA: %d (%d)\n", threadNum, System.nanoTime());  
        if (threadNum == 1)  
        {  
            Thread.sleep(1000); // 특정 스레드의 일 처리가 지연되는 구간  
        }  
        
        System.out.printf("PointB: %d (%d)\n", threadNum, System.nanoTime());  
        resource.put(threadKey, "myValue");  
        /*  
            do something here...
        */       
        
        resource.remove(threadKey);  
        System.out.printf("PointC: %d (%d)\n", threadNum, System.nanoTime());  
    }
}
```

```java
// MyThread.java
package util;  
  
import test.HashMapTest;  
  
public class MyThread implements Runnable  
{  
    private final int num;  
    private final String key;  
  
    public MyThread(int num, String key)  
    {  
        this.num = num;  
        this.key = key;  
    }  
  
    @Override  
    public void run()  
    {  
        while (true)  
        {  
            try  
            {  
                HashMapTest.myFunc(key, num);
                Thread.sleep(3000);  
            } catch (Exception e)  
            {  
                throw new RuntimeException(e);  
            }  
        }  
    }  
}
```

`myFunc`의 의도는 다음과 같습니다.
1. `resource` 맵에 현재 스레드가 전달하는 key 값이 이미 들어있는지 확인한다. 만약 그렇다면 이 key 값과 관련된 일이 이미 처리 중인 상태이므로, 현재 스레드를 반환한다.
2. 만약 key 값이 들어있지 않다면, `resource` 맵에 key를 삽입하여, 해당 key 값에 대해서 작업이 진행 중임을 표시한다.
3. 필요한 작업을 진행하고, 현재 스레드가 가지고 있던 key 값을 remove 하여 해당 key 값에 대해서 더 이상 작업이 진행 중이 아님을 표시한다.

이 코드에는 큰 문제가 있는데, 바로 1번과 2번 사이에 틈이 있기 때문입니다. 이 틈을 위 코드에서는 '1번' 스레드의 1초 간 sleep으로 표현하였습니다. 물론 실제 이 정도로 지연되는 일은 웬만해서는 없겠지만, 의도 전달 차원에서 포함했습니다.

예를 들어 1번 스레드가 `containsKey` 호출을 지난 후 `put`으로 작업 상태를 표시하기 전, 그 사이에 2번 스레드가 `containsKey` 호출을 `false`로 통과하여 다음 단계로 진행하게 되면, 두 스레드가 같은 `key` 값에 대해서 작업을 동시에 진행하는 상태가 됩니다. 

실제 실행한 결과는 다음과 같습니다.

```
> Task :test.HashMapTest.main()
PointA: 1 (1376395750474700) // 1번 스레드 myFunc 작업 시작 전
PointA: 2 (1376395750484600) // 2번 스레드가 치고 들어와서
PointB: 2 (1376395757760200) // 
PointC: 2 (1376395759039900) // 작업을 끝내고 나감
PointB: 1 (1376396770906700) // 1번 스레드는 그 후에야 작업을 시작
PointC: 1 (1376396771108600)
PointA: 2 (1376398763549000)
PointB: 2 (1376398765120100)
PointC: 2 (1376398765216600)
PointA: 1 (1376399775084900)
...
```

위 코드는 아주 간단한 예시이기 때문에 잘 드러나지 않았지만, 만약 수행하는 작업이 트랜잭션의 성격을 가지고 있어야 한다면, 두 개 이상의 스레드가 동시에 작업 구간에 진입하는 것이 문제가 되었을 것입니다.

### 어떻게 고칠까?

문제를 고치기 위해서는 1번과 2번 단계를 `putIfAbsent` 호출로 합치면 됩니다. 다음과 같습니다.

```java
public static void myFunc(String threadKey, int threadNum) throws Exception  
{  
    if (resource.putIfAbsent(threadKey, "myValue") != null)  
    {  
        return;  
    }  
  
    System.out.printf("PointA: %d (%d)\n", threadNum, System.nanoTime());  
    if (threadNum == 0)  
    {  
        Thread.sleep(1000);  
    }  
    System.out.printf("PointB: %d (%d)\n", threadNum, System.nanoTime());  
  
    resource.remove(threadKey);  
    System.out.printf("PointC: %d (%d)\n", threadNum, System.nanoTime());  
}
```

`myFunc`를 위와 같이 교체하고, 코드를 실행한 결과는 다음과 같습니다. 1번과 2번 스레드가 서로 교차해 가며 작업을 실행하고 있고, 한 스레드의 작업이 끝나기 전(PointC)까지 다른 스레드가 끼어드는 일은 없는 것으로 보입니다.
```
> Task :test.HashMapTest.main()
PointA: 1 (1377052155497800)
PointB: 1 (1377052160804200)
PointC: 1 (1377052160855200)
PointA: 2 (1377055168714700)
PointB: 2 (1377055169260000)
PointC: 2 (1377055169514700)
PointA: 1 (1377058172981600)
PointB: 1 (1377058174254700)
PointC: 1 (1377058174529800)
PointA: 1 (1377061182425400)
PointB: 1 (1377061182970300)
PointC: 1 (1377061183866500)
PointA: 1 (1377064191613800)
PointB: 1 (1377064193380100)
PointC: 1 (1377064193868100)
PointA: 1 (1377067203612900)
PointB: 1 (1377067204021800)
PointC: 1 (1377067204138100)
PointA: 2 (1377070205518500)
PointB: 2 (1377070206708400)
PointC: 2 (1377070206979100)
...
```

[해당 함수에 대한 Java 문서의 해설](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentHashMap.html#putIfAbsent-K-V-)에 따르면, `putIfAbsent`는 다음의 코드를 원자적으로 실행한 것과 같습니다.

```java
 if (!map.containsKey(key))
   return map.put(key, value);
 else
   return map.get(key);
```

스레드가 가진 key가 resource에 있는지 확인하고 없으면 삽입하고, 있으면 읽어들이는 과정을 하나로 묶어서 수행하는 것입니다.

이러한 점 덕분에, `putIfAbsent` 함수는 `myFunc` 함수의 본래 의도에 맞게, 즉, `resource` 내 key 존재 여부를 통해 작업 상태를 표시하고 동시 접근을 제한하기 위한 목적으로 사용될 수 있습니다.

## 마치며

짧은 예시를 통해 `ConcurrentHashMap`의 (크게) 잘못된 사용례를 알아 보았습니다.

[ConcurrentHashMap (Java Platform SE 8 ) - Oracle Help Center](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentHashMap.html "ConcurrentHashMap (Java Platform SE 8 ) - Oracle Help Center")  문서를 보면 다른 메서드들에 대한 설명도 자세하게 나와 있어 도움이 될 것으로 생각합니다.