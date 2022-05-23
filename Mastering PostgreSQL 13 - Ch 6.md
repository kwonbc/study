# 6. 좋은 성능을 위한 쿼리 최적화

5장에서, 로그 파일 및 시스템 통계에서 시스템 통계를 읽는 방법과 PostgreSQL이 제공하는 것을 활용하는 방법을 배웠습니다. 이제 이 지식으로 무장했으므로 이 장은 우수한 쿼리 성능에 관한 것입니다. 모두가 좋은 쿼리 성능을 찾고 있습니다. 따라서 이 주제를 심층적으로 다루는 것이 중요합니다.

이 장에서는 다음 주제에 대해 배웁니다.

* 옵티마이저가 하는 일 학습
* 실행 계획 이해
* 조인 이해 및 수정
* 옵티마이저 설정 활성화 및 비활성화
* 데이터 분할
* 좋은 쿼리 성능을 위한 매개변수 조정
* 병렬 쿼리 사용
* JIT(Just-in-Time) 컴파일 도입

이 장의 끝에서 우리는 더 좋고 더 빠른 쿼리를 작성할 수 있을 것입니다. 쿼리가 여전히 좋지 않다면 왜 그런지 이해할 수 있어야 합니다. 또한 데이터 분할에 대해 배운 새로운 기술을 사용할 수 있습니다.

## 옵티마이저가 하는 일 배우기

쿼리 성능에 대해 생각하기 전에 쿼리 최적화 프로그램이 하는 일에 익숙해지는 것이 좋습니다. 내부에서 무슨 일이 일어나고 있는지 더 깊이 이해하면 데이터베이스가 실제로 무엇을 하는지 알 수 있기 때문에 많은 의미가 있습니다.

### 실제 예 – 쿼리 최적화 프로그램이 샘플 쿼리를 처리하는 방법

옵티마이저가 어떻게 작동하는지 보여주기 위해 예제를 컴파일했습니다. PostgreSQL 교육을 위해 수년 동안 사용한 것입니다. 다음과 같은 세 개의 테이블이 있다고 가정해 보겠습니다.

```
CREATE TABLE a (aid int, ...); -- 100 million rows
CREATE TABLE b (bid int, ...); -- 200 million rows
CREATE TABLE C (cid int, ...); -- 300 million rows
```

이러한 테이블에 수백만 또는 수억 개의 행이 포함되어 있다고 가정해 보겠습니다. 그 외에도 다음과 같은 색인이 있습니다.

```
CREATE INDEX idx_a ON a (aid); 
CREATE INDEX idx_b ON b (bid); 
CREATE INDEX idx_c ON c (cid); 
CREATE VIEW v AS SELECT * 
	FROM a, b
	WHERE aid = bid;
```

마지막으로 처음 두 테이블을 결합하는 뷰가 있습니다.

최종 사용자가 다음을 실행하기를 원한다고 가정해 봅시다.
옵티마이저는 이 쿼리로 무엇을 합니까? 기획자는 어떤 선택을 할 수 있습니까?

```
SELECT *
FROM v, c
WHERE v.aid = c.cid
	AND cid = 4;
```

실제 최적화 프로세스를 살펴보기 전에 플래너가 가지고 있는 몇 가지 옵션에 중점을 둘 것입니다.

#### 조인 옵션 평가

계획자는 여기에 몇 가지 옵션이 있으므로 이 기회를 통해 직접적인 접근 방식을 사용할 경우 무엇이 잘못될 수 있는지 이해해 보겠습니다.

플래너가 계속 진행하여 보기의 출력을 계산한다고 가정합니다. 1억 행과 2억 행을 결합하는 가장 좋은 방법은 무엇입니까?

이 섹션에서는 PostgreSQL이 무엇을 할 수 있는지 보여주기 위해 몇 가지(전부는 아님) 조인 옵션에 대해 설명합니다.

##### 중첩 루프

두 테이블을 결합하는 한 가지 방법은 중첩 루프를 사용하는 것입니다. 여기의 원리는 간단하다. 다음은 몇 가지 의사 코드입니다.

```
for x in table1:
	for y in table2:
		if x.field == y.field
			issue row
		else
			keep doing
```

중첩 루프는 측면 중 하나가 매우 작고 제한된 데이터 집합만 포함하는 경우에 종종 사용됩니다. 이 예에서 중첩된 루프는 코드를 통해 1억 x 2억 번의 반복을 초래할 것이다. 런타임은 단순히 폭발적으로 증가할 것이기 때문에 이것은 분명히 옵션이 아니다.

중첩 루프는 일반적으로 O(n2)이므로 조인 한쪽이 매우 작을 경우에만 효율적입니다. 이 예에서는 그렇지 않으므로 중첩 루프는 뷰를 계산하기 위해 제외될 수 있습니다.

##### 해시 조인

두 번째 옵션은 해시 조인입니다. 다음의 전략이 우리의 작은 문제를 해결하기 위해 적용될 수 있다. 다음 목록은 해시 조인 작동 방식을 보여줍니다.

```
Hash join
	Sequential scan table 1
	Sequential scan table 2
```

양쪽을 해시할 수 있고 해시 키를 비교할 수 있으므로 결과를 얻을 수 있습니다.
조인의 여기서 문제는 모든 값을 해시하여 어딘가에 저장해야 한다는 것입니다.

##### 결합 병합

마지막으로 병합 조인이 있습니다. 여기서 목적은 정렬된 목록을 사용하여 결과를 결합하는 것입니다. 조인 양쪽이 정렬된 경우 시스템은 맨 위에서 행을 가져와 일치하는지 확인하고 해당 행을 반환할 수 있습니다. 여기서 주요 요구 사항은 목록이 정렬되어야 한다는 것입니다. 다음은 샘플 계획입니다.

```
Merge join
	Sort table 1
		Sequential scan table 1
	Sort table 2
		Sequential scan table 2
```

이 두 테이블(표 1과 표 2)을 결합하려면 데이터가 정렬된 순서로 제공되어야 합니다. 많은 경우, PostgreSQL이 데이터를 정렬합니다. 그러나 정렬된 데이터로 조인할 수 있는 다른 옵션이 있습니다. 한 가지 방법은 다음 예제와 같이 인덱스를 참조하는 것입니다.

```
Merge join
	Index scan table 1
	Index scan table 2
```

조인 한 쪽 또는 두 쪽 모두 계획의 하위 수준에서 가져온 정렬된 데이터를 사용할 수 있습니다. 테이블에 직접 액세스한 경우 인덱스가 이를 위한 확실한 선택이지만 반환된 결과 집합이 전체 테이블보다 훨씬 작은 경우에만 가능합니다. 그렇지 않으면 전체 인덱스를 읽은 다음 전체 표를 읽어야 하므로 오버헤드가 거의 두 배로 증가합니다. 결과 세트가 테이블에서 많은 부분을 차지하는 경우, 특히 기본 키 순서로 액세스하는 경우 순차적 검색이 더 효율적입니다.

병합 조인의 장점은 많은 데이터를 처리할 수 있다는 것이다. 단점은 데이터가 어느 시점에서 분류되거나 인덱스에서 추출되어야 한다는 것입니다.

정렬은 O(n * log(n))입니다. 따라서 조인을 수행하기 위해 3억 개의 행을 정렬하는 것도 매력적이지 않습니다.

> PostgreSQL 10.0이 도입된 이후 여기에 설명된 모든 조인 옵션은 병렬 버전에서도 사용할 수 있습니다. 따라서 옵티마이저는 이러한 표준 조인 옵션을 고려할 뿐만 아니라 병렬 쿼리를 수행하는 것이 합리적인지 여부도 평가합니다.

#### 변환 적용

분명히, 명백한 일을 하는 것(뷰에 먼저 합류하는 것)은 전혀 의미가 없습니다. 중첩 루프는 루프를 통해 실행 시간을 보냅니다. 해시 조인은 수백만 행을 해시해야 하고 중첩 루프는 3억 행을 정렬해야 합니다. 세 가지 옵션 모두 여기에는 분명히 적합하지 않습니다. 탈출구는 논리적 변환을 적용하여 쿼리를 빠르게 만드는 것입니다. 이 섹션에서는 플래너가 쿼리 속도를 높이기 위해 수행하는 작업을 배웁니다. 몇 가지 단계를 수행해야 합니다.

1. 뷰의 선긋기: 최적기가 수행하는 첫 번째 변환은 뷰의 인라인입니다. 다음과 같은 일이 발생합니다:

```
SELECT *
FROM
(
	SELECT *
	FROM a, b
	WHERE aid = bid
) AS v, с
WHERE v.aid = c.cid
	AND cid = 4;
```

보기가 인라인되고 하위 선택으로 변환됩니다. 이것은 우리에게 무엇을 사줍니까? 사실, 아무것도. 이것이 하는 일은 추가 최적화를 위한 문을 여는 것뿐이며, 이는 실제로 이 쿼리의 판도를 바꿀 것입니다.

2. 하위 선택 병합: 다음으로 해야 할 일은 하위 선택을 평면화하는 것입니다. 이는 하위 선택을 기본 쿼리에 통합하는 것을 의미합니다. 하위 선택을 제거하면 쿼리를 최적화하는 데 사용할 수 있는 몇 가지 옵션이 더 나타납니다.

다음은 하위 선택을 병합한 후의 쿼리 모양입니다:

```
SELECT * FROM a, b, c WHERE a.aid = c.cid AND aid = bid AND cid = 4;
```

이제 더 많은 최적화를 적용할 수 있는 옵션을 제공하는 일반 조인입니다. 이 인라이닝 단계가 없으면 불가능합니다. 현재 사용할 수 있는 최적화를 살펴보겠습니다.

> 이 SQL을 자체적으로 다시 작성할 수도 있었지만 어쨌든 계획자가 이러한 변환을 처리할 것입니다. 옵티마이저는 이제 추가 최적화를 수행할 수 있습니다.

#### 등식 제약 조건 적용

다음 프로세스는 동등 제약 조건을 생성합니다. 아이디어는 추가 제약 조건, 조인 옵션 및 필터를 감지하는 것입니다. 심호흡을 하고 다음 쿼리를 살펴보겠습니다. 만약 aid = cid이고, aid = bid라면, 우리는 bid = cid라는 것을 압니다. cid = 4이고 다른 모든 항목이 동일하면 Aid와 bid도 4여야 한다는 것을 알고 있으므로 다음 쿼리가 생성됩니다:

```
SELECT *
FROM a, b, c
WHERE a.aid = c.cid
	AND aid = bid
	AND cid = 4
	AND bid = cid
	AND aid = 4
	AND bid = 4
```

이 최적화의 중요성은 아무리 강조해도 지나치지 않습니다. 여기서 플래너는 원래 쿼리에서 명확하게 볼 수 없었던 두 개의 추가 인덱스에 대한 문을 열었습니다.

세 개의 열 모두에서 인덱스를 사용할 수 있으므로 쿼리 비용이 훨씬 저렴해졌습니다. 옵티마이저에는 인덱스에서 몇 개의 행만 검색하고 합당한 조인 옵션을 사용하는 옵션이 있습니다.

#### 철저한 검색

이제 이러한 형식 변환이 완료되었으므로 PostgreSQL은 철저한 검색을 수행합니다. 가능한 모든 계획을 시도하고 쿼리에 대한 가장 저렴한 솔루션을 제공합니다. PostgreSQL은 가능한 인덱스를 알고 있으며 비용 모델을 사용하여 가능한 최선의 방법으로 작업을 수행하는 방법을 결정합니다.

철저한 검색 중에 PostgreSQL은 최상의 조인 순서도 결정하려고 시도합니다. 원래 쿼리에서 조인 순서는 A → B 및 A → C로 고정되었습니다. 그러나 이러한 동등 제약 조건을 사용하여 B → C를 조인하고 나중에 A를 조인할 수 있습니다. 이 모든 옵션은 기획자에게 열려 있습니다.

#### 실행 계획 확인

이제 모든 최적화 옵션에 대해 논의했으므로 PostgreSQL에서 어떤 종류의 실행 계획을 생성하는지 확인할 차례입니다.

먼저 완전히 분석되었지만 빈 테이블이 있는 쿼리를 사용해 보겠습니다:

```
test #
explain SELECT * FROM V, C WHERE v.aid = c.cid AND cid = 4; QUERY PLAN
Nested Loop (cost=12.77..74.50 rows=2197 width=12) -> Nested -> Bitmap Loop Heap (cost =Scan 8.51..32.05 on a rows=169 width=8)
Recheck Cond: (aid = 4()cost=4.26..14.95 rows=13 width=4) ->Bitmap Index Scan on idx_a (cost=0.00..4.25 rows=13 width=0)
Index Cond: (aid = 4)
Materialize
Bitmap Heap Scan on b width=4)
->
(cost=4.26..15.02 rows-13 width=4) (cost=4.26..14.95 rows=13 Recheck Cond: (bid=4)
-> Bitmap Index Scan on idx_b (cost=0.00..4.25
rows=13 width=0)
Index Cond: (bid = 4)
Materialize (cost=4.26..15.02 rows=13 width=4)
Bitmap Heap Scan on c (cost=4.26..14.95 rows=13 width=4)
Recheck Cond: (cid = 4)
Bitmap Index Scan on idx_c (cost=0.00..4.25 rows=13
width=0)
Index Cond: (cid = 4)
(16 rows)
```

보이는 것은 빈 테이블을 사용하여 생성된 계획입니다. 그러나 데이터를 추가하면 어떻게 되는지 확인해 보겠습니다:

```
test=# INSERT INTO a SELECT * FROM generate_series (1, 1000000);
INSERT 0 1000000
test=# INSERT INTO E SELECT * FROM generate_series (1, 1000000);
INSERT 1000000
test=# INSERT INTO C SELECT * FROM generate_series (1, 1000000);
INSERT 0 1000000
test=# ANALYZE;
ANALYZE
```

다음 코드와 같이 계획이 변경되었습니다. 그러나 중요한 것은 두 계획 모두에서 쿼리의 모든 열에 필터가 자동으로 적용되는 것을 볼 수 있다는 것입니다. PostgreSQL의 평등 제약 조건이 제대로 작동했습니다.

```
test=# explain SELECT * FROM V, C WHERE v.aid = c.cid AND cid = 4;

Nested Loop  (cost=1.71..8.38 rows=1 width=47)
  ->  Nested Loop  (cost=1.14..5.58 rows=1 width=29)
        ->  Index Scan using idx_a on a  (cost=0.57..2.79 rows=1 width=12)
              Index Cond: (aid = 4)
        ->  Index Scan using idx_b on b  (cost=0.57..2.79 rows=1 width=17)
              Index Cond: (bid = 4)
  ->  Index Scan using idx_c on c  (cost=0.57..2.79 rows=1 width=18)
        Index Cond: (cid = 4)
```

이 장에 표시된 계획은 관찰할 내용과 반드시 100% 동일하지는 않습니다. 로드한 데이터의 양에 따라 약간의 차이가 있을 수 있습니다. 비용은 디스크에 있는 데이터의 물리적 정렬(디스크에 있는 순서)에 따라 달라질 수도 있습니다. 이 예제를 실행할 때 이 점을 염두에 두십시오.

보시다시피 PostgreSQL은 3개의 인덱스를 사용합니다. PostgreSQL이 데이터를 결합하기 위해 중첩 루프를 사용하기로 결정한 것도 흥미롭습니다. 인덱스 스캔에서 되돌아오는 데이터가 사실상 없기 때문에 이것은 완벽합니다. 따라서 루프를 사용하여 사물을 결합하는 것은 완벽하게 실현 가능하고 매우 효율적입니다.

#### 프로세스를 실패로 만들기

