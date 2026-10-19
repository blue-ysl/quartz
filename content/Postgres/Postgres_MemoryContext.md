---
title: Postgres backend process의 memory context
---
원글: [Memory context: private memory management in PostgreSQL](https://www.cybertec-postgresql.com/en/memory-context-for-postgresql-memory-management/)

최종 수정일자: 2026-10-19 (월)

---
### PostgreSQL 메모리 컨텍스트

PostgreSQL 백엔드 프로세스(이하 PG 세션)은 작업 수행에 필요한 메모리를 memory context로부터 할당받는다. Memory context란, 간단히 말하자면 메모리를 미리 할당한 창고이다. 만약 PG 세션이 작업을 수행하기 위해 메모리를 일일이 할당하고 그 중 몇 개를 해제하지 않으면, 메모리 누수가 발생한다.

만약 메모리를 그때마다 일일이 할당하지 않고, 이미 할당된 메모리에서 필요한 부분만 잘라서 주고 모든 작업이 끝날 때 한 번만 해제하는 방식으로 구성한다면, 위와 같은 메모리 누수 발생 가능성은 줄 것이다. 자세한 내용은 구현을 봐야겠지만, PostgreSQL을 활용하는 사람 입장에서는 이 정도로 충분해 보인다.

컨텍스트끼리는 계층(트리형)을 구성하는데, 이 역시 간편한 메모리 관리를 위한 것으로 보인다. 상위 컨텍스트를 정리할 때 재귀적으로 하위 컨텍스트까지 정리하도록 구성하면, 메모리 해제를 신경써야 하는 포인트가 상대적으로 일일이 할당할 때에 비하여 줄어들기 때문이다. 

### 메모리 컨텍스트의 종류

cache memory context, portal context, executor state, cached plan source 등 여러 가지가 있다. 각각이 무엇을 의미하는지에 대해서는 필요할 때 찾으면 된다. PG 세션을 하나 만들고 여러 가지의 SQL을 실행하며 `pg_backend_memory_contexts` 뷰를 조회하면 파악에 도움이 될 것이다.

`pg_backend_memory_contexts` 뷰는 PostgreSQL 14 버전부터 지원된다.
- 문서: [pg_backend_memory_contexts](https://www.postgresql.org/docs/17/view-pg-backend-memory-contexts.html)

```sql
select distinct
	parent,
	level - 1
from pg_backend_memory_contexts
order by 2, 1;
```

재밌는 것 한 가지 나누자면, 위 쿼리에서 `order by` 절을 넣고 빼며 결과를 비교해 보길 바란다. 정렬을 위한 메모리 공간이 필요하다는 걸 알고 있다면 흥미롭게 느껴지는 결과일 거라 생각한다.

### 메모리 컨텍스트의 크기

memory context는 계층형 구조를 이루지만 부모가 자식의 메모리 공간을 포함하지는 않는다. 각 메모리 컨텍스트의 할당된 공간은 서로 독립적인 것이다. 따라서, PG 세션의 전체 memory context 공간 크기가 궁금하다면 다음 쿼리를 실행하면 된다.

```sql
SELECT
    pg_size_pretty(SUM(total_bytes)) AS total_memory,
    pg_size_pretty(SUM(used_bytes)) AS used_memory,
    pg_size_pretty(SUM(free_bytes)) AS free_memory
FROM pg_backend_memory_contexts;
```

현재 자기 세션에 대해서 알고자 할 때 위 쿼리를 실행하는 것이다. 특정 세션의 memory context 상태에 대해서 알고 싶다면 `pg_log_backend_memory_contexts`를 활용해야 하며, 필요에 따라 Postgres 서버의 log level을 변경해야 할 수도 있다. 

### `CacheMemoryContext`

개인적으로 응용 프로그램을 개발하는 사람들에게 의미가 있다고 생각하는 memory context는 바로 이 cache memory context (이하 '캐시 컨텍스트')이다. 이 공간을 만만하게 여기다가는 시스템 메모리 부족 현상을 겪을 수 있기 떄문이다.

캐시 컨텍스트는 데이터베이스 메타데이터와 실행 계획을 캐시하는 공간이다. 다음 쿼리 결과를 보면 쉽게 다가올 것이다. 결과는 PG 세션에서 실행한 작업에 따라 더 나올 수도, 덜 나올 수도 있다.

```sql
select distinct
	name
from pg_backend_memory_contexts
where parent = 'CacheMemoryContext';

-- result
/*
name                |
--------------------+
CachedExpression    |
CachedPlan          |
CachedPlanSource    |
SPI Plan            |
index info          |
partition constraint|
partition descriptor|
partition key       |
relation rules      |
*/
```

캐시 컨텍스트에 당연히 실제 user data 캐시가 담기지는 않는다. 그런 건 shared buffer 영역에 있다. 캐시 컨텍스트가 담는 것은 user data 의존적이지 않은 정보, 즉, 테이블의 위치나 이미 실행한 적 있는 SQL의 실행계획 등이다.

`pg_backend_memory_contexts`의 `ident` 컬럼을 같이 조회하면 세션에서 실행한 SQL이 참조하는 파티션이나 인덱스 정보를 쉽게 확인할 수 있다.

```sql
select distinct
	name, ident
from pg_backend_memory_contexts
where parent = 'CacheMemoryContext';
```

캐시 컨텍스트의 특성을 아는 것이 메모리 관리 측면에서 왜 중요할까?
- 첫째, 캐시 컨텍스트의 크기는 PG 세션이 접근하는 DB 오브젝트 개수, 실행하는 SQL 개수에 비례한다.
- 둘째, **이 캐시는 줄어들지 않기 때문이다.** PG 세션 종료 전까지는, 메모리는 할당된 상태로 유지된다.

이런 이유로, 크게는 다음과 같은 두 가지 경우에 문제가 될 수 있다.
- **PG 세션을 한 번 연결한 후에는 해제하지 않고 계속해서 사용하는 경우**: 캐시 컨텍스트가 할당한 메모리를 종료 전까지는 해제하지 않는다. 이것은 엄밀한 의미에서 '메모리 누수'는 아니지만, 실제로 곧바로 활용되지도 않는 메모리가 점유되는 상태이기 때문에 시스템 레벨에서의 메모리 부족을 유발할 수 있다.
- **여러 세션 각각이 단시간에 많은 DB 오브젝트에 접근하거나, 다양한 SQL를 파싱/실행하는 경우**: 접근하는 개체 개수, 사용하는 SQL 개수에 비례하여 증가하기 때문에, 단시간에 PG 세션들의 메모리 사용량이 증가할 수 있다. 순간적으로 메모리 사용량이 피크를 치면서 시스템 자원이 부족해질 수 있다.

본인의 시스템에서 PG 세션들이 유의미하게 메모리를 많이 사용하고 있는 경우라면, 실제 작업과 유사한 패턴으로 동작하는 PG 세션을 만들어서 `pg_backend_memory_contexts` 뷰를 조회하거나, 문제가 되는 세션에 대해 `pg_log_backend_memory_contexts` 함수를 실행해서 데이터를 확인해 보기를 권한다.

### 곁다리: `workmem`과의 차이

`workmem`은 개별 SQL의 실행게획 한 단계를 수행할 때 사용하는 메모리 공간 크기 설정이다. 거칠게 말하면 메모리 컨텍스트의 크기와는 무관하다. 그냥 그렇게 외우는 편이 낫다. 물론 따지자면, 한 세션의 메모리 컨텍스트의 하위 컨텍스트가 SQL을 실행할 대 할당하는 메모리 크기에 관한 파라미터이니 '무관하다'고 하기는 애매하다. 다만, '`workmem`보다 PG 세션의 크기가 클 수 없다'고 생각하는 것은 분명한 오해이다.

(끝)