지금까지 PostgreSQL이 무엇을 할 수 있고 옵티마이저가 쿼리 속도를 높이는 데 도움이 되는지 살펴보았습니다. PostgreSQL은 꽤 똑똑하지만 똑똑한 사용자가 필요합니다. 최종 사용자가 어리석은 일을 하여 전체 최적화 프로세스를 무력화시키는 경우가 있습니다. 다음 명령을 사용하여 뷰를 삭제해 보겠습니다.

```
test=# DROP VIEW V;
DROP VIEW
```

이제 뷰가 다시 생성되었습니다. OFFSET 0가 뷰 끝에 추가되었습니다. 다음 예를 살펴보겠습니다.

```
test=# CREATE VIEW V AS SELECT * 
		FROM a, b
		WHERE aid = bid
		OFFSET 0;
CREATE VIEW
```

이 보기는 이전에 표시된 예와 논리적으로 동일하지만 옵티마이저는 상황을 다르게 처리해야 합니다. 0이 아닌 모든 오프셋은 결과를 변경하므로 보기를 계산해야 합니다. 전체 최적화 프로세스는 OFFSET과 같은 항목을 추가하여 손상됩니다.

> PostgreSQL 커뮤니티는 이 패턴을 최적화하지 않기로 결정했습니다. 보기에서 OFFSET 0을 사용한 경우 플래너는 이를 제거하지 않습니다. 사람들은 그렇게 해서는 안 됩니다. 특정 작업이 성능을 저하시킬 수 있는 방법과 개발자로서 기본 최적화 프로세스를 알고 있어야 하는 방법을 관찰하기 위해 이것을 예로 사용할 것입니다. 그러나 PostgreSQL의 작동 방식을 알고 있다면 이 트릭을 최적화에 사용할 수 있습니다.

뷰에 OFFSET 0이 포함된 경우 새 계획은 다음과 같습니다.

```
test # EXPLAIN SELECT * FROM v, c WHERE v.aid = c.cid AND cid = 4;

Nested Loop  (cost=768.13..7427921.25 rows=1 width=47)
  ->  Subquery Scan on v  (cost=767.56..7427918.45 rows=1 width=29)
        Filter: (v.aid = 4)
        ->  Merge Join  (cost=767.56..6177918.45 rows=100000000 width=29)
              Merge Cond: (a.aid = b.bid)
              ->  Index Scan using idx_a on a  (cost=0.57..2342154.07 rows=100000000 width=12)
              ->  Index Scan using idx_b on b  (cost=0.57..4780704.97 rows=200000000 width=17)
  ->  Index Scan using idx_c on c  (cost=0.57..2.79 rows=1 width=18)
        Index Cond: (cid = 4)
```

계획자가 예측한 비용을 살펴보십시오. 비용이 두 자리 숫자에서 엄청난 숫자로 치솟았습니다. 분명히 이 쿼리는 나쁜 성능을 제공할 것입니다.

성능을 저하시키는 다양한 방법이 있지만 최적화 프로세스를 염두에 두는 것이 좋습니다.

#### 상수 접기 (Constant folding)

그러나 PostgreSQL에는 백그라운드에서 발생하고 전반적인 우수한 성능에 기여하는 더 많은 최적화가 있습니다. 이러한 기능 중 하나를 상수 접기라고 합니다. 아이디어는 다음 예제와 같이 표현식을 상수로 바꾸는 것입니다.

```
test # explain SELECT * FROM a WHERE aid = 3 + 1;

Index Scan using idx_a on a  (cost=0.57..2.79 rows=1 width=12)
  Index Cond: (aid = 4)
```

보시다시피 PostgreSQL은 4를 찾으려고 시도합니다. 도움이 인덱싱되었으므로 PostgreSQL은 인덱스 스캔을 수행합니다. 우리 테이블에는 단 하나의 열만 있으므로 PostgreSQL은 필요한 모든 데이터를 인덱스에서 찾을 수 있다는 것을 알아냈습니다.

표현식이 왼쪽에 있으면 어떻게 됩니까?

```
test=# explain SELECT * FROM a WHERE aid - 1 = 3;

Gather  (cost=1000.00..1473892.94 rows=500000 width=12)
  Workers Planned: 1
  ->  Parallel Seq Scan on a  (cost=0.00..1422892.94 rows=294118 width=12)
        Filter: ((aid - 1) = 3)
```

이 경우 인덱스 조회 코드가 실패하고 PostgreSQL은 순차 스캔을 수행해야 합니다. 여기에는 병렬 계획과 단일 코어 계획이라는 두 가지 예가 포함되어 있습니다.

간단하게 하기 위해 지금부터 보여지는 모든 계획은 단일 핵심 계획입니다.

#### 함수 인라이닝 이해

이 섹션에서 이미 설명했듯이 쿼리 속도를 높이는 데 도움이 되는 많은 최적화가 있습니다. 그 중 하나를 함수 인라이닝이라고 합니다. PostgreSQL은 변경할 수 없는 SQL 함수를 인라인할 수 있습니다. 주요 아이디어는 속도를 높이기 위해 수행해야 하는 함수 호출의 수를 줄이는 것입니다.

다음은 옵티마이저가 인라인할 수 있는 함수의 예입니다:

```
test=# CREATE OR REPLACE FUNCTION ld(int)
RETURNS numeric AS
$$
SELECT log (2, $1);
$$
LANGUAGE 'sql' IMMUTABLE;
CREATE FUNCTION
```

이것은 IMMUTABLE로 표시된 일반 SQL 함수입니다. 이것은 옵티마이저를 위한 완벽한 최적화 사료입니다. 간단하게 하기 위해 내 함수가 하는 일은 로그를 계산하는 것뿐입니다.

```
test=# SELECT ld (1024)
ld
10.0000000000000000
(1 row)
```

보시다시피 기능은 예상대로 작동합니다.

작동 방식을 보여주기 위해 인덱스 생성 프로세스의 속도를 높이기 위해 더 적은 콘텐츠로 테이블을 다시 생성합니다.

```
test=# TRUNCATE a;
TRUNCATE TABLE
```

그런 다음 TRUNCATE 데이터를 다시 추가하고 인덱스를 적용할 수 있습니다.

```
test=# INSERT INTO a SELECT * FROM generate_series (1, 10000);
INSERT 0 10000
test=# CREATE INDEX idx_ld ON a (ld (aid));
CREATE INDEX
```

예상대로 함수에 생성된 인덱스는 다른 인덱스처럼 사용됩니다. 그러나 인덱싱 조건을 자세히 살펴보겠습니다:

```
test=# EXPLAIN SELECT * FROM a WHERE ld(aid) = 10;

Index Scan using idx_ld on a  (cost=0.29..2.50 rows=1 width=36)
  Index Cond: (log('2'::numeric, (aid)::numeric) = '10'::numeric)
  
test=# ANALYZE ;

Index Scan using idx_ld on a  (cost=0.29..2.50 rows=1 width=36)
  Index Cond: (log('2'::numeric, (aid)::numeric) = '10'::numeric)
  
--------------------------------------------------

test=# EXPLAIN SELECT * FROM a WHERE ld(aid) = 10;
QUERY PLAN
Bitmap Heap Scan on a (cost=4.67..52.77 rows=50 width=4)
Recheck Cond: (log('2':: numeric, (aid) :: numeric) = '10'::numeric)
-> Bitmap Index Scan on idx_id (cost=0.00..4.66 rows=50 width=0)
Index Cond: (log('2'::numeric, (aid) ::numeric) = '10':: numeric)
(4 rows)
test=# ANALYZE ;
ANALYZE
test=# EXPLAIN SELECT * FROM a WHERE id (aid) = 10;
QUERY PLAN
Index Scan using idx_ld on a (cost=0.29..8.30 rows=1 width=4)
Index Cond: (log('2':: numeric, (aid) :: numeric) = '10'::numeric)
(2 rows)
```

여기서 중요한 관찰은 인덱싱 조건이 실제로 1d 함수 대신 로그 함수를 찾는다는 것입니다. 옵티마이저는 함수 호출을 완전히 제거했습니다. 새로운 최적화 통계는 효율적인 계획을 생성하는 데 정말 중요할 수 있다는 점도 언급할 가치가 있습니다.

논리적으로 이것은 다음 쿼리에 대한 문을 엽니다.

```
test=# EXPLAIN SELECT * FROM a WHERE log (2, aid) = 10;

Index Scan using idx_ld on a  (cost=0.29..2.50 rows=1 width=36)
  Index Cond: (log('2'::numeric, (aid)::numeric) = '10'::numeric)
```

옵티마이저는 함수를 인라인할 수 있었고 값비싼 순차 작업보다 훨씬 우수한 인덱스 스캔을 제공했습니다.

#### 조인 프루닝 소개

PostgreSQL은 조인 프루닝(join pruning)이라는 최적화 기능도 제공합니다. 아이디어는 쿼리에 필요하지 않은 경우 조인을 제거하는 것입니다. 쿼리가 일부 미들웨어 또는 ORM에 의해 생성되는 경우 유용할 수 있습니다. 조인을 제거할 수 있으면 자연스럽게 속도가 크게 빨라지고 오버헤드가 줄어듭니다.

이제 문제는 조인 가지 치기가 어떻게 작동합니까? 다음은 예입니다.

```
CREATE TABLE x (id int, PRIMARY KEY (id));
CREATE TABLE y (id int, PRIMARY KEY (id));
```

먼저 두 개의 테이블이 생성됩니다. 조인 조건의 양쪽이 실제로 고유한지 확인하십시오. 이러한 제약은 1분 안에 중요할 것입니다.

이제 간단한 쿼리를 작성할 수 있습니다:

```
test # EXPLAIN SELECT *
FROM x LEFT JOIN y ON (x.id = y.id)
WHERE x.id = 3;

Nested Loop Left Join  (cost=0.31..4.76 rows=1 width=8)
  Join Filter: (x.id = y.id)
  ->  Index Only Scan using x_pkey on x  (cost=0.15..2.37 rows=1 width=4)
        Index Cond: (id = 3)
  ->  Index Only Scan using y_pkey on y  (cost=0.15..2.37 rows=1 width=4)
        Index Cond: (id = 3)
```

보시다시피 PostgreSQL은 해당 테이블을 직접 조인합니다. 지금까지는 놀라움이 없습니다. 그러나 다음 쿼리가 약간 수정되었습니다. 모든 열을 선택하는 대신 조인의 왼쪽에 있는 열만 선택합니다:

```
test # EXPLAIN SELECT x.*
FROM x LEFT JOIN Y ON (x.id = y.id)
WHERE x.id = 3;

Index Only Scan using x_pkey on x  (cost=0.15..2.37 rows=1 width=4)
  Index Cond: (id = 3)
```

PostgreSQL은 직접 내부 스캔을 수행하고 조인을 완전히 건너뜁니다. 이것이 실제로 가능하고 논리적으로 올바른 두 가지 이유가 있습니다.

* 조인의 오른쪽에서 선택한 열이 없습니다. 따라서 해당 열을 찾는 것은 우리에게 아무 것도 사지 않습니다.
* 우변은 고유합니다. 즉, 조인은 우변의 중복으로 인해 행 수를 늘릴 수 없습니다.

조인을 자동으로 정리할 수 있으면 쿼리가 훨씬 더 빨라질 수 있습니다. 여기서의 장점은 어떤 경우에도 애플리케이션에 필요하지 않을 수 있는 열을 제거하기만 하면 속도를 높일 수 있다는 것입니다.

#### 속도 향상 세트 작업

집합 작업은 둘 이상의 쿼리 결과를 단일 결과 집합으로 결합합니다. 여기에는 UNION, INTERSECT 및 EXCEPT가 포함됩니다. PostgreSQL은 이 모든 것을 구현하고 속도를 높이기 위해 많은 중요한 최적화를 제공합니다.

플래너는 제한 사항을 설정 작업에 적용하여 일반적으로 멋진 색인 생성 및 속도 향상의 문을 열 수 있습니다. 이것이 어떻게 작동하는지 보여주는 다음 쿼리를 살펴보겠습니다.

```
test=# EXPLAIN SELECT *
	FROM
	(
		SELECT aid AS xid
		FROM a
		UNION ALL
		SELECT bid FROM b
	) AS Y
	WHERE xid = 3;

Append  (cost=0.29..4.20 rows=2 width=4)
  ->  Index Only Scan using idx_a on a  (cost=0.29..1.40 rows=1 width=4)
        Index Cond: (aid = 3)
  ->  Index Only Scan using idx_b on b  (cost=0.57..2.79 rows=1 width=4)
        Index Cond: (bid = 3)
```

여기서 볼 수 있는 것은 두 개의 관계가 서로 추가되었다는 것입니다. 문제는 유일한 제한이 subselect 밖에 있다는 것입니다. 그러나 PostgreSQL은 필터가 계획 아래로 더 밀릴 수 있음을 파악합니다. 따라서 xid = 3이 보조 및 입찰에 첨부되어 두 테이블 모두에서 인덱스를 사용할 수 있는 옵션이 열립니다. 두 테이블에 대한 순차 스캔을 피함으로써 쿼리가 훨씬 더 빠르게 실행됩니다.

> UNION 절과 UNION ALL 절 사이에는 차이점이 있습니다. UNION ALL 절은 맹목적으로 데이터를 추가하고 두 테이블의 결과를 전달합니다.
 
UNION 절은 중복을 필터링하므로 다릅니다. 다음 계획은 작동 방식을 보여줍니다:

```
test=# EXPLAIN SELECT *
	FROM
	(
		SELECT aid AS xid
		FROM a
		UNION SELECT bid
		FROM b
	) AS Y
	WHERE xid = 3;
	
Unique  (cost=4.23..4.24 rows=2 width=4)
  ->  Sort  (cost=4.23..4.23 rows=2 width=4)
        Sort Key: a.aid
        ->  Append  (cost=0.29..4.22 rows=2 width=4)
              ->  Index Only Scan using idx_a on a  (cost=0.29..1.40 rows=1 width=4)
                    Index Cond: (aid = 3)
              ->  Index Only Scan using idx_b on b  (cost=0.57..2.79 rows=1 width=4)
                    Index Cond: (bid = 3)
```

실행 계획은 이미 매우 매력적으로 보입니다. 두 개의 인덱스 스캔을 볼 수 있습니다. PostgreSQL은 나중에 중복을 필터링할 수 있도록 Append 노드 위에 Sort 노드를 추가해야 합니다.

> UNION 절과 UNION ALL 절의 차이점을 완전히 인식하지 못하는 많은 개발자는 PostgreSQL이 중복 항목을 필터링해야 한다는 사실을 알지 못하기 때문에 성능이 좋지 않다고 불평합니다. 이는 대용량 데이터 세트의 경우 특히 고통스럽습니다.

이 섹션에서는 가장 중요한 몇 가지 최적화에 대해 논의했습니다. 플래너 내부에서 더 많은 일이 일어나고 있음을 명심하십시오. 그러나 가장 중요한 단계를 이해하고 나면 적절한 쿼리를 작성하기가 더 쉽습니다.

## 실행 계획 이해

PostgreSQL에 구현된 몇 가지 중요한 최적화를 살펴보았으므로 이제 실행 계획을 자세히 살펴보겠습니다. 이 책에서 이미 몇 가지 실행 계획을 보았습니다. 그러나 계획을 최대한 활용하려면 이 정보를 읽을 때 체계적인 접근 방식을 개발하는 것이 중요합니다.

### 계획에 체계적으로 접근

가장 먼저 알아야 할 것은 EXPLAIN 절이 당신을 위해 많은 일을 할 수 있다는 것입니다. 이러한 기능을 최대한 활용하는 것이 좋습니다.

많은 분들이 이미 알고 계시겠지만 EXPLAIN ANALYZE 절은 쿼리를 실행하고 실제 런타임 정보를 포함한 계획을 반환합니다. 다음은 예입니다.

```
test=# EXPLAIN ANALYZE SELECT *
	FROM 
	(
		SELECT *
		FROM b
		LIMIT 1000000
	) AS b
	ORDER BY cos(bid);
	
Sort  (cost=130547.94..133047.94 rows=1000000 width=25) (actual time=1300.535..1402.580 rows=1000000 loops=1)
  Sort Key: (cos((b.bid)::double precision))
  Sort Method: quicksort  Memory: 102670kB
  ->  Subquery Scan on b  (cost=0.00..30890.10 rows=1000000 width=25) (actual time=0.021..886.864 rows=1000000 loops=1)
        ->  Limit  (cost=0.00..15890.10 rows=1000000 width=17) (actual time=0.013..740.664 rows=1000000 loops=1)
              ->  Seq Scan on b b_1  (cost=0.00..3176571.32 rows=199908832 width=17) (actual time=0.012..703.265 rows=1000000 loops=1)
Planning Time: 0.097 ms
Execution Time: 1458.492 ms

--------------------------------------------------

QUERY PLAN
Sort (cost=146173.34..148673.34 rows=1000000 width=12)
(actual time=494.028..602.733 rows=1000000 loops=1)
Sort Key: (cos((b.bid) ::double precision))
Sort Method: external merge Disk: 25496kB
-> Subquery Scan on b (cost=0.00..29425.00 rows=1000000 width=12)
(actual time=6.274..208.224 rows=1000000 loops=1)
-> Limit (cost=0.00.. 14425.00 rows=1000000 width=4)
(actual time=5.930..105.253 rows=1000000 loops=1)
-> Seq Scan on b b_1 (cost=0.00..14425.00 rows=1000000
width=4)
(actual time=0.014..55.448 rows=1000000 loops=1)
Planning Time: 0.170 ms
JIT:
Functions: 3
Options: Inlining false, Optimization false, Expressions true, Deforming
true
Timing: Generation 0.319 ms, Inlining 0.000 ms, Optimization 0.242 ms,
Emission 5.196 ms, Total 5.757 ms
Execution Time: 699.903 ms
(12 rows)
```

계획은 약간 무섭게 보이지만 당황하지 마십시오. 우리는 그것을 단계별로 진행할 것입니다. 계획을 읽을 때 안쪽에서 바깥쪽으로 읽어야 합니다. 이 예에서 실행은 b의 순차 스캔으로 시작됩니다. 여기에는 실제로 비용 블록과 실제 시간 블록이라는 두 가지 정보 블록이 있습니다. 비용 블록에는 추정치가 포함되지만 실제 시간 블록은 확실한 증거입니다. 실제 실행 시간을 보여줍니다. 여기서도 볼 수 있는 것은 PostgreSQL 12 이후로 JIT 컴파일이 기본적으로 켜져 있다는 것입니다. 쿼리는 이미 JIT 컴파일을 정당화할 만큼 시간이 많이 걸립니다.

시스템에 표시된 비용은 동일하지 않을 수 있습니다. 최적화 프로그램 통계의 작은 차이로 인해 차이가 발생할 수 있습니다. 여기서 중요한 것은 계획을 읽는 방식입니다.

인덱스 스캔에서 오는 데이터는 데이터가 너무 많지 않도록 제한 노드로 전달됩니다. 각 실행 단계에는 관련된 행 수도 표시됩니다. 보시다시피 PostgreSQL은 처음에 테이블에서 100만 행만 가져옵니다. Limit 노드는 이것이 실제로 발생하는지 확인합니다. 그러나 런타임이 이미 169밀리초로 점프했기 때문에 이 단계에서 가격표가 있습니다. 마지막으로 데이터가 정렬되는데 시간이 많이 걸립니다. 계획을 볼 때 가장 중요한 것은 실제로 시간이 손실되는 부분을 파악하는 것입니다. 그렇게 하는 가장 좋은 방법은 실제 시간 블록을 살펴보고 시간이 점프하는 위치를 파악하는 것입니다. 이 예에서 순차 스캔은 시간이 걸리지만 크게 속도를 높일 수는 없습니다. 대신 정렬이 시작되면서 시간이 급증하는 것을 볼 수 있습니다.

#### EXPLAIN을 더 장황하게 만들기

PostgreSQL에서 EXPLAIN 절의 출력은 더 많은 정보를 제공하기 위해 약간 강화될 수 있습니다. 계획에서 가능한 한 많이 추출하려면 다음 옵션을 켜는 것이 좋습니다.

```
test=# EXPLAIN (analyze, verbose, costs, timing, buffers)
SELECT * FROM a ORDER BY random();

Sort  (cost=834.39..859.39 rows=10000 width=44) (actual time=5.318..5.633 rows=10000 loops=1)
  Output: aid, aname, (random())
  Sort Key: (random())
  Sort Method: quicksort  Memory: 853kB
  Buffers: shared hit=45
  ->  Seq Scan on public.a  (cost=0.00..170.00 rows=10000 width=44) (actual time=0.033..1.373 rows=10000 loops=1)
        Output: aid, aname, random()
        Buffers: shared hit=45
Planning Time: 0.079 ms
Execution Time: 6.015 ms

--------------------------------------------------

QUERY PLAN
Sort (cost=834.39..859.39 rows=10000 width=12)
(actual time=4.124.. 4.965 rows=10000 loops=1)
Output: aid, (random())
Sort Key: (random())
Sort Method: quicksort Memory: 853kB
Buffers: shared hit=45
-> Seq Scan on public.a (cost=0.00..170.00 rows=10000 width=12)
(actual time=0.057..1.457 rows=10000 loops=1)
Output: aid, random()
Buffers: shared hit=45
Planning Time: 0.109 ms
Execution Time: 5.895 ms
(10 rows)
```

분석 true는 이전에 표시된 대로 실제로 쿼리를 실행합니다. verbose true는 계획에 더 많은 정보(예: 열 정보)를 추가합니다. 비용이 true이면 비용에 대한 정보가 표시됩니다. 타이밍 true도 마찬가지로 중요합니다. 좋은 런타임 데이터를 제공하여 계획에서 시간이 손실되는 부분을 볼 수 있기 때문입니다. 마지막으로, 매우 계몽될 수 있는 true 버퍼가 있습니다. 내 예에서는 쿼리를 실행하기 위해 수천 개의 버퍼에 액세스해야 함을 보여줍니다.

이 소개 후에는 몇 가지 추가 주제를 살펴볼 시간입니다. 계획에서 문제를 어떻게 발견할 수 있습니까?

### 문제 발견

5장, 로그 파일 및 시스템 통계에 표시된 모든 정보를 감안할 때 실생활에서 매우 중요한 몇 가지 잠재적인 성능 문제를 발견하는 것이 이미 가능합니다.

#### 런타임에 변경 사항 감지

계획을 볼 때 항상 스스로에게 물어야 하는 두 가지 질문이 있습니다.

* EXPLAIN ANALYZE 절에 의해 표시되는 런타임이 주어진 쿼리에 대해 정당화됩니까?
* 쿼리가 느린 경우 런타임은 어디로 이동합니까?

필자의 경우 순차 스캔은 2.625밀리초로 평가됩니다. 정렬은 7.199밀리초 후에 수행되므로 정렬을 완료하는 데 대략 4.5밀리초가 걸리므로 쿼리에 필요한 대부분의 런타임을 담당합니다.

쿼리 실행 시간의 점프를 찾으면 실제로 무슨 일이 일어나고 있는지 알 수 있습니다. 시간이 너무 많이 소모되는 작업 유형에 따라 그에 따라 조치를 취해야 합니다. 문제를 일으킬 수 있는 일이 너무 많기 때문에 일반적인 조언은 여기에서 불가능합니다.

#### 견적 검사

그러나 항상 해야 할 일이 있습니다. 추정치와 실수가 합리적으로 가까운지 확인해야 합니다. 어떤 경우에는 옵티마이저가 어떤 이유로 추정치가 빗나가기 때문에 잘못된 결정을 내릴 것입니다. 때때로 시스템 통계가 최신 상태가 아니기 때문에 추정치가 빗나갈 수 있습니다. 따라서 ANALYZE 절을 실행하는 것은 확실히 시작하는 것이 좋습니다. 그러나 옵티마이저 통계는 대부분 자동 진공 데몬에 의해 처리되므로 잘못된 추정을 유발하는 다른 옵션을 고려해 볼 가치가 있습니다.

테이블에 일부 데이터를 추가하는 데 도움이 되는 다음 예를 살펴보세요:

```
test=# CREATE TABLE t_estimate AS
	SELECT FROM generate_series (1, 10000) AS id;
SELECT 10000
```

10000개의 행을 로드한 후 옵티마이저 통계가 생성됩니다.

```
test # ANALYZE t_estimate;
ANALYZE
```

이제 견적을 살펴보겠습니다:

```
test=# EXPLAIN ANALYZE SELECT * FROM t_estimate WHERE cos (id) < 4;

Seq Scan on t_estimate  (cost=0.00..210.00 rows=3333 width=4) (actual time=0.693..0.693 rows=0 loops=1)
  Filter: (cos((id)::double precision) < '4'::double precision)
  Rows Removed by Filter: 10000
Planning Time: 0.350 ms
Execution Time: 0.699 ms

--------------------------------------------------

QUERY PLAN
Seq Scan on t_estimate (cost=0.00..220.00 rows=3333 width=4)
(actual time=0.010..4.006 rows=10000 loops=1)
Filter: (cos((id): :double precision) < '4':: double precision)
Planning time: 0.064 ms
Execution time: 4.701 ms
(4 rows)
```

많은 경우 PostgreSQL은 표현식이 아닌 열에 대한 통계만 가지고 있기 때문에 WHERE 절을 제대로 처리하지 못할 수 있습니다. 여기에서 볼 수 있는 것은 WHERE 절에서 반환된 데이터를 심하게 과소평가한 것입니다.

물론 다음 코드와 같이 데이터의 양을 과대평가할 수도 있습니다:

```
test # EXPLAIN ANALYZE
	SELECT *
	FROM t_estimate
	WHERE cos (id) > 4;

Seq Scan on t_estimate  (cost=0.00..210.00 rows=3333 width=4) (actual time=2.968..2.968 rows=0 loops=1)
  Filter: (cos((id)::double precision) > '4'::double precision)
  Rows Removed by Filter: 10000
Planning Time: 0.112 ms
Execution Time: 2.991 ms

--------------------------------------------------

QUERY PLAN
Seq Scan on t_estimate (cost=0.00..220.00 rows=3333 width=4) obalaba
(actual time-3.802..3.802 rows=0 loops=1)
Optimizing Queries for Good Performance
Filter: (cos((id) :: double precision) > '4': :double precision) Rows Removed by Filter: 10000
Planning time: 0.037 ms
Execution time: 3.813 ms
(5 rows)
```

계획 내부에서 이와 같은 일이 발생하면 프로세스가 잘못된 계획을 생성할 수 있습니다. 따라서 추정치가 특정 범위 내에 있는지 확인하는 것이 좋습니다.

다행히도 이 문제를 해결할 수 있는 방법이 있습니다. 다음 코드 블록을 고려하십시오.

```
test=# CREATE INDEX idx_cosine ON t_estimate (cos(id));
CREATE INDEX
```

목록은 기능적 인덱스를 생성하는 방법을 보여줍니다. 인덱스를 생성하면 PostgreSQL이 표현식의 통계를 추적하게 됩니다:

```
test=# ANALYZE t_estimate;
ANALYZE
```

이 계획은 훨씬 더 나은 성능을 보장한다는 사실 외에도 다음 코드와 같이 인덱스가 사용되지 않는 경우에도 통계를 수정합니다.

```
test=# EXPLAIN ANALYZE SELECT * FROM t_estimate WHERE cos (id) > 4;

Index Scan using idx_cosine on t_estimate  (cost=0.29..2.50 rows=1 width=4) (actual time=0.021..0.021 rows=0 loops=1)
  Index Cond: (cos((id)::double precision) > '4'::double precision)
Planning Time: 0.111 ms
Execution Time: 0.029 ms

--------------------------------------------------

QUERY PLAN
Index Scan using idx_cosine on t_estimate
(cost=0.29..8.30 rows=1 width=4)
(actual time=0.002..0.002 rows=0 loops=1)
Index Cond: (cos((id) ::double precision) > '4'::double precision)
Planning time: 0.095 ms
Execution time: 0.011 ms
(4 rows)
```

그러나 눈에 보이는 것보다 잘못된 추정이 더 많습니다. 종종 과소평가되는 한 가지 문제를 열간 상관관계라고 합니다. 두 개의 열이 포함된 간단한 예를 고려하십시오.

* 20%의 사람들이 스키를 좋아합니다.
* 인구의 20%가 아프리카 출신입니다.

우리가 아프리카의 스키어 수를 세고 싶다면 수학에 따르면 그 결과는 전체 인구의 0.2 x 0.2 = 4%가 됩니다. 그러나 아프리카에는 눈이 없습니다. 따라서 실제 결과는 확실히 더 낮을 것입니다. 관측 아프리카와 관측 스키는 통계적으로 독립적이지 않습니다. 많은 경우 PostgreSQL이 두 개 이상의 열에 걸쳐 있지 않은 열 통계를 유지한다는 사실은 나쁜 결과를 초래할 수 있습니다.

물론 기획자는 이러한 일이 가능한 한 자주 발생하지 않도록 많은 노력을 기울입니다. 그럼에도 불구하고 문제가 될 수 있습니다.

PostgreSQL 10.0부터 교차 열 상관 관계를 완전히 종식시킨 다변수 통계가 있습니다.

#### 버퍼 사용 검사

그러나 계획 자체가 문제를 일으킬 수 있는 유일한 것은 아닙니다. 많은 경우 위험한 것은 다른 수준에 숨겨져 있습니다. 메모리와 캐싱으로 인해 원치 않는 동작이 발생할 수 있으며, 이는 이 섹션에서 설명할 문제를 확인하도록 교육을 받지 않은 최종 사용자가 이해하기 어려운 경우가 많습니다.

다음은 테이블에 데이터를 무작위로 삽입하는 방법을 보여주는 예입니다. 쿼리는 무작위로 정렬된 데이터를 생성하고 새 테이블에 추가합니다.

```
test=# CREATE TABLE t_random AS
	SELECT * FROM generate_series (1, 10000000) AS id ORDER BY random();
SELECT 10000000
test=# ANALYZE t_random;
ANALYZE
```

이제 천만 개의 행이 포함된 간단한 테이블을 생성하고 옵티마이저 통계를 생성했습니다.

다음 단계에서는 소수의 행만 검색하는 간단한 쿼리가 실행됩니다.

```
test=# EXPLAIN (analyze true, buffers true, costs true, timing true)
	SELECT * FROM t_random WHERE id < 1000;

Gather  (cost=1000.00..118877.24 rows=1000 width=4) (actual time=1.745..472.791 rows=999 loops=1)
  Workers Planned: 1
  Workers Launched: 1
  Buffers: shared hit=2144 read=42104 dirtied=5290
  ->  Parallel Seq Scan on t_random  (cost=0.00..117777.24 rows=588 width=4) (actual time=1.327..402.445 rows=500 loops=2)
        Filter: (id < 1000)
        Rows Removed by Filter: 4999501
        Buffers: shared hit=2144 read=42104 dirtied=5290
Planning:
  Buffers: shared hit=5
Planning Time: 0.419 ms
Execution Time: 472.852 ms

--------------------------------------------------

QUERY PLAN
Seq Scan on t_random (cost=0.00.. 169247.71 rows=1000 width=4)
(actual time=0.976..856.663 rows=999 loops=1)
Filter: (id < 1000)
Rows Removed by Filter: 9999001
Buffers: shared hit=2080 read=42168 dirtied=14248 written=13565 Planning Time: 0.099 ms
Buffers: shared hit=5 dirtied-1
Execution Time: 856.808 ms
(7 rows)
```

데이터를 검사하기 전에 쿼리를 두 번 실행했는지 확인하십시오. 물론 여기에서 인덱스를 사용하는 것이 합리적입니다. 그러나 이 쿼리에서 PostgreSQL은 캐시 내부에서 2112개의 버퍼를 찾았고 운영 체제에서 가져와야 했던 421136개의 버퍼를 찾았습니다. 이제 두 가지 일이 발생할 수 있습니다. 운이 좋다면 운영 체제가 몇 번의 캐시 적중을 일으키고 쿼리가 빠릅니다. 파일 시스템 캐시가 운이 좋지 않으면 해당 블록을 디스크에서 가져와야 합니다. 이것은 명백해 보일 수 있습니다. 그러나 실행 시간에 거친 스윙으로 이어질 수 있습니다. 캐시에서 완전히 실행되는 쿼리는 디스크에서 임의의 블록을 천천히 수집해야 하는 쿼리보다 100배 더 빠를 수 있습니다.

간단한 예를 사용하여 이 문제를 간략하게 살펴보겠습니다. 100억 개의 행을 저장하는 전화 시스템이 있다고 가정합니다(대형 전화 사업자에게는 드문 일이 아닙니다). 데이터는 빠른 속도로 유입되고 사용자는 이 데이터를 쿼리하기를 원합니다. 100억 개의 행이 있는 경우 데이터는 메모리에 부분적으로만 들어가므로 많은 항목이 자연스럽게 디스크에서 나옵니다.

PostgreSQL에서 전화번호를 조회하는 방법을 알아보기 위해 간단한 쿼리를 실행해 보겠습니다.

```
SELECT * FROM data WHERE phone_number = '+12345678';
```

당신이 전화를 하고 있더라도 당신의 데이터는 여기저기 흩어져 있을 것입니다. 다음 통화를 시작하기 위해 전화를 끊는 경우 수천 명의 사람들이 동일한 작업을 수행하므로 두 개의 통화가 동일한 8,000블록에서 끝날 확률은 0에 가깝습니다. 동시에 100,000건의 통화가 진행 중이라고 잠시 상상해 보십시오. 디스크에서 데이터는 무작위로 배포됩니다. 전화 번호가 자주 표시되면 각 행에 대해 디스크에서 최소 하나의 블록을 가져와야 함을 의미합니다(매우 낮은 캐시 적중률이 있다고 가정). 5,000개의 행이 반환되었다고 가정해 보겠습니다. 디스크에 5,000번 이동해야 한다고 가정하면 5,000 x 5밀리초 = 25초의 실행 시간이 발생합니다. 이 쿼리의 실행 시간은 운영 체제 또는 PostgreSQL에 의해 캐시된 양에 따라 밀리초에서 30초 사이로 달라질 수 있습니다.

모든 서버가 다시 시작될 때마다 PostgreSQL 및 파일 시스템 캐시가 자연스럽게 지워지므로 노드 실패 후 실제 문제가 발생할 수 있습니다.

#### 높은 버퍼 사용량 수정

답을 구하는 질문은 이 상황을 어떻게 개선할 수 있습니까? 이를 수행하는 한 가지 방법은 CLUSTER 절을 실행하는 것입니다.

```
test=# \h CLUSTER
Command: CLUSTER
Description: cluster a table according to an index
Syntax:
CLUSTER (VERBOSE) table_name ( USING index_name]
CLUSTER (VERBOSE)
URL: https://www.postgresql.org/docs/13/sql-cluster.html
```

CLUSTER 절은 btree 인덱스와 같은 순서로 테이블을 다시 작성합니다. 분석 워크로드를 실행하는 경우 이는 의미가 있을 수 있습니다. 그러나 OLTP 시스템에서는 테이블을 다시 쓰는 동안 테이블 잠금이 필요하기 때문에 CLUSTER 절이 실행 가능하지 않을 수 있습니다.

높은 버퍼 사용량을 수정하는 것이 중요합니다. 상당한 성능 향상을 가져올 수 있습니다. 따라서 이러한 문제를 항상 주시하는 것이 좋습니다.

## 조인 이해 및 수정

조인이 중요합니다. 누구나 정기적으로 필요합니다. 결과적으로 조인은 좋은 성능을 유지하거나 달성하는 것과도 관련이 있습니다. 좋은 조인을 작성할 수 있도록 이 책에서 조인에 대해서도 배울 것입니다.

### 올바른 조인 얻기

조인 최적화에 대해 알아보기 전에 조인에서 발생하는 가장 일반적인 문제와 그 중 어떤 것이 경보를 울려야 하는지 살펴보는 것이 중요합니다.

다음은 조인이 작동하는 방식을 보여주는 간단한 테이블 구조의 예입니다.

```
test=# CREATE TABLE a (aid int);
CREATE TABLE
test=# CREATE TABLE b (bid int);
CREATE TABLE
test=# INSERT INTO a VALUES (1), (2), (3);
INSERT 0 3
test=# INSERT INTO b VALUES (2), (3), (4);
INSERT 0 3
```

몇 개의 행을 포함하는 두 개의 테이블이 생성되었습니다.

다음 예에서는 간단한 외부 조인을 보여줍니다.

```
test=# SELECT * FROM a LEFT JOIN b ON (aid = bid);
```

| aid | bid |
|---|---|
| 1 |   |
| 2 | 2 |
| 3 | 3 |

보시다시피 PostgreSQL은 왼쪽에서 모든 행을 가져와서 조인에 맞는 행만 나열합니다.

다음 예는 많은 사람들에게 의외일 수 있습니다.

```
test SELECT * FROM a LEFT JOIN b ON (aid = bid AND bid = 2); obnu aid | bid
11
21
(3 rows)
2
```

아니요, 행 수는 줄어들지 않고 일정하게 유지됩니다. 대부분의 사람들은 조인에 행이 하나만 있을 것이라고 가정하지만 이는 사실이 아니며 숨겨진 문제로 이어질 것입니다.

단순 조인을 수행하는 다음 쿼리를 고려하십시오.

```
test=# SELECT avg(aid), avg (bid)
FROM a LEFT JOIN b
ON (aid = bid AND bid = 2);
avg avg
2.0000000000000000 | 2.0000000000000000
(1 row)
```

대부분의 사람들은 평균이 단일 행을 기반으로 계산된다고 가정합니다. 그러나 앞에서 언급한 것처럼 이것은 사실이 아니며 어떤 이유로 PostgreSQL이 조인의 왼쪽에 있는 테이블을 인덱싱하지 않기 때문에 이와 같은 쿼리는 종종 성능 문제로 간주됩니다. 물론 우리는 여기서 성능 문제를 보고 있지 않습니다. 우리는 확실히 의미론적 문제를 보고 있습니다. 종종 외부 조인을 작성하는 사람들은 PostgreSQL에 무엇을 요청하는지 실제로 알지 못합니다. 따라서 제 조언은 클라이언트가 보고한 성능 문제를 공격하기 전에 항상 외부 조인의 의미론적 정확성에 대해 질문하는 것입니다.

이러한 종류의 작업이 귀하의 쿼리가 정확하고 필요한 작업을 정확하게 수행하는지 확인하는 것이 얼마나 중요한지 아무리 강조해도 지나치지 않습니다.

### 외부 조인 처리

비즈니스 관점에서 쿼리가 실제로 올바른지 확인한 후에는 옵티마이저가 외부 조인의 속도를 높이기 위해 무엇을 할 수 있는지 확인하는 것이 좋습니다. 가장 중요한 것은 PostgreSQL이 많은 경우에 속도를 높이기 위해 내부 조인을 재정렬할 수 있다는 것입니다.
극적으로. 그러나 외부 조인의 경우 이것이 항상 가능한 것은 아닙니다. 소수의 재정렬 작업만 실제로 허용됩니다.

```
(A left join B on (Pab)) innerjoin C on (Pac) = (A innerjoin C on (Pac))
left join Bon (Pab)
```

Pac는 A와 C 등을 참조하는 술어입니다. 이 경우 Pac는 B를 참조할 수 없습니다. 그렇지 않으면 변환이 무의미합니다.

* (A left join Bon (Pab)) left joi Con (Pac) = left join con
(Pac)) left join B on (Pab)
* (A left join B on (Pab)) left join Con (Pbc) = (A left join (B
left join Con (Pbc)) on (Pab)

마지막 규칙은 Pbc 술어가 모든 null Brows에 대해 실패해야 하는 경우에만 true를 유지합니다(즉, Pbc는 B의 최소한 하나의 열에 대해 엄격함). Pbc가 엄격하지 않은 경우 첫 번째 형식은 null이 아닌 C 열이 있는 일부 행을 생성하는 반면 두 번째 형식은 해당 항목을 null로 만듭니다.

일부 조인은 재정렬될 수 있지만 일반적인 유형의 쿼리는 조인 재정렬의 이점을 얻을 수 없습니다. 몇 가지 특별한 속성이 있는 다음 코드 스니펫을 고려하십시오.

```
SELECT
FROM a LEFT JOIN ON (aid = bid)
LEFT JOIN C ON (bid = cid)
LEFT JOIN D ON (cid = did)
```

조인 재정렬은 여기서 우리에게 아무런 도움이 되지 않습니다(불가능하기 때문에).

이에 접근하는 방법은 모든 외부 조인이 실제로 필요한지 여부를 확인하는 것입니다. 많은 경우 사람들이 실제로 필요하지 않은 상태에서 외부 조인을 작성합니다. 종종 비즈니스 사례는 외부 조인을 요구하지 않습니다. 이 섹션에 이어 몇 가지 더 많은 계획 옵션을 파고들 필요가 있습니다.

### join_collapse_limit 변수 이해하기

계획 프로세스 동안 PostgreSQL은 가능한 모든 조인 순서를 확인하려고 시도합니다. 많은 경우 순열이 많아 계획 프로세스가 자연스럽게 느려지기 때문에 비용이 많이 들 수 있습니다.

join_collapse_limit 변수는 개발자에게 실제로 이러한 문제를 해결하고 쿼리를 보다 간단한 방식으로 처리하는 방법을 정의할 수 있는 도구를 제공합니다.

이 설정이 무엇인지 보여주기 위해 다음과 같은 작은 예를 컴파일합니다.

```
SELECT * FROM tabi, tab2, tab3
WHERE tabi.id = tab2.id
AND tab2.ref = tab3. id;
SELECT * FROM tab1 CROSS JOIN tab2
CROSS JOIN tab3
WHERE tabi.id = tab2.id
AND tab2.ref = tab3.id;
SELECT * FROM tabi JOIN (tab2 JOIN tab3
ON (tab2.ref = tab3.id))
ON (tab1.id = tab2.id);
```

기본적으로 이 세 가지 쿼리는 동일하며 플래너에서 동일한 방식으로 처리됩니다. 첫 번째 쿼리는 암시적 조인으로 구성됩니다. 마지막 것은 명시적 조인으로만 구성됩니다. 내부적으로 플래너는 이러한 요청을 검사하고 그에 따라 조인을 주문하여 최상의 런타임을 보장합니다. 여기서 문제는 PostgreSQL이 암시적으로 계획하는 명시적 조인의 수입니다. 이것이 바로 join_collapse_limit 변수를 설정하여 플래너에게 알릴 수 있는 것입니다. 기본값은 일반 쿼리에 적합합니다. 그러나 쿼리에 매우 많은 수의 조인이 포함된 경우 이 설정을 사용하면 계획 시간을 상당히 줄일 수 있습니다. 계획 시간을 줄이는 것은 좋은 처리량을 유지하는 데 필수적일 수 있습니다.

join_collapse_limit 변수가 계획을 어떻게 변경하는지 보기 위해 다음과 같은 간단한 쿼리를 작성합니다:

```
test=# EXPLAIN WITH X AS
(
SELECT *
FROM generate_series (1, 1000) AS id
)
SELECT *
FROM X AS a
JOIN X AS bON (a.id = b.id)
JOIN X AS C ON (b.id = c.id)
JOIN X AS d ON (c.id = d.id)
JOIN X AS e ON (d.id = e. id)
JOIN X AS E ON (e.id = f.id);
```

축소 제한을 처리한 후 이제 몇 가지 추가 플래너 옵션을 살펴보겠습니다.

## 옵티마이저 설정 활성화 및 비활성화

지금까지 플래너가 수행하는 가장 중요한 최적화에 대해 자세히 설명했습니다. PostgreSQL은 수년에 걸쳐 많이 개선되었습니다. 그래도 문제가 발생할 수 있으며 사용자는 계획자가 올바른 일을 하도록 설득해야 합니다.

계획을 수정하기 위해 PostgreSQL은 계획에 상당한 영향을 미칠 몇 가지 런타임 변수를 제공합니다. 아이디어는 최종 사용자에게 계획의 특정 유형의 노드를 다른 노드보다 더 비싸게 만들 수 있는 기회를 제공하는 것입니다. 그것은 실제로 무엇을 의미합니까?
다음은 간단한 계획입니다.

```
test=# explain SELECT *
FROM generate_series (1, 100) AS a,
generate_series (1, 100) AS b
WHERE a = b;
QUERY PLAN
Hash Join (cost=2.25..4.63 rows=100 width=8)
Hash Cond: (a.a = b.b)
-> Function Scan on generate_series a (cost=0.00..1.00 rows=100 width=4)
-> Hash (cost=1.00..1.00 rows=100 width=4)
-> Function Scan on generate_series b (cost=0.00..1.00 rows=100 width=4)
(5 rows)
```

여기서 PostgreSQL은 함수를 스캔하고 해시 조인을 수행합니다. PostgreSQL 11 또는 이전 버전에서 동일한 쿼리를 실행하고 실행 계획을 보여드리겠습니다.

```
QUERY PLAN
Merge Join (cost=119.66..199.66 rows=5000 width=8)
Merge Cond: (a.a - b.b)
-> Sort (cost=59.83..62.33 rows=1000 width=4)
Sort Key: a.a
[203]
Optimizing Queries for Good Performance
-> Function Scan on generate_series a
(cost=0.00.. 10.00 rows=1000 width=4)
-> Sort (cost=59.83..62.33 rows=1000 width=4)
Sort Key: b.b
-> Function Scan on generate_series b
(cost=0.00..10.00 rows=1000 width=4)
(8 rows)
```

이 두 요금제의 차이점이 보이시나요? PostgreSQL 12에서 집합 반환 함수의 추정치는 이미 정확합니다. 이전 버전에서 옵티마이저는 여전히 집합 반환 함수가 항상 100개의 행을 반환할 것으로 추정합니다. PostgreSQL에는 결과 집합을 추정하는 데 도움이 되는 옵티마이저 지원 기능이 있습니다. 따라서 PostgreSQL 12 이상의 계획은 이전 계획보다 훨씬 우수합니다.

새로운 계획에서 우리가 볼 수 있는 것은 해시 조인이 수행된다는 것입니다. 이것은 물론 일을 수행하는 가장 효율적인 방법입니다. 그러나 우리가 옵티마이저보다 더 똑똑하다면 어떻게 될까요? 다행히 PostgreSQL에는 옵티마이저를 무시할 수 있는 수단이 있습니다. 에서 변수를 설정할 수 있습니다.
기본 비용 견적을 변경하는 연결. 작동 방식은 다음과 같습니다.

```
test=# SET enable_hashjoin to off;
SET
test=# explain SELECT *
FROM generate_series (1, 100) AS a,
generate_series (1, 100) AS b
WHERE a = b;
QUERY PLAN
Merge Join (cost=8.65..10.65 rows=100 width=8)
Merge Cond: (a.a = b.b)
-> Sort (cost=4.32..4.57 rows=100 width=4)
Sort Key: a.a
-> Function Scan on generate_series a (cost=0.00..1.00 rows=100
width=4)
-> Sort (cost=4.32..4.57 rows=100 width=4)
Sort Key: b.b
-> Function Scan on generate_series b (cost=0.00..1.00 rows=100
width=4)
(8 rows)
```

PostgreSQL은 hashjoin 함수가 나쁘다고 가정하고 비용을 무한대로 만듭니다. 따라서 병합 조인으로 대체됩니다. 그러나 병합 조인을 끌 수도 있습니다.

```
test=# explain SELECT *
FROM generate_series (1, 100) AS a, generate_series (1, 100) AS b
WHERE a = b;
QUERY PLAN
Nested Loop (cost=0.01. .226.00 rows=100 width=8)
Join Filter: (a.a = b.b)
-> Function Scan on generate_series a (cost=0.00..1.00 rows=100 width=4)
-> Function Scan on generate_series b (cost=0.00..1.00 rows=100 width=4)
(4 rows)
```

PostgreSQL에 옵션이 서서히 부족해지고 있습니다. 다음 예제에서는 중첩 루프도 끄면 어떻게 되는지 보여줍니다.

```
test=# SET enable_nestloop to off;
SET
test=# explain SELECT *
FROM generate_series (1, 100) AS a,
generate_series (1, 100) AS b
WHERE a = b;
QUERY PLAN
->
Nested Loop (cost=10000000000.00..10000000226.00 rows=100 width=8)
Join Filter: (a.a = b.b)
Function Scan on generate_series a (cost=0.00..1.00 rows=100
width=4)
Function Scan on generate_series b (cost=0.00..1.00 rows=100
width=4)
JIT:
Functions: 10
Options: Inlining true, Optimization true, Expressions true, Deforming
true
(7 rows)
```

중요한 것은 전원을 끄는 것이 실제로 꺼지는 것을 의미하지 않는다는 것입니다. 그것은 단지 엄청나게 비싸다는 것을 의미합니다. PostgreSQL에 더 저렴한 옵션이 없으면 우리가 해제한 옵션으로 대체됩니다. 그렇지 않으면 더 이상 SQL을 실행할 방법이 없습니다.

어떤 설정이 플래너에 영향을 줍니까? 다음 스위치를 사용할 수 있습니다.

```
# - Planner Method Configuration -
#enable_bitmapscan = on
#enable_hashagg = on
#enable_hashjoin - on
#enable_indexscan - on
#enable_indexonlyscan = on
#enable_material - on
#enable_mergejoin = on
#enable_nest loop = on #enable_parallel_append = on
#enable_seqscan = on
#enable_sort = on
#enable_incrementalsort = on
#enable_tidscan = on
#enable_partitionwise_join = off
#enable_partitionwise_aggregate = off
#enable_parallel_hash = on
#enable_partition_pruning = on
```

이러한 설정은 확실히 도움이 될 수 있지만 이러한 조정은 주의해서 처리해야 합니다. 개별 쿼리의 속도를 높이는 데만 사용해야 하며 전역적으로 기능을 끄지 않아야 합니다. 옵션을 끄면 상당히 빨리 불리하게 작용하고 성능이 저하될 수 있습니다. 따라서 이러한 매개변수를 변경하기 전에 두 번 생각하는 것이 좋습니다.

그러나 철저한 검색이 쿼리를 최적화하는 유일한 방법은 아닙니다. 다음 섹션에서 다룰 유전자 쿼리 최적화 프로그램도 있습니다.

### 유전자 쿼리 최적화 이해

계획 프로세스의 결과는 우수한 성과를 달성하는 데 중요합니다. 이 장에서 보았듯이 계획은 단순하지 않으며 다양한 복잡한 계산을 포함합니다. 쿼리의 영향을 받는 테이블이 많을수록 계획이 더 복잡해집니다. 테이블이 많을수록 플래너는 더 많은 선택을 할 수 있습니다. 논리적으로 계획 시간이 늘어납니다. 어느 시점에서 계획하는 데 시간이 너무 오래 걸리므로 고전적인 전체 검색을 수행하는 것이 더 이상 가능하지 않습니다. 게다가 계획 과정에서 발생하는 오류가 너무 커서 이론적으로 가장 좋은 계획을 찾는다고 해서 반드시 런타임 측면에서 최상의 계획이 되는 것은 아닙니다.

이러한 경우 GEQO(Genetic Query Optimization)가 도움이 될 수 있습니다. GEQO란? 아이디어는 자연에서 영감을 얻었고 자연적인 진화 과정과 유사합니다.

PostgreSQL은 여행하는 세일즈맨 문제처럼 이 문제에 접근하고 가능한 조인을 정수 문자열로 인코딩합니다. 예를 들어, 4-1-3-2는 4와 1을 먼저 결합한 다음 3, 2를 결합하는 것을 의미합니다. 숫자는 관계의 ID를 나타냅니다.

첫째, 유전자 최적화 프로그램은 무작위 계획 세트를 생성합니다. 그런 다음 해당 계획을 검사합니다. 나쁜 것은 버리고 좋은 것의 유전자를 바탕으로 새로운 것이 만들어집니다. 이러한 방식으로 잠재적으로 더 나은 계획이 생성됩니다. 이 과정은 원하는 만큼 반복할 수 있습니다. 하루가 끝나면 무작위 계획을 사용하는 것보다 훨씬 더 좋을 것으로 예상되는 계획이 남습니다. 다음 코드 줄과 같이 geqo 변수를 조정하여 GEQO를 켜고 끌 수 있습니다.

```
test=# SHOW gego;
geqo
on
(1 row)
test=# SET geqo To off;
SET
```

목록은 GEQO를 켜고 끄는 방법을 보여줍니다. 기본적으로 geqo 변수는 명령문이 다음 변수에 의해 제어되는 특정 수준의 복잡성을 초과하면 시작됩니다.

```
test=# SHOW geqo_threshold;
geqo_threshold
12
(1 row)
```

쿼리가 너무 커서 이 임계값에 도달하기 시작하면 이 설정을 사용하여 해당 변수를 변경할 경우 계획자가 계획을 어떻게 변경하는지 확인하는 것이 좋습니다.

그러나 일반적으로 가능한 한 GEQO를 피하고 join_collapse_limit 변수를 사용하여 조인 순서를 어떻게든 수정하여 문제를 먼저 수정하는 것이 좋습니다. 모든 쿼리가 다르기 때문에 플래너가 다양한 상황에서 어떻게 작동하는지 학습하여 실험하고 더 많은 경험을 얻는 데 확실히 도움이 됩니다.

> 조인에 대해 자세히 알아보려면 다음 링크를 참조하십시오. http://de.slideshare.net/hansjurgenschonig/postgresql-joining-1-million-tables

다음 섹션에서는 큰 데이터 세트를 더 작은 청크로 나누는 방법인 파티셔닝을 살펴보겠습니다.

## 데이터 분할

기본 8,000 블록이 주어지면 PostgreSQL은 단일 테이블 내에 최대 32TB의 데이터를 저장할 수 있습니다. 32,000개의 블록으로 PostgreSQL을 컴파일하면 단일 테이블에 최대 128TB를 넣을 수도 있습니다. 그러나 이와 같은 큰 테이블은 더 이상 반드시 편리하지 않으며 처리를 더 쉽게 하고 경우에 따라 조금 더 빠르게 하기 위해 테이블을 분할하는 것이 합리적일 수 있습니다. 버전 10.0부터 PostgreSQL은 향상된 파티셔닝을 제공하여 최종 사용자가 데이터 파티셔닝을 훨씬 쉽게 처리할 수 있도록 합니다.

이 장에서는 이전의 분할 방법과 PostgreSQL 13.0에서 사용할 수 있는 새로운 기능에 대해 설명합니다. 사람들이 PostgreSQL의 모든 향후 버전에서 더 많고 더 나은 파티셔닝을 기대할 수 있도록 파티셔닝 기능이 모든 영역에 추가되었습니다. 가장 먼저 배울 것은 고전적인 PostgreSQL 상속으로 작업하는 방법입니다.

### 상속된 테이블 생성

먼저 오래된 데이터 분할 방식에 대해 자세히 살펴보겠습니다. 이 기술을 이해하는 것은 PostgreSQL이 배후에서 실제로 하는 일에 대한 더 깊은 개요를 얻는 데 중요합니다.

파티셔닝의 장점을 더 깊이 파고들기 전에 파티션을 만드는 방법을 보여 드리겠습니다. 모든 것은 다음 명령을 사용하여 생성할 수 있는 상위 테이블로 시작합니다.

```
test=# CREATE TABLE t_data (id serial, t date, payload text);
CREATE TABLE
```

이 예에서 상위 테이블에는 세 개의 열이 있습니다. 날짜 열은 파티셔닝에 사용되지만 나중에 더 자세히 다룰 것입니다.

이제 부모 테이블이 준비되었으므로 자식 테이블을 만들 수 있습니다. 작동 방식은 다음과 같습니다.

```
test=# CREATE TABLE t_data_2016 () INHERITS (t_data);
CREATE TABLE
test=# \d t_data_2016
Table "public.t_data_2016"
Column Type | Collation Nullable | Default
( 208 )
Chapter 6
====
id | integer | | not null |
next val('t_data_id_seq' : :regclass)
t I date
payload | text 1 1
Inherits: t_data
```

테이블은 t_data_2016이라고 하며 t_data에서 상속됩니다. (). 이는 하위 테이블에 추가 열이 추가되지 않음을 의미합니다. 보시다시피 상속이란 부모의 모든 열을 자식 테이블에서 사용할 수 있음을 의미합니다. 또한 id 열은 부모로부터 시퀀스를 상속하므로 모든 자식이 동일한 번호를 공유할 수 있습니다.

테이블을 더 만들어 보겠습니다.

```
test=# CREATE TABLE t_data_2015 () INHERITS (t_data);
CREATE TABLE
test=# CREATE TABLE t_data_2014 () INHERITS (t_data);
CREATE TABLE
```

지금까지 모든 테이블은 동일하며 부모로부터 상속됩니다. 그러나 더 많은 것이 있습니다.
자식 테이블은 실제로 부모보다 더 많은 열을 가질 수 있습니다. 더 많은 필드를 추가하는 것은 간단합니다.

```
test=# CREATE TABLE t_data_2013 (special text) INHERITS (t_data); CREATE TABLE
```

이 경우 특수 열이 추가되었습니다. 부모에게는 영향을 미치지 않습니다. 그것은 단지 아이들을 풍요롭게 하고 더 많은 데이터를 보유할 수 있게 합니다.

소수의 테이블을 만든 후 행을 추가할 수 있습니다.

```
test # INSERT INTO t_data_2015 (t, payload)
VALUES ('2015-05-04', 'some data');
INSERT 0 1
```

이제 가장 중요한 것은 부모 테이블을 사용하여 자식 테이블의 모든 데이터를 찾을 수 있다는 것입니다.

```
test # SELECT * FROM t_data;
id | t I payload.
1 2015-05-04 | some data
(1 row)
```

상위 항목을 쿼리하면 간단하고 효율적인 방식으로 상위 항목 아래에 있는 모든 항목에 액세스할 수 있습니다.

PostgreSQL이 파티셔닝을 수행하는 방법을 이해하려면 계획을 살펴보는 것이 좋습니다.

```
test=# EXPLAIN SELECT * FROM t_data;
QUERY PLAN
Append (cost=0.00..106.16 rows=4411 width=40)
-> Seq Scan on t_data t_data_1 (cost=0.00..0.00 rows=1 width=40)
-> Seq Scan on t_data_2016 t_data_2 (cost=0.00..22.00 rows=1200
width=40)
-> Seq Scan on t_data_2015 t_data_3 (cost=0.00..22.00 rows=1200
width=40)
-> Seq Scan on t_data_2014 t_data_4 (cost=0.00..22.00 rows=1200
width=40)
-> Seq Scan on t_data_2013 t_data_5 (cost=0.00..18.10 rows=810 width=40)
(6 rows)
```

사실, 그 과정은 아주 간단합니다. PostgreSQL은 단순히 모든 테이블을 통합하고 우리가 보고 있는 내부 및 아래에 있는 모든 테이블의 모든 콘텐츠를 보여줍니다. 모든 테이블은 독립적이며 시스템 카탈로그를 통해 논리적으로 연결되어 있지만 옵티마이저 결정을 더 스마트하게 내릴 수 있는 방법이 있다면 어떨까요?

### 테이블 제약 조건 적용

필터가 테이블에 적용되면 어떻게 됩니까? 옵티마이저는 가능한 가장 효율적인 방법으로 이 쿼리를 실행하기 위해 무엇을 하기로 결정합니까? 다음 예는 PostgreSQL 플래너가 어떻게 작동하는지 보여줍니다.

```
test=# EXPLAIN SELECT * FROM t_data WHERE t = '2016-01-04';
QUERY PLAN
Append (cost=0.00..95.24 rows=23 width=40)
-> Seq Scan on t_data t_data_1 (cost=0.00..0.00 rows=1 width=40)
Filter: (t = '2016-01-04'::date)
-> Seq Scan on t_data_2016 t_data_2 (cost=0.00..25.00 rows=6 width=40)
Filter: (t = '2016-01-04'::date)
-> Seq Scan on t_data_2015 t_data_3 (cost=0.00..25.00 rows=6 width=40)
Filter: (t 2016-01-04'::date)
-> Seq Scan on t_data_2014 t_data_4 (cost=0.00..25.00 rows=6 width=40)
Filter: (t = '2016-01-04'::date)
-> Seq Scan on t_data_2013 t_data_5 (cost=0.00..20.12 rows=4 width=40)
Filter: (t = '2016-01-04'::date)
(11 rows)
```

PostgreSQL은 구조의 모든 파티션에 필터를 적용합니다. 테이블 이름이 테이블의 내용과 어떻게 든 관련되어 있다는 것을 알지 못합니다. 데이터베이스에서 이름은 단지 이름일 뿐이며 우리가 찾고 있는 것과는 아무 관련이 없습니다. 물론 다른 일을 하는 것에 대한 수학적 정당성이 없기 때문에 이것은 의미가 있습니다.

이제 요점은 2016년 테이블에는 2016년 데이터만 포함되고 2015년 테이블에는 2015년 데이터만 포함된다는 등의 식으로 데이터베이스를 가르칠 수 있다는 것입니다. 테이블 제약 조건은 정확히 이를 수행하기 위해 존재합니다. 그들은 PostgreSQL에 해당 테이블의 내용을 가르치므로 계획자가 이전보다 더 현명한 결정을 내릴 수 있습니다. 이 기능을 제약 조건 제외라고 하며 많은 경우 쿼리 속도를 크게 높이는 데 도움이 됩니다.

다음 목록은 테이블 제약 조건을 생성하는 방법을 보여줍니다.

```
test=# ALTER TABLE t_data_2013
ADD CHECK (t < '2014-01-01');
ALTER TABLE
test=# ALTER TABLE t_data_2014
ADD CHECK (t >= '2014-01-01' AND t < '2015-01-01');
ALTER TABLE
test=# ALTER TABLE t_data_2015
ADD CHECK (t >= '2015-01-01' AND t < '2016-01-01');
ALTER TABLE
test=# ALTER TABLE t_data_2016
ADD CHECK (t >= '2016-01-01' AND t < '2017-01-01');
ALTER TABLE
```

각 테이블에 대해 CHECK 제약 조건을 추가할 수 있습니다.

> PostgreSQL은 해당 테이블의 모든 데이터가 완벽하게 정확하고 모든 단일 행이 제약 조건을 충족하는 경우에만 제약 조건을 생성합니다. MySQL과 달리 PostgreSQL의 제약 조건은 어떤 상황에서도 심각하게 받아들여지고 존중됩니다.

PostgreSQL에서 이러한 제약 조건은 겹칠 수 있습니다. 이는 금지되지 않으며 어떤 경우에는 의미가 있을 수 있습니다. 그러나 PostgreSQL에는 더 많은 테이블을 정리할 수 있는 옵션이 있기 때문에 일반적으로 겹치지 않는 제약 조건을 갖는 것이 좋습니다.

다음은 해당 테이블 제약 조건을 추가한 후 발생하는 일입니다.

```
test=# EXPLAIN SELECT * FROM t_data WHERE t - 2016-01-04';
QUERY PLAN
Append (cost=0.00.. 25.04 rows=7 width=40)
-> Seq Scan on t_data t_data_1 (cost=0.00..0.00 rows=1 width=40)
Filter: (t. 2016-01-04'::date)
[211]
Optimizing Queries for Good Performance
-> Seq Scan on t_data_2016 t_data_2 (cost=0.00..25.00 rows=6 width=40)
Filter: (t = '2016-01-04'::date)
(5 rows)
```

플래너는 쿼리에서 많은 테이블을 제거하고 잠재적으로 데이터를 포함하는 테이블만 유지할 수 있습니다. 이 쿼리는 더 짧고 효율적인 계획에서 큰 이점을 얻을 수 있습니다. 특히 해당 테이블이 정말 큰 경우 제거하면 속도가 상당히 향상될 수 있습니다.

다음 단계에서는 이러한 구조를 수정하는 방법을 배웁니다.

### 상속된 구조 수정

때때로 데이터 구조를 수정해야 합니다. ALTER TABLE 절은 정확히 이를 수행하기 위해 있습니다. 여기서 질문은 분할된 테이블을 어떻게 수정할 수 있습니까?

기본적으로 부모 테이블을 처리하고 열을 추가하거나 제거하기만 하면 됩니다. PostgreSQL은 자동으로 이러한 변경 사항을 자식 테이블에 전파하고 다음과 같이 모든 관계가 변경되었는지 확인합니다.

```
test # ALTER TABLE t_data ADD COLUMN x int;
ALTER TABLE
test # \d t_data_2016
Table "public.t_data_2016"
Column Type | Collation | Nullable | Default
id I integer I I not null |
nextval('t_data_id_seq' :: regclass)
I date. 1 t 1 910
payload | text 1 1
x | integer | 1
Check constraints:
"t_data_2016_t_check" CHECK (t >= '2016-01-01'::date AND t <
'2017-01-01'::date)
Inherits: t_data
```

보시다시피 열은 부모에 추가되고 여기에서 자식 테이블에 자동으로 추가됩니다.

이것은 열에도 적용됩니다. 인덱스는 완전히 다른 이야기입니다. 상속된 구조에서 모든 테이블은 별도로 인덱싱되어야 합니다. 상위 테이블에 인덱스를 추가하면 상위 테이블에만 표시되며 해당 하위 테이블에는 배포되지 않습니다. 해당 테이블의 모든 열을 인덱싱하는 것은 귀하의 작업이며 PostgreSQL은 귀하를 위해 그러한 결정을 내리지 않을 것입니다. 물론 이것은 하나의 특징으로 볼 수도 있고 한계로 볼 수도 있다. 반대로 PostgreSQL은 항목을 별도로 색인화할 수 있는 유연성을 제공하므로 잠재적으로 더 효율적이라고 말할 수 있습니다. 그러나 사람들은 이러한 모든 인덱스를 하나씩 배포하는 것이 훨씬 더 많은 작업이라고 주장할 수도 있습니다.

### 파티션된 구조 안팎으로 테이블 이동

상속된 구조가 있다고 가정합니다. 데이터는 날짜별로 분할되며 최종 사용자에게 가장 최근 연도를 제공하려고 합니다. 어떤 시점에서 실제로 건드리지 않고 사용자 범위에서 일부 데이터를 제거하고 싶을 수 있습니다. 데이터를 일종의 아카이브에 넣을 수 있습니다.

PostgreSQL은 정확히 이를 달성하기 위한 간단한 수단을 제공합니다. 먼저 새 부모를 만들 수 있습니다.

```
test=# CREATE TABLE t_history (LIKE t_data);
CREATE TABLE
```

LIKE 키워드를 사용하면 테이블의 t_dat와 레이아웃이 정확히 동일한 테이블을 생성할 수 있습니다. t_data 테이블에 실제로 어떤 열이 있는지 잊어버린 경우 많은 작업을 절약할 수 있으므로 유용할 수 있습니다. 인덱스, 제약 조건 및 기본값을 포함할 수도 있습니다.

그런 다음 테이블을 이전 상위 테이블에서 멀리 이동하여 새 테이블 아래에 놓을 수 있습니다. 작동 방식은 다음과 같습니다.

```
test # ALTER TABLE t_data_2013 NO INHERIT t_data;
ALTER TABLE
test # ALTER TABLE t_data_2013 INHERIT t_history;
ALTER TABLE
```

물론 전체 프로세스는 단일 트랜잭션으로 수행되어 작업이 원자성을 유지하도록 할 수 있습니다.

### 데이터 정리

분할된 테이블의 장점 중 하나는 데이터를 빠르게 정리할 수 있다는 것입니다. 연도 전체를 삭제한다고 가정해 보겠습니다. 데이터가 그에 따라 분할되면 간단한 DROP TABLE 절이 작업을 수행할 수 있습니다.

```
test=# DROP TABLE t_data_2014;
DROP TABLE
```

보시다시피 자식 테이블을 삭제하는 것은 쉽습니다. 그러나 부모 테이블은 어떻습니까? 종속 개체가 있으므로 PostgreSQL은 예기치 않은 일이 발생하지 않도록 자연스럽게 오류를 발생시킵니다.

```
test=# DROP TABLE t_data;
ERROR: cannot drop table t_data because other objects depend on it
DETAIL: default for table t_data_2013 column id depends on
sequence t_data_id_seq
table t_data_2016 depends on table t_data
table t_data_2015 depends on table t_data
HINT: Use DROP ... CASCADE to drop the dependent objects too.
```

DROP TABLE 절은 종속 개체가 있음을 경고하고 해당 테이블 삭제를 거부합니다. 다음 예는 계단식 DROP TABLE을 사용하는 방법을 보여줍니다.

```
test=# DROP TABLE t_data CASCADE;
NOTICE: drop cascades to 3 other objects
DETAIL: drop cascades to default value for column id of table t_data_2013
drop cascades to table t_data_2016
drop cascades to table t_data_2015
DROP TABLE
```

CASCADE 절은 PostgreSQL이 부모 테이블과 함께 해당 개체를 실제로 제거하도록 하는 데 필요합니다.

고전적인 방법에 대한 이 소개에 이어 이제 고급 PostgreSQL 13.x 파티셔닝을 살펴보겠습니다.

### PostgreSQL 13.x 파티셔닝 이해

파티셔닝이 도입된 이후로 많은 것들이 PostgreSQL에 추가되었고 예전 세계에서 보았던 많은 것들이 그 이후로 자동화되거나 더 쉬워졌습니다. 그러나 이러한 것들을 보다 체계적으로 살펴보겠습니다.

수년 동안 PostgreSQL 커뮤니티는 기본 제공 파티셔닝에 대해 작업해 왔습니다. 마지막으로 PostgreSQL 10.0은 코어 내 파티셔닝의 첫 번째 구현을 제공했습니다. PostgreSQL 10에서 파티셔닝 기능은 여전히 매우 기본적이어서 많은
PostgreSQL 11, 12, 그리고 현재 13에서는 이 중요한 기능을 사용하려는 사람들이 더 쉽게 사용할 수 있도록 항목이 개선되었습니다.

파티셔닝이 어떻게 작동하는지 보여주기 위해 다음과 같이 범위 파티셔닝을 특징으로 하는 간단한 예제를 컴파일했습니다.

```
CREATE TABLE data (
payload integer
> PARTITION BY RANGE (payload);
CREATE TABLE negatives PARTITION
OF data FOR VALUES FROM (MINVALUE) TO (0) ;
CREATE TABLE positives PARTITION
OF data FOR VALUES FROM (0) TO (MAXVALUE);
```

이 예에서 한 파티션은 모든 음수 값을 보유하고 다른 파티션은 양수 값을 처리합니다. 상위 테이블을 생성하는 동안 데이터를 분할할 방법을 간단히 지정할 수 있습니다.

부모 테이블이 생성되면 파티션을 생성할 차례입니다. 그러려면 PARTITION OF 절을 추가해야 합니다. PostgreSQL 10에는 여전히 몇 가지 제한 사항이 있습니다. 가장 중요한 것은 다음과 같이 튜플(행)이 한 파티션에서 다른 파티션으로 이동할 수 없다는 것입니다.

```
UPDATE data SET payload = -10 WHERE payload - 5
```

다행히도 이 제한이 해제되었으며 PostgreSQL 11은 한 파티션에서 다른 파티션으로 행을 이동할 수 있습니다. 그러나 파티션 간에 데이터를 이동하는 것은 일반적으로 최선의 아이디어가 아닐 수 있습니다.

살펴보고 후드 아래에서 무슨 일이 일어나는지 봅시다.

```
test-# INSERT INTO data VALUES (5);
INSERT O 1
test=# SELECT * FROM data;
payload
[215]
Optimizing Queries for Good Performance
5
(1 row)
test=# SELECT * FROM positives;
payload
5
(1 row)
```

데이터가 올바른 파티션으로 이동됩니다. 값을 변경하면 파티션도 변경되는 것을 볼 수 있습니다. 다음 목록은 이에 대한 예를 보여줍니다.

```
test=# UPDATE data
SET payload = -10
WHERE payload = 5
RETURNING *;
payload
-10
(1 row)
UPDATE 1
test=# SELECT * FROM negatives;
payload
-10
(1 row)
```

모든 행은 이전 목록에 표시된 대로 올바른 테이블에 배치되었습니다.

다음으로 중요한 측면은 인덱싱과 관련이 있습니다. PostgreSQL 10에서는 모든 테이블(모든 파티션)을 별도로 인덱싱해야 했습니다. 이것은 PostgreSQL 11 이상에서 더 이상 사실이 아닙니다.

이것을 시도하고 어떤 일이 일어나는지 봅시다:

```
test=# CREATE INDEX idx_payload ON data (payload);
CREATE INDEX
test=# \d positives
Table "public.positives"
Column I Type | Collation | Nullable | Default
payload | integer | 1
Partition of: data FOR VALUES FROM (0) TO (MAXVALUE)
Indexes:
"positives_payload_idx" btree (payload)
```

여기서 볼 수 있는 것은 인덱스도 자식 테이블에 자동으로 추가되었다는 것입니다. 이는 PostgreSQL 11의 정말 중요한 기능이며 이미 PostgreSQL 11 이상으로 애플리케이션을 이동하는 사용자들에게 널리 인정받고 있습니다.

또 다른 중요한 기능은 기본 파티션을 만드는 기능입니다. 작동 방식을 보여주기 위해 두 파티션 중 하나를 삭제할 수 있습니다.

```
test=# DROP TABLE negatives;
DROP TABLE
```

그런 다음 데이터 테이블의 기본 파티션을 쉽게 만들 수 있습니다.

```
test=# CREATE TABLE p_def PARTITION OF data DEFAULT;
CREATE TABLE
```

어디에도 맞지 않는 모든 데이터는 이 기본 파티션에 저장되므로 올바른 파티션 생성을 절대 잊어서는 안 됩니다. 경험에 따르면 기본 파티션의 존재는 시간이 지남에 따라 응용 프로그램을 훨씬 더 안정적으로 만듭니다.

이 섹션에서는 파티셔닝에 대해 배웠습니다. 다음 섹션에서는
몇 가지 고급 성능 매개변수를 통해

## 좋은 쿼리 성능을 위한 매개변수 조정

좋은 쿼리를 작성하는 것은 좋은 성능을 얻기 위한 첫 번째 단계입니다. 좋은 쿼리가 없으면 성능이 저하될 가능성이 큽니다. 따라서 훌륭하고 지능적인 코드를 작성하면 최대한의 우위를 확보할 수 있습니다. 논리적 및 의미론적 관점에서 쿼리가 최적화되면 좋은 메모리 설정으로 최종 속도를 높일 수 있습니다.

이 섹션에서 우리는 더 많은 메모리가 당신을 위해 무엇을 할 수 있고 PostgreSQL이 당신을 위해 그것을 어떻게 사용할 수 있는지 배울 것입니다. 다시 말하지만, 이 섹션에서는 계획을 더 읽기 쉽게 만들기 위해 단일 코어 쿼리를 사용한다고 가정합니다. 항상 하나의 코어만 작동하도록 하려면 다음 명령을 사용하십시오.

```
test=# SET max_parallel_workers_per_gather to 0; SET
```

다음은 메모리 매개변수가 수행할 수 있는 작업을 보여주는 간단한 예입니다.

```
test=# CREATE TABLE t_test (id serial, name text); CREATE TABLE
test=# INSERT INTO t_test (name) SELECT 'hans' FROM generate_series (1, 100000); INSERT 0 100000
test=# INSERT INTO t_test (name)
SELECT 'paul' FROM generate_series (1, 100000);
INSERT 0 100000
```

hans가 포함된 100만 행이 테이블에 추가됩니다. 그러면 paul이 포함된 100만 행이 로드됩니다. 전체적으로 2백만 개의 고유 ID가 있지만 두 개의 다른 이름만 있을 것입니다.

PostgreSQL의 기본 메모리 설정을 사용하여 간단한 쿼리를 실행해 보겠습니다.

```
test=# SELECT name, count(*) FROM t_test GROUP BY 1;
name | count
hans | 100000
paul 100000
(2 rows)
```

두 개의 행이 반환되며 이는 놀라운 일이 아닙니다. 여기서 중요한 것은 결과가 아니라 PostgreSQL이 배후에서 수행하는 작업입니다.

```
test=# EXPLAIN ANALYZE SELECT name, count(*)
FROM t_test
GROUP BY 1; Ortesis pritenib
QUERY PLAN
HashAggregate (cost=4082.00..4082.01 rows=1 width=13)
(actual time=59.876..59.877 rows=2 loops=1)
Group Key: name
Peak Memory Usage: 24 kB
-> Seq Scan on t_test (cost=0.00..3082.00 rows=200000 width=5)
(actual time=0.009..24.186 rows=200000 loops=1)
Planning Time: 0.052 ms
Execution Time: 59.907 ms
(6 rows)
```

PostgreSQL은 그룹의 수가 실제로 매우 적다는 것을 알아냈습니다. 따라서 해시를 생성하고 그룹당 하나의 해시 항목을 추가하고 계산을 시작합니다. 그룹 수가 적기 때문에 해시가 정말 작으며 PostgreSQL은 각 그룹의 숫자를 증가시켜 빠르게 계산할 수 있습니다.

이름이 아닌 ID로 그룹화하면 어떻게 됩니까? 그룹의 수가 급증할 것입니다. PostgreSQL 13에서는 개선 사항이 구현되었습니다. 이제 해시가 디스크로 유출될 수 있습니다.

```
test=# EXPLAIN ANALYZE SELECT id, count(*) FROM t_test GROUP BY 1;
QUERY PLAN
HashAggregate (cost=7207.00..9988.25 rows=200000 width=12)
(actual time=76.609..140.297 rows=200000 loops=1)
Planned Partitions: 8 Peak Memory Usage: 4177 kB
Disk Usage: 7680 kB
HashAgg Batches: 8
-> Seq Scan on t_test (cost=0.00..3082.00 rows=200000 width=4)
(actual time=0.008..22.947 rows=200000 loops=1)
Planning Time: 0.115 ms
Execution Time: 270.249 ms
(6 rows)
```

이전 목록의 실행 계획은 몇 가지 유용한 통찰력을 제공합니다. 어떤 작업이 필요한지 보여줍니다.

PostgreSQL에서 대체 전략은 GroupAggregate를 사용하는 것이었습니다. 이전 동작을 쉽게 시뮬레이션할 수 있습니다.

```
test # SET enable_hashagg TO off;
SET
```

대체 계획은 다음 코드 스니펫에 나와 있습니다.

```
test # EXPLAIN ANALYZE SELECT id, count (*) FROM t_test GROUP BY 1;
QUERY PLAN
GroupAggregate (cost=23428.64..26928.64 rows=200000 width=12)
(actual time-55.259..130.352 rows=200000 loops=1)
Group Key: id
-> Sort (cost=23428.64..23928.64 rows=200000 width=4)
(actual time=55.250..75.275 rows=200000 loops=1)
Sort Key: id
Sort Method: external merge Disk: 3328kB
-> Seq Scan on t_test (cost=0.00..3082.00 rows=200000 width=4)
(actual time=0.009..21.601 rows=200000
loops=1)
Planning Time: 0.046 ms
Execution Time: 153.923 ms
(8 rows)
```

PostgreSQL은 이제 그룹 수가 훨씬 더 많다는 것을 파악하고 빠르게 전략을 변경합니다. 문제는 너무 많은 항목을 포함하는 해시가 메모리에 맞지 않는다는 것입니다.

```
test # SHOW work_mem ;
work_mem
4MB
(1 row)
```

보시다시피 work_mem 변수는 GROUP BY 절에서 사용하는 해시의 크기를 제어합니다. 항목이 너무 많기 때문에 PostgreSQL은 전체 데이터 세트를 메모리에 보유할 필요가 없는 전략을 찾아야 합니다. 해결책은 데이터를 ID별로 정렬하고 그룹화하는 것입니다. 데이터가 정렬되면 PostgreSQL은 목록 아래로 이동하여 차례로 그룹을 형성할 수 있습니다. 첫 번째 유형의 값이 계산되면 부분 결과를 읽고 내보낼 수 있습니다. 그런 다음 다음 그룹을 처리할 수 있습니다. 아래로 이동할 때 정렬된 목록의 값이 변경되면 다시 표시되지 않습니다. 따라서 시스템은 부분적 결과가 준비되었음을 알고 있습니다.

쿼리 속도를 높이기 위해 work_mem 변수에 더 높은 값을 즉시(물론 전역적으로) 설정할 수 있습니다.

```
test=# SET work_mem TO '1 GB';
SET
```

이제 계획은 다시 한 번 빠르고 효율적인 해시 집계 기능을 제공합니다.

```
test=# EXPLAIN ANALYZE SELECT id, count(*) FROM t_test GROUP BY 1;
QUERY PLAN
HashAggregate (cost=4082.00..6082.00 rows=200000 width=12)
(actual time=76.967..118.926 rows=200000 loops=1)
Group Key: id
-> Seq Scan on t_test
(cost=0.00..3082.00 rows=200000 width=4)
(actual time=0.008..13.570 rows=200000 loops=1)
Planning time: 0.073 ms
Execution time: 126.456 ms
(5 rows)
```

PostgreSQL은 데이터가 메모리에 적합하고 더 빠른 계획으로 전환된다는 것을 알고 있습니다(또는 최소한 가정합니다). 보시다시피 실행 시간이 더 짧습니다. 더 많은 해시 값을 계산해야 하기 때문에 쿼리는 GROUP BY 이름의 경우만큼 빠르지 않습니다. 대부분의 경우에 훌륭하고 안정적인 이점을 볼 수 있습니다. 이전에 언급했듯이 이 동작은 버전에 따라 약간 다릅니다.

### 정렬 속도 향상

work_mem 변수는 그룹화 속도를 높일 뿐만 아니라 또한 정렬과 같은 간단한 작업에도 매우 좋은 영향을 미칠 수 있습니다. 정렬은 전 세계의 모든 데이터베이스 시스템이 숙달한 필수 메커니즘입니다.

```
test # SET work_mem TO '1 GB';
SET
test # EXPLAIN ANALYZE SELECT * FROM t_test ORDER BY name, id;
QUERY PLAN
Sort (cost=20691.64..21191.64 rows=200000 width=9)
(actual time-36.481..47.899 rows=200000 loops=1)
Sort Key: name, id
Sort Method: quicksort Memory: 15520kB
-> Seq Scan on t_test
(cost=0.00..3082.00 rows=200000 width=9)
(actual time=0.010..14.232 rows=200000 loops=1)
Planning time: 0.037 ms
Execution time: 55.520 ms
(6 rows)
```

이제 메모리가 충분하기 때문에 데이터베이스는 메모리에서 모든 정렬을 수행하므로 프로세스 속도가 크게 빨라집니다. 정렬에 걸리는 시간은 이제 33밀리초로, 이전 쿼리에 비해 7배가 더 소요됩니다. 메모리가 많을수록 정렬 속도가 빨라지고 시스템 속도가 빨라집니다.

지금까지 데이터를 정렬하는 데 사용할 수 있는 두 가지 메커니즘인 외부 병합 디스크와 빠른 정렬 메모리를 살펴보았습니다. 이 두 가지 메커니즘 외에도 세 번째 알고리즘인 top-N 힙 정렬 메모리가 있습니다. 상위 N개 행만 제공하는 데 사용할 수 있습니다.

```
test=# EXPLAIN ANALYZE SELECT * FROM t_test ORDER BY name,
QUERY PLAN
id LIMIT 10;
Limit (cost=7403.93..7403.95 rows=10 width=9)
(actual time=31.837..31.838 rows=10 loops=1)
-> Sort (cost=7403.93..7903.93 rows=200000 width=9)
(actual time=31.836..31.837 rows=10 loops=1)
Sort Key: name, id
Sort Method: top-N heapsort Memory: 25kB
-> Seq Scan on t_test
(cost=0.00..3082.00 rows=200000 width=9)
(actual time=0.011..13.645 rows=200000 loops=1)
Planning time: 0.053 ms
Execution time: 31.856 ms
(7 rows)
```

알고리즘은 번개처럼 빠르며 전체 쿼리는 30밀리초가 조금 넘는 시간에 완료됩니다. 정렬 부분은 이제 18밀리초에 불과하므로 처음에 데이터를 읽는 것만큼 빠릅니다.

PostgreSQL 13에서 새로운 알고리즘이 추가되었습니다:

```
test=# CREATE INDEX idx_id ON t_test (id);
CREATE INDEX
test=# explain analyze SELECT * FROM t_test ORDER BY id, name;
QUERY PLAN
Incremental Sort (cost=0.46..15289.42 rows=200000 width=9) (actual time=0.047..71.622 rows=200000 loops=1)
Sort Key: id, name
Presorted Key: id
Full-sort Groups: 6250 Sort Method: quicksort
Average Memory: 26kB Peak Memory: 26kB
-> Index Scan using idx_id on t_test (cost=0.42..6289.42 rows=200000
width=9) (actual time=0.032..37.965 rows=200000 loops=1)
Planning Time: 0.165 ms
Execution Time: 83.681 ms
(7 rows)
```

데이터가 일부 변수에 의해 이미 정렬된 경우 증분 정렬이 사용됩니다. 이 경우 idx_id는 id별로 정렬된 데이터를 반환합니다. 이미 정렬된 데이터를 이름별로 정렬하기만 하면 됩니다.

work_mem 변수는 작업별로 할당됩니다. 이론적으로 쿼리에는 work_mem 변수가 두 번 이상 필요할 수 있습니다. 전역 설정이 아닙니다. 실제로는 작업별로 설정됩니다. 따라서 신중하게 설정해야 합니다.

한 가지 명심해야 할 것은 OLTP 시스템에서 work_mem 변수를 너무 높게 설정하면 서버의 메모리가 부족해질 수 있다고 주장하는 책이 많다는 것입니다. 예; 1,000명이 동시에 100MB를 정렬하면 메모리 오류가 발생할 수 있습니다. 그러나 디스크가 이를 처리할 수 있을 것으로 기대합니까? 나는 그것을 의심한다. 해결책은 하고 있는 일을 재고하는 것뿐입니다. 100MB를 동시에 1,000번 정렬하는 것은 어쨌든 OLTP 시스템에서 일어나야 하는 일이 아닙니다. 적절한 인덱스를 배포하거나 더 나은 쿼리를 작성하거나 단순히 요구 사항을 재고하는 것을 고려하십시오. 어떤 상황에서든 너무 많은 데이터를 동시에 너무 자주 정렬하는 것은 나쁜 생각입니다. 그런 일이 애플리케이션을 중지하기 전에 중지하십시오.

### 관리 작업 속도 향상

실제로 어떤 종류의 정렬 또는 메모리 할당을 수행해야 하는 더 많은 작업이 있습니다. CREATE INDEX 절과 같은 관리 항목은 work_mem 변수에 의존하지 않고 대신 maintenance_work_mem 변수를 사용합니다. 작동 방식은 다음과 같습니다.

```
test # DROP INDEX idx_id; 
DROP INDEX
test # SET maintenance_work_mem TO '1 MB';
SET
test-# \timing Tim isg Timing is on.
test=# CREATE INDEX idx_id ON t_test (id);
CREATE INDEX
Time: 104.268 ms
```

보시다시피 2백만 행에 대한 인덱스를 생성하는 데 약 100밀리초가 소요되며 이는 정말 느립니다. 따라서 maintenance_work_mem 변수를 사용하여 정렬 속도를 높일 수 있으며, 이는 본질적으로 CREATE INDEX 절이 수행하는 작업입니다.

```
test # SET maintenance_work_mem TO '1 GB';
SET
test # CREATE INDEX idx_id ON t test (id);
CREATE INDEX
Time: 46.774 ms
```

분류 기능이 많이 향상되었기 때문에 속도가 두 배로 향상되었습니다.

더 많은 메모리를 활용할 수 있는 더 많은 관리 작업이 있습니다. 가장 눈에 띄는 것은 VACUUM 절(인덱스 정리용)과 ALTER TABLE 절입니다. Maintenance_work_mem 변수에 대한 규칙은 다음과 같습니다.
work_mem 변수. 설정은 작업별로 이루어지며 필요한 메모리만 즉시 할당됩니다.

PostgreSQL 11에서는 데이터베이스 엔진에 추가 기능이 추가되었습니다. PostgreSQL은 이제 btree 인덱스를 병렬로 구축할 수 있어 큰 테이블의 인덱싱 속도를 크게 높일 수 있습니다. 병렬화를 담당하는 파라미터는 다음과 같다.

```
test=# SHOW max_parallel_maintenance_workers;
max_parallel_maintenance_workers
2
(1 row)
```

max_parallel_maintenance_workers는 CREATE INDEX에서 사용할 수 있는 작업자 프로세스의 최대 수를 제어합니다. 모든 병렬 작업에 대해 PostgreSQL은 테이블 크기에 따라 작업자 수를 결정합니다. 큰 테이블을 인덱싱할 때 인덱스 생성이 크게 향상될 수 있습니다. 광범위한 테스트를 수행하고 내 블로그 게시물(https://www.cybertec-postgresql.com/en/postgresql-parallel-create-index-for-better-performance/) 중 하나에서 결과를 요약했습니다. 여기에서 인덱스 생성과 관련된 몇 가지 중요한 성능 통찰력에 대해 배우게 됩니다.

그러나 인덱스 생성만이 동시성을 지원하는 것은 아닙니다.

## 병렬 쿼리 사용

버전 9.6부터 PostgreSQL은 병렬 쿼리를 지원합니다. 병렬 처리에 대한 이러한 지원은 시간이 지남에 따라 점진적으로 개선되었으며 버전 11에서는 이 중요한 기능에 더 많은 기능을 추가했습니다. 이 섹션에서는 병렬 처리가 작동하는 방식과 속도를 높이기 위해 수행할 수 있는 작업을 살펴보겠습니다.

세부 사항을 파헤치기 전에 다음과 같이 몇 가지 샘플 데이터를 생성해야 합니다.

```
test=# CREATE TABLE t_parallel AS
SELECT * FROM generate_series (1, 25000000) AS id;
SELECT 25000000
```

초기 데이터를 로드한 후 첫 번째 병렬 쿼리를 실행할 수 있습니다. 간단한 카운트는 병렬 쿼리가 일반적으로 어떻게 보이는지 보여줍니다.

```
test=# explain SELECT count(*) FROM t_parallel;
QUERY PLAN
Finalize Aggregate (cost=241829.17..241829.18 rows=1 width=8)
-> Gather (cost=241828.96..241829.17 rows=2 width=8)
Workers Planned: 2
-> Partial Aggregate (cost=240828.96..240828.97 rows=1 width=8)
-> Parallel Seq Scan on t_parallel (cost=0.00..214787.17 rows=10416717 width=0)
(5 rows)
```

쿼리의 실행 계획을 자세히 살펴보자. 먼저 PostgreSQL은 병렬 순차 스캔을 수행합니다. 이는 PostgreSQL이 2개 이상의 CPU를 사용하여 테이블을 처리하고(블록 단위로) 부분 집계를 생성함을 의미합니다. 수집 노드의 작업은 데이터를 수집하고 최종 집계를 수행하기 위해 전달하는 것입니다. 수집 노드는 병렬 처리의 끝입니다. 병렬 처리는 (현재) 중첩되지 않는다는 점을 언급하는 것이 중요합니다. 다른 수집 노드 내부에는 수집 노드가 있을 수 없습니다. 이 예에서 PostgreSQL은 두 개의 작업자 프로세스를 결정했습니다. 왜 그런 겁니까?

다음 변수를 고려해 보겠습니다.

```
test=# SHOW max_parallel_workers_per_gather;
max_parallel_workers_per_gather
2
(1 row)
```

max_parallel_workers_per_gather는 수집 노드 아래에서 허용되는 작업자 프로세스 수를 2로 제한합니다. 중요한 것은 테이블이 작으면 병렬 처리를 사용하지 않는다는 것입니다. 테이블의 크기는 다음 구성 설정에 정의된 대로 8MB 이상이어야 합니다.

```
test=# SHOW min_parallel_table_scan_size; min_parallel_table_scan_size
8MB
(1 row)
```

이제 병렬 처리 규칙은 다음과 같습니다. PostgreSQL이 작업자 프로세스를 하나 더 추가하려면 테이블 크기가 3배가 되어야 합니다. 즉, 4명의 작업자를 추가로 얻으려면 최소 81배의 데이터가 필요합니다. 데이터베이스 크기가 100배 증가하고 스토리지 시스템이 일반적으로 100배 빠르지 않기 때문에 이는 의미가 있습니다. 따라서 유용한 코어의 수가 다소 제한됩니다.

그러나 우리 테이블은 상당히 큽니다.

```
test=# \d+
Schema
List of relations
Name 1 Type Owner | Persistence Size | Description
public | t_parallel | table | hs
(1 row)
 permanent 864 MB
```

이 예에서 max_parallel_workers_per_gather는 코어 수를 제한합니다. 이 설정을 변경하면 PostgreSQL이 더 많은 코어를 결정합니다.

```
test=# SET max_parallel_workers_per_gather To 10;
SET
test=# explain SELECT count(*) FROM t_parallel;
QUERY PLAN
Finalize Aggregate (cost=174120.82..174120.83 rows=1 width=8)
-> Gather (cost=174120.30..174120.81 rows=5 width=8)
Workers Planned: 5
-> Partial Aggregate (cost=173120.30..173120.31 rows=1 width=8)
-> Parallel Seq Scan on t_parallel (cost=0.00..160620.24
rows=5000024 width=0)
JIT:
Functions: 4
Options: Inlining false, Optimization false, Expressions true, Deforming
true
(8 rows)
```

이 경우 우리는 5명의 작업자를 얻습니다(예상대로).

그러나 특정 테이블에 사용되는 코어 수가 훨씬 더 많아야 하는 경우가 있습니다. 200GB 데이터베이스, 1TB RAM, 단 한 명의 사용자를 상상해 보십시오. 이 사용자는 다른 사람에게 피해를 주지 않고 모든 CPU를 사용할 수 있습니다. ALTER TABLE을 사용하여 방금 논의한 내용을 무효화할 수 있습니다.

```
test=# ALTER TABLE t_parallel set (parallel_workers = 9);
ALTER TABLE
```

x3 규칙을 무시하여 원하는 CPU 수를 결정하려면 ALTER TABLE을 사용하여 CPU 수를 명시적으로 하드코딩할 수 있습니다.

max_parallel_workers_per_gather는 여전히 유효하며 상한선 역할을 합니다.

계획을 보면 실제로 코어 수가 고려된다는 것을 알 수 있습니다(볼 예정인 작업자 참조).

```
test # explain SELECT count (*) FROM t_parallel;
Finalize Aggregate (cost=146343.32..146343.33 rows=1 width=8)
-> Gather (cost=146342.39..146343.30 rows=9 width=8)
Workers Planned: 9
QUERY PLAN
->Partial Aggregate (cost=145342.39..145342.40 rows=1 width=8)
-> Parallel Seq Scan on t_parallel (cost=0.00..138397.91
rows=2777791 width=0)
JIT:
Functions: 4
Options: Inlining false, Optimization false, Expressions true, Deforming
true
rows)
Time: 2.454 ms
```

그러나 이것이 해당 코어가 실제로 사용된다는 의미는 아닙니다.

```
test # explain analyze SELECT count (*) FROM t_parallel;
QUERY PLAN
Finalize Aggregate (cost=146343.32..146343.33 rows=1 width=8)
(actual time=1375.606..1375.606 rows=1 loops=1)
-> Gather (cost=146342.39..146343.30 rows=-9 width=8)
(actual time=1374.411..1376.442 rows=8 loops=1)
Workers Planned: 9
Workers Launched: 7
-> Partial Aggregate (cost=145342.39..145342.40 rows=1 width=8)
(actual time=1347.573..1347.573 rows=1
loops=8)
rows=2777791 width=0)
->Parallel Seq Scan on t_parallel (cost=0.00..138397.91
(actual time=0.049..844.601 rows=3125000 loops=8)
Planning Time: 0.028 ms
JIT:
Functions: 18
Options: Inlining false, Optimization false, Expressions true, Deforming
true
Timing: Generation 1.703 ms, Inlining 0.000 ms, Optimization 1.119 ms,
Emission 14.707 ms, Total 17.529 ms
Execution Time: 1164.922 ms.
(12 rows)
```

보시다시피 9개의 프로세스가 계획되었음에도 불구하고 7개의 코어만 출시되었습니다. 그 이유는 무엇입니까? 이 예에서는 두 개의 변수가 더 작용합니다.

```
test # SHOW max_worker_processes;
max_worker_processes
(1 row)
8
test # SHOW max_parallel_workers;
max_parallel workers
(1 row)
```

첫 번째 프로세스는 PostgreSQL에 일반적으로 사용 가능한 작업자 프로세스 수를 알려줍니다. max_parallel 작업자는 병렬 쿼리에 사용할 수 있는 작업자 수를 나타냅니다. 두 개의 매개변수가 있는 이유는 무엇입니까? 백그라운드 프로세스는 병렬 쿼리 인프라에서만 사용되는 것이 아니라 다른 용도로도 사용할 수 있으므로 대부분의 개발자는 두 개의 매개변수를 사용하기로 결정합니다.

일반적으로 우리 Cybertec(https://www.cybertec-postgresql.com)은 60620,24 max_worker_processes를 서버의 CPU 수로 설정하는 경향이 있습니다. 더 많이 사용하는 것은 일반적으로 유익하지 않은 것 같습니다.

### PostgreSQL은 병렬로 무엇을 할 수 있습니까?

이 섹션에서 이미 언급했듯이 병렬 처리에 대한 지원은 PostgreSQL 9.6 이후로 점진적으로 향상되었습니다. 모든 버전에서 새로운 내용이 추가됩니다.

다음은 병렬로 수행할 수 있는 가장 중요한 작업입니다.

* 병렬 순차 스캔
* 병렬 인덱스 스캔(btree만 해당)
* 병렬 비트맵 힙 스캔
* 병렬 조인(모든 유형의 조인)
* 병렬 btree 생성(CREATE INDEX)
* 병렬 집계
* 병렬 추가
* 진공
* 인덱스 생성

PostgreSQL 11에서는 병렬 인덱스 생성 지원이 추가되었습니다. 정상적인 정렬 작업은 아직 완전히 병렬적이지 않습니다. 지금까지는 btree 생성만 병렬로 수행할 수 있습니다. 병렬 처리의 양을 제어하려면 다음 매개변수를 적용해야 합니다.

```
test=# SHOW max_parallel_maintenance_workers;
max_parallel_maintenance_workers
2
(1 row)
```

병렬 처리 규칙은 기본적으로 일반 작업과 동일합니다.

인덱스 생성 속도를 높이려면 인덱스 생성 및 성능과 관련된 내 블로그 게시물 중 하나를 확인하십시오. https://www.cybertec-postgresql.com/en/ postgresql-parallel-create-index-for-better-performance/.

### 실제로 병렬 처리

이제 병렬 처리의 기본 사항을 소개했으므로 실제 세계에서 병렬 처리가 의미하는 바를 배워야 합니다. 다음 쿼리를 살펴보겠습니다.

```
test=# explain SELECT * FROM t_parallel;
QUERY PLAN
Seq Scan on t_parallel (cost=0.00..360621.20 rows=25000120 width=4) (1 row)
```

PostgreSQL이 병렬 쿼리를 사용하지 않는 이유는 무엇입니까? 테이블이 충분히 크고 PostgreSQL 작업자를 사용할 수 있는데 병렬 쿼리를 사용하지 않는 이유는 무엇입니까? 대답은 프로세스 간 통신이 실제로 비용이 많이 든다는 것입니다. PostgreSQL이 프로세스 간에 행을 전달해야 하는 경우 쿼리는 실제로 단일 프로세스 모드보다 느릴 수 있습니다. 옵티마이저는 비용 매개변수를 사용하여 프로세스 간 통신을 처벌합니다.

```
#parallel_tuple_cost = 0.1
```

튜플이 프로세스 간에 이동할 때마다 계산에 0.1포인트가 추가됩니다. PostgreSQL이 강제로 병렬 쿼리를 실행하는 방법을 확인하기 위해 다음 예제를 포함했습니다.

```
test # SET force_parallel_mode TO on;
SET
test #explain SELECT * FROM t_parallel;
QUERY PLAN
Gather (cost=1000.00..2861633.20 rows=25000120 width=4)
Workers Planned: 1
Single Copy: true
-> Seq Scan on t_parallel (cost=0.00..360621.20 rows=25000120 width=4)
(4 rows)
```

보시다시피 비용은 단일 코어 모드보다 높습니다. 실제 세계에서 많은 사람들이 PostgreSQL이 단일 코어를 사용하는 이유를 궁금해 할 것이기 때문에 이것은 중요한 문제입니다.

실제 예에서 더 많은 코어가 자동으로 더 빠른 속도를 가져오지 않는다는 것을 확인하는 것도 중요합니다. 완벽한 코어 수를 찾기 위해서는 섬세한 밸런싱 작업이 필요합니다.

## JIT 컴파일 소개

JIT 컴파일은 PostgreSQL 11의 뜨거운 주제 중 하나였습니다. 이것은 주요 작업이었고 첫 번째 결과는 유망해 보입니다. 그러나 기본부터 시작하겠습니다. JIT 컴파일이란 무엇입니까? 쿼리를 실행할 때 PostgreSQL은 런타임에 많은 것을 파악해야 합니다. PostgreSQL 자체가 컴파일되면 다음에 어떤 쿼리를 실행할지 모르기 때문에 모든 시나리오에 대비해야 합니다.

코어는 모든 종류의 작업을 수행할 수 있다는 의미에서 일반적입니다. 그러나 쿼리에 있을 때 다른 임의 항목이 아닌 현재 쿼리를 가능한 한 빨리 실행하기를 원합니다. 요점은 런타임에 컴파일 시간(즉, PostgreSQL이 컴파일될 때)보다 수행해야 하는 작업에 대해 더 많이 알고 있다는 것입니다. 이것이 바로 요점입니다. JIT 컴파일이 활성화되면 PostgreSQL이 쿼리를 확인하고 시간이 충분히 소요되는 경우 쿼리에 대해 고도로 최적화된 코드가 즉석에서 생성됩니다(적시).

### JIT 구성

JIT를 사용하려면 컴파일 시에 추가해야 합니다. 다음 구성 옵션을 사용할 수 있습니다.

```
--with-llvm build with LLVM based JIT support
LLVM_CONFIG path to 1lvm-config command
```

일부 Linux 배포판은 JIT 지원이 포함된 추가 패키지를 제공합니다. JIT를 사용하려면 해당 패키지가 설치되어 있는지 확인하십시오.

JIT를 사용할 수 있는지 확인한 후에는 쿼리에 대한 JIT 컴파일을 미세 조정할 수 있도록 다음 구성 매개변수를 사용할 수 있습니다.

```
#allow JIT compilation
#JIT implementation to use.
#perform JIT compilation if
# and query more expensive, -1
disables
#jit_optimize_above_cost = 500000
is
#optimize JITed functions if query
#jit_inline_above_cost = 500000
#more expensive, -1 disables
#attempt to inline operators and
#functions if query is more
expensive,
# -1 disables
#jit = on
#jit provider = '1lvmjit'
#jit_above_cost = 100000
available
```

jit_above_cost는 예상 비용이 최소 100,000단위인 경우에만 JIT가 고려됨을 의미합니다. 왜 관련이 있습니까? 쿼리가 충분히 길지 않으면 컴파일 오버헤드가 잠재적 이득보다 훨씬 높을 수 있습니다. 따라서 최적화만 시도됩니다. 그러나 두 가지 매개변수가 더 있습니다. 쿼리가 500,000개 단위보다 더 비싼 것으로 간주되는 경우 매우 심층적인 최적화가 시도됩니다. 이 경우 함수 호출이 인라인됩니다.

이 시점에서 PostgreSQL은 JIT 백엔드로 LLVM(Low Level Virtual Machine)만 지원합니다. 향후 추가 백엔드도 사용할 수 있을 것입니다. 현재로서는 LLVM이 정말 잘 하고 있으며 전문적인 맥락에서 사용되는 대부분의 환경을 다루고 있습니다.

### 쿼리 실행

JIT가 어떻게 작동하는지 보여주기 위해 간단한 예제를 컴파일합니다. 많은 데이터를 포함하는 큰 테이블을 만드는 것부터 시작하겠습니다. JIT 컴파일은 작업이 충분히 큰 경우에만 유용하다는 것을 기억하십시오. 초보자의 경우 5천만 행이면 충분합니다. 다음 예에서는 테이블을 채우는 방법을 보여줍니다.

```
jit=# CREATE TABLE t_jit AS SELECT (random() *10000) :: int AS X, (random() *100000)::int AS Y
(random() *1000000) :: int AS z FROM generate_series (1, 50000000) AS id;
SELECT 50000000
jit=# VACUUM ANALYZE t_jit;
VACUUM
```

이 경우 random 함수를 사용하여 일부 데이터를 생성합니다. JIT의 작동 방식을 보여주고 실행 계획을 읽기 쉽게 만들기 위해 병렬 쿼리를 끌 수 있습니다. JIT는 병렬 쿼리에서 잘 작동하지만 실행 계획은 훨씬 더 긴 경향이 있습니다.

```
jit=# max_parallel_workers_per_gather to 0;
SET
jit=# SET jit to off;
SET
jit=# explain (analyze, verbose) SELECT avg(z+y-pi()), avg(y-pi()),
max (x/pi())
FROM t_jit
WHERE ((y+z))> ((y-x) *0.000001);
QUERY PLAN
Aggregate (cost=1936901.68..1936901.69 rows=1 width=24) (actual time=20617.425..20617.425 rows=1 loops=1)
Output: avg((((z + y)) :: double precision - "3.14159265358979': :double
precision))
avg(((y) :: double precision - "3.14159265358979':: double
precision))
max(((x) :: double precision / '3.14159265358979' : :double
precision))
Seq Scan on public.t_jit (cost=0.00..1520244.00 rows=16666307
width=12)
(actual time=0.061..15322.555 rows=50000000 loops=1)
Output: X, Y, Z
Filter: (((t_jit.y + tujit.z)) ::numeric > (((t_jit.y -
tujit.x)) :: numeric + 0.000001))
Planning Time: 0.078 ms
Execution Time: 20617.473 ms
(7 rows)
```

이 경우 쿼리는 20초가 소요되었습니다.

JIT 쿼리와 일반 쿼리 간의 공정한 비교를 보장하기 위해 모든 힌트 비트 등이 올바르게 설정되었는지 확인하기 위해 VACUUM 함수를 사용했습니다.

JIT를 활성화한 상태에서 이 테스트를 반복해 보겠습니다.

```
jit # SET jit TO on;
SET
jit # explain (analyze, verbose) SELECT avg (z+y-pi()), avg (y-pi()),
max (x/pi())
FROM t_jit
WHERE ((y+z))> ((y-x) *0.000001);
QUERY PLAN
Aggregate (cost=1936901.68..1936901.69 rows=1 width=24)
(actual time=15585.788..15585.789 rows=1 loops=1)
Output: avg((((z + y)) :: double precision 3.14159265358979':: double
precision)),
avg (((y) : :double precision 3.14159265358979'::double
precision)),
max (((x) :: double precision / '3.14159265358979': :double
precision))
-> Seq Scan on public.t_jit (cost=0.00..1520244.00 rows=16666307 width=12)
(actual time=81.991..13396.227 rows=50000000 loops=1)
Output: x, y, z
Filter: (((t_jit.y + t_jit.z)) :: numeric > (((t_jit.y - t_jit.x)) :: numeric 0.000001))
Planning Time: 0.135 ms ure languages JIT:
Functions: 5
Options: Inlining true, Optimization true, Expressions true, Deforming true
Timing: Generation 2.942 ms, Inlining 15.717 ms, Optimization 40.806 ms,
Emission 25.233 ms,
Total 84.698 ms
Execution Time: 15588.851 ms
(11 rows)
```

이 경우 쿼리가 이전보다 훨씬 빨라진 것을 볼 수 있습니다. 이는 이미 중요합니다. 어떤 경우에는 혜택이 더 커질 수 있습니다. 그러나 코드를 다시 컴파일하는 것은 몇 가지 추가 작업과 관련이 있으므로 모든 종류의 쿼리에 적합하지 않습니다.

PostgreSQL 옵티마이저를 이해하면 우수한 성능을 제공하는 데 매우 유용할 수 있습니다. 좋은 성능을 보장하려면 이러한 주제를 자세히 살펴보는 것이 좋습니다.

## 요약

이 장에서는 여러 쿼리 최적화에 대해 논의했습니다. 옵티마이저와 상수 폴딩, 뷰 인라인 조인 등과 같은 다양한 내부 최적화에 대해 배웠습니다. 이러한 모든 최적화는 우수한 성능에 기여하고 작업 속도를 상당히 높이는 데 도움이 됩니다.

이제 최적화에 대한 소개를 다루었으므로 다음 장인 7장, 저장 프로시저 작성에서 저장 프로시저에 대해 이야기하겠습니다. 사용자 정의 코드를 처리할 수 있는 PostgreSQL의 모든 옵션에 대해 배우게 됩니다.

