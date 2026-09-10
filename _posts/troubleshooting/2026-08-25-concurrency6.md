---
title: "[주문 동시성 문제 해결기 4] 락 범위 확산 문제 해결"
date: 2026-08-25 10:00:00 +0900
categories: [Backend, Troubleshooting]
tags: [동시성, 트러블슈팅, 락범위, Hibernate, QueryDSL]
---

> 주문 동시성 문제 해결기 (6/7)
{: .prompt-tip }

## 현재 상태

### 시리즈에서 다루는 문제

지금까지 확인한 건 두 가지다. `FOR UPDATE` + JOIN이 `product_skus`뿐 아니라 조인된 `products`까지 잠가서, 같은 상품의 다른 SKU를 사려는 트랜잭션끼리도 Product 행에서 경합한다(문제 B). 그리고 정렬로 데드락을 막았다고 본 판단(문제 A)은 실제 락 순서와 무관해, 인덱스 덕에 우연히 가려져 있을 뿐이었다.  
[2편 - 문제 A. 락의 획득 순서로 인한 데드락 위험]({% link _posts/troubleshooting/2026-08-18-concurrency2.md %})   
[3-2편 - 문제 B. FOR UPDATE + JOIN 구조 자체의 문제]({% link _posts/troubleshooting/2026-08-20-concurrency4.md %})


### 이 글에서 다루는 것

락 범위를 SKU 단위로 좁힌다. MySQL 8.0의 `FOR UPDATE OF`를 쓰면 간단하지만 JPA·QueryDSL은 이 구문을 지원하지 않아, 조회(락 없음)와 락(SKU 단독)을 쿼리 두 개로 분리하는 방식으로 간다.   
문제는 이 분리가 기존에 통과하던 재고 초과 판매 테스트를 깨뜨린다는 점이다. 락은 제대로 걸리는데 애플리케이션이 읽는 값은 레거시 상태로 남는다. 원인을 Hibernate 영속성 컨텍스트에서 찾고, 쿼리 순서를 다시 잡아 해결하는 데까지 다룬다.

### 환경

Spring Boot 3.x / JPA(Hibernate) / QueryDSL / MySQL 8.0 (InnoDB, REPEATABLE READ)


## 기존 (문제 있는 버전)

- 여러 테이블을 JOIN한 상태로 `FOR UPDATE`를 걸면, InnoDB는 조인에서 실제로 매칭되는 세 테이블의 행 모두에 락을 걸게 됨 (락 확산)
- 서로 다른 SKU를 서로 다른 사용자가 동시에 주문하려 할 때, 결국 같은 Product row에 락 경합이 생겨 동시성이 떨어짐
- 동시성 통합 테스트 결과
    - 시나리오 1 (재고 초과 판매 여부), 시나리오 2 (데드락 회피 - 락 순서 정렬 검증) 통과
    - 시나리오 3 (Product Row 락 경합 - 락 범위 검증) 실패
- `cart_items`, `product_skus`, `products` 3개 테이블 모두 락이 걸렸고, 같은 Product의 다른 SKU 주문 시 그 Product row에서 경합함
- 실측 과정(EXPLAIN, performance_schema): [3-2편]({% link _posts/troubleshooting/2026-08-20-concurrency4.md %})
- 통합 테스트 전체: [3-3편]({% link _posts/troubleshooting/2026-08-24-concurrency5.md %})


```java
@Override
public List<CartItem> findAllWithSkuForUpdate(List<Long> itemIds, Long memberId) {
    return queryFactory
        .selectFrom(cartItem)
        .join(cartItem.productSku, productSku).fetchJoin()
        .join(productSku.product, product).fetchJoin()
        .where(
            cartItem.id.in(itemIds),
            cartItem.cart.memberId.eq(memberId),
            productSku.status.ne(SkuStatus.ARCHIVED),
            product.status.ne(ProductStatus.ARCHIVED)
        )
        .orderBy(productSku.id.asc())
        .setLockMode(LockModeType.PESSIMISTIC_WRITE) // ← products까지 잠김
        .fetch();
}
```


#### EXPLAIN

- 정렬은 락이 다 걸린 후에 일어나는 후처리라 락 순서에 관여 못함
- `using_filesort: true`
- `ordering_operation`이 `nested_loop` 전체를 감싸는 구조

```sql
EXPLAIN
SELECT ci.*, ps.*, p.*
FROM cart_items ci
JOIN product_skus ps ON ci.sku_id = ps.id
JOIN products p ON ps.product_id = p.id
WHERE ci.id IN (8, 9, 10, 11)
  AND ci.cart_id IN (SELECT c.id FROM carts c WHERE c.member_id = @user1_id)
  AND ps.status <> 'ARCHIVED'
  AND p.status <> 'ARCHIVED'
ORDER BY ps.id ASC
FOR UPDATE;
```

<details markdown="1">
<summary>EXPLAIN 결과 전체 보기</summary>

```text
+----+-------+--------+------------------------+------------------------+------+-----------------------------------------------+
| id | table | type   | key                    | ref                    | rows | Extra                                          |
+----+-------+--------+------------------------+------------------------+------+-----------------------------------------------+
|  1 | c     | const  | UKj43ag...             | const                  |    1 | Using index; Using temporary; Using filesort   |
|  1 | ci    | ref    | uk_cart_item_cart_sku  | const                  |    4 | Using index condition                          |
|  1 | ps    | eq_ref | PRIMARY                | fittura.ci.sku_id      |    1 | Using where                                    |
|  1 | p     | eq_ref | PRIMARY                | fittura.ps.product_id  |    1 | Using where                                    |
+----+-------+--------+------------------------+------------------------+------+-----------------------------------------------+
```

```sql
-> Sort: ps.id
    -> Stream results  (cost=3.35 rows=2.25)
        -> Nested loop inner join  (cost=3.35 rows=2.25)
            -> Nested loop inner join  (cost=2.3 rows=3)
                -> Index lookup on ci using uk_cart_item_cart_sku (cart_id='1'), with index condition: (ci.id in (8,9,10,11))  (cost=0.9 rows=4)
                -> Filter: (ps.`status` <> 'ARCHIVED')  (cost=0.269 rows=0.75)
                    -> Single-row index lookup on ps using PRIMARY (id=ci.sku_id)  (cost=0.269 rows=1)
            -> Filter: (p.`status` <> 'ARCHIVED')  (cost=0.275 rows=0.75)
                -> Single-row index lookup on p using PRIMARY (id=ps.product_id)  (cost=0.275 rows=1)

```

```text
"ordering_operation": {
    "using_temporary_table": true,
    "using_filesort": true,
    "nested_loop": [ {c}, {ci}, {ps}, {p} ]
}
```

</details>


#### performance_schema

- 테이블 레벨 `IX`(Intention Exclusive) 락
- `cart_items` 두 인덱스에 걸쳐 락이 걸림
- `products` 1개 행 락 잠김
- `product_skus` 각각 락 잠김

```sql
START TRANSACTION;

SELECT ci.*, ps.*, p.*
FROM cart_items ci
JOIN product_skus ps ON ci.sku_id = ps.id
JOIN products p ON ps.product_id = p.id
WHERE ci.id IN (8, 9, 10, 11)
  AND ci.cart_id IN (SELECT c.id FROM carts c WHERE c.member_id = 1)
  AND ps.status <> 'ARCHIVED'
  AND p.status <> 'ARCHIVED'
ORDER BY ps.id ASC
FOR UPDATE;
```

```sql
SELECT * 
FROM performance_schema.data_locks
WHERE OBJECT_SCHEMA = 'fittura';
```

<details markdown="1">
<summary>performance_schema.data_locks 결과 전체 보기</summary>

```text
+--------------+-----------------------+-----------+---------------+-------------+-------------------------+
| OBJECT_NAME  | INDEX_NAME            | LOCK_TYPE | LOCK_MODE     | LOCK_STATUS | LOCK_DATA               |
+--------------+-----------------------+-----------+---------------+-------------+-------------------------+
| products     | NULL                  | TABLE     | IX            | GRANTED     | NULL                    |
| product_skus | NULL                  | TABLE     | IX            | GRANTED     | NULL                    |
| cart_items   | NULL                  | TABLE     | IX            | GRANTED     | NULL                    |
| cart_items   | uk_cart_item_cart_sku | RECORD    | X             | GRANTED     | supremum pseudo-record  |
| cart_items   | uk_cart_item_cart_sku | RECORD    | X             | GRANTED     | 1, 2048, 8              |
| cart_items   | uk_cart_item_cart_sku | RECORD    | X             | GRANTED     | 1, 2049, 9              |
| cart_items   | uk_cart_item_cart_sku | RECORD    | X             | GRANTED     | 1, 2050, 10             |
| cart_items   | uk_cart_item_cart_sku | RECORD    | X             | GRANTED     | 1, 2051, 11             |
| cart_items   | PRIMARY               | RECORD    | X,REC_NOT_GAP | GRANTED     | 8                       |
| cart_items   | PRIMARY               | RECORD    | X,REC_NOT_GAP | GRANTED     | 9                       |
| cart_items   | PRIMARY               | RECORD    | X,REC_NOT_GAP | GRANTED     | 10                      |
| cart_items   | PRIMARY               | RECORD    | X,REC_NOT_GAP | GRANTED     | 11                      |
| products     | PRIMARY               | RECORD    | X,REC_NOT_GAP | GRANTED     | 512                     |
| product_skus | PRIMARY               | RECORD    | X,REC_NOT_GAP | GRANTED     | 2048                    |
| product_skus | PRIMARY               | RECORD    | X,REC_NOT_GAP | GRANTED     | 2049                    |
| product_skus | PRIMARY               | RECORD    | X,REC_NOT_GAP | GRANTED     | 2050                    |
| product_skus | PRIMARY               | RECORD    | X,REC_NOT_GAP | GRANTED     | 2051                    |
+--------------+-----------------------+-----------+---------------+-------------+-------------------------+
```

</details>

---

## 해결 방법

### 1) FOR UPDATE OF table_name

- 락 범위를 SKU 단위로 한정
- MySQL 8.0부터 `SELECT ... FOR UPDATE OF table_name` 문법 지원
- JOIN된 여러 테이블 중 어느 테이블에만 락을 걸지 명시적으로 한정할 수 있음
- `product_skus`만 락이 걸리고, `cart_items`와 `products`는 락이 걸리지 않음

```sql
SELECT ci.*, ps.*, p.*
FROM cart_items ci
JOIN product_skus ps ON ci.sku_id = ps.id
JOIN products p ON ps.product_id = p.id
WHERE ci.id IN (?, ?, ?)
ORDER BY ps.id ASC
FOR UPDATE OF ps
```

#### Hibernate/QueryDSL이 해당 문법을 지원 안 함

- `LockModeType.PESSIMISTIC_WRITE` + `setLockMode()`
    - JPA 표준 API
    - 해당 API 자체에는 "특정 테이블만 락"을 지정하는 표준화된 옵션이 없음
- Hibernate가 내부적으로 `FOR UPDATE OF`를 생성해주는 기능이 있는지, QueryDSL이 이를 노출하는 API를 제공하는지 확인 필요
- QueryDSL:
    - 특정 테이블(alias)만 지정해서 락을 거는 `OF ps` 같은 구문은 생성할 수 없음
    - RDBMS(특히 MySQL, PostgreSQL 등)의 특성에 따라 조인에 참여한 모든 테이블에 락이 걸리거나, 락의 범위가 의도와 다르게 광범위해져 데드락의 원인이 될 수 있음
- Hibernate:
    - Native API(`org.hibernate.query.Query`)에는 특정 alias에 락을 거는 기능이 존재하긴 함
    - QueryDSL을 통해 이를 깔끔하게 우회하여 사용하는 것은 프레임워크의 추상화를 깨는 일이라 좋은 유지보수성을 가지기 어려움

#### 대안

1. **네이티브 쿼리로 우회**
    - `@Query(nativeQuery = true)` 또는 `EntityManager.createNativeQuery()`로 SQL을 직접 작성해서 `FOR UPDATE OF ps` 문법을 그대로 사용
    - 확실하게 원하는 SQL을 통제할 수 있는 대신, 타입 안전성(type safety)과 QueryDSL 컨벤션을 포기하게 됨
2. **쿼리 분리**
    - 락이 필요한 대상(`ProductSku`)만 별도로 `FOR UPDATE` 조회하고, `CartItem`/`Product` 조회는 락 없는 별도 쿼리로 분리
    - 쿼리 두 번이 나가는 대신, JPA 표준 API로 처리 가능하고 락 범위가 명확해짐

---

### 2) 조회(락 없음)와 락(SKU 단독)을 분리

락 범위를 SKU 단위로 한정
1. 락 범위 좁히기: 
    - 목록 결과에서 SKU id만 뽑아서, `product_skus` 테이블 하나만 대상으로 `FOR UPDATE` 실행

2. 락 확산 차단:
    - `FOR UPDATE`가 붙은 SELECT문의 FROM/JOIN 절에 `cart_items`, `products`가 아예 없음
    - MySQL이 그 테이블들의 행을 잠글 근거 자체가 없음
    - 조인해서 매칭되는 모든 테이블의 행에 락이 걸리는 게 원래 문제였는데, 이제 조인 자체를 안 하니 확산이 원천 차단됨

```java
@Override
public List<CartItem> findAllWithSkuForUpdate(List<Long> itemIds, Long memberId) {
		// 1. cartItems 조회 (조인, 락 없음)
    List<CartItem> cartItems = queryFactory
        .selectFrom(cartItem)
        .join(cartItem.productSku, productSku).fetchJoin()
        .join(productSku.product, product).fetchJoin()
        .where(
            cartItem.id.in(itemIds),
            cartItem.cart.memberId.eq(memberId),
            productSku.status.ne(SkuStatus.ARCHIVED),
            product.status.ne(ProductStatus.ARCHIVED)
        )
        .orderBy(productSku.id.asc())
        .fetch();

		// 2. cartItems.isEmpty()면 그냥 리턴
    if (cartItems.isEmpty()) {
        return cartItems;
    }

    // 락 전용 — product_skus만 잠금 (products는 조인 자체가 없어서 잠길 수 없음)
    List<Long> skuIds = cartItems.stream()
        .map(ci -> ci.getProductSku().getId())
        .distinct()
        .sorted() // 데드락 회피를 위한 락 순서 유지
        .toList();

		// 3. 비어있지 않으면 SKU들에 대해 FOR UPDATE 쿼리 전송 - 반환값 필요 없음
    queryFactory
        .selectFrom(productSku)
        .where(productSku.id.in(skuIds))
        .orderBy(productSku.id.asc())
        .setLockMode(LockModeType.PESSIMISTIC_WRITE)
        .fetch();

    return cartItems;
}
```

#### EXPLAIN

1. 락 범위 문제 - 해결
    - `table_name`이 `product_skus` 하나만 나옴
2. 락 순서 문제 - 해결된 것으로 보임
    - `"using_filesort": false`
    - `type = range`로 `PRIMARY` 인덱스를 스캔
        - PK를 오름차순으로 스캔하는 것만으로 이미 `ORDER BY id ASC` 조건이 만족 됨
        - 별도 정렬 단계(filesort) 없이 인덱스 스캔 순서 = 최종 결과 순서
        - 인덱스 스캔 도중에 각 행을 잠금(FOR UPDATE)
        → 스캔 순서와 락 획득 순서가 일치할 가능성이 높음
    
    => `ORDER BY`가 이번엔 실제로 락 순서에 관여한다고 볼 수 있음
    

```sql
EXPLAIN 
SELECT *
FROM product_skus
WHERE id IN (2048, 2049, 2050, 2051)
ORDER BY id ASC
FOR UPDATE;
```


```text
+----+--------------+-------+---------+-------------------+------+-------------+
| id | table        | type  | key     | ref               | rows | Extra       |
+----+--------------+-------+---------+-------------------+------+-------------+
|  1 | product_skus | range | PRIMARY | fittura.ci.sku_id |    4 | Using where |
+----+--------------+-------+---------+-------------------+------+-------------+
```

```sql
-> Filter: (product_skus.id in (2048,2049,2050,2051))  (cost=4.49 rows=4)
    -> Index range scan on product_skus using PRIMARY over (id = 2048) OR (id = 2049) OR (2 more)  (cost=4.49 rows=4)
```

```text
"ordering_operation": {
    "using_filesort": false,
    "table": {
      "table_name": "product_skus",
      ...
```


#### performance_schema

1. 락 확산 차단
    - `OBJECT_NAME` 열에 처음부터 끝까지 `product_skus`만 나옴
    - `products`, `cart_items`는 락에 걸리지 않음
2. 락 순서
    - 정확히 오름차순으로 락이 걸림
    - filesort 없이 인덱스 스캔 순서 = 락 순서
3. 락 종류 정상
    - `TABLE / IX`: 행 단위 락을 걸기 전에 항상 선행되는 테이블 의도 락(정상)
    - `RECORD / X,REC_NOT_GAP`: PK로 정확히 매칭되는 4개 행에 배타 락, gap lock 없음(정확한 PK 조회라 간격 잠금 불필요)

```sql
START TRANSACTION;

SELECT *
FROM product_skus
WHERE id IN (2048, 2049, 2050, 2051)
ORDER BY id ASC
FOR UPDATE;
```

```sql
SELECT * 
FROM performance_schema.data_locks
WHERE OBJECT_SCHEMA = 'fittura';
```

```text
+--------------+------------+-----------+---------------+-------------+-----------+
| OBJECT_NAME  | INDEX_NAME | LOCK_TYPE | LOCK_MODE     | LOCK_STATUS | LOCK_DATA |
+--------------+------------+-----------+---------------+-------------+-----------+
| product_skus | NULL       | TABLE     | IX            | GRANTED     | NULL      |
| product_skus | PRIMARY    | RECORD    | X,REC_NOT_GAP | GRANTED     | 2048      |
| product_skus | PRIMARY    | RECORD    | X,REC_NOT_GAP | GRANTED     | 2049      |
| product_skus | PRIMARY    | RECORD    | X,REC_NOT_GAP | GRANTED     | 2050      |
| product_skus | PRIMARY    | RECORD    | X,REC_NOT_GAP | GRANTED     | 2051      |
+--------------+------------+-----------+---------------+-------------+-----------+
```

---

### 문제: 락은 걸었지만 읽은 값은 stale인 버그

#### cartItem 목록 조회 후 productSku 방식의 문제

- 시나리오 1 테스트 코드
    
    ```java
    @Test
    @DisplayName("재고보다 많은 동시 주문이 들어와도 초과 판매되지 않음")
    void concurrentOrdersDoNotExceedStock() throws InterruptedException {
        int initialStock = 3;
        int memberCnt = 5;
    
        Product chair = ProductFixture.complete(category, "의자");
        productRepository.save(chair);
        ProductSku chairSku = ProductSkuFixture.sku(chair, 100_000L, initialStock, "RED", null);
        skuRepository.save(chairSku);
    
        List<Long> memberIds = LongStream.rangeClosed(1, memberCnt)
            .map(i -> 90000L + i)
            .boxed()
            .toList();
    
        List<Long> cartItemIds = new ArrayList<>();
        for (Long memberId : memberIds) {
            Cart cart = Cart.create(memberId);
            cartRepository.save(cart);
    
            CartItem item = CartItem.create(cart, chairSku, 1);
            cartItemRepository.save(item);
            cartItemIds.add(item.getId());
        }
    
        // ===== 동시 실행 =====
    		// 작업을 동시에 실행할 스레드 풀
    		ExecutorService executor = Executors.newFixedThreadPool(memberCount);
    		
    		// 워커 스레드 전원이 "출발 준비 완료" 상태에 도달했는지 확인하는 용도
    		CountDownLatch readyLatch = new CountDownLatch(memberCount);
    		// 출발 신호. 카운터가 1이라 countDown() 한 번으로 대기 중인 스레드 전부가 동시에 풀림
    		CountDownLatch startLatch = new CountDownLatch(1);
    		// 워커 스레드 전원이 작업(성공/실패 무관)을 끝냈는지 확인하는 용도
    		CountDownLatch doneLatch = new CountDownLatch(memberCount);
    		
    		// 여러 스레드가 동시에 add()를 호출하므로 스레드 안전한 컬렉션 필요
    		List<Throwable> failures = Collections.synchronizedList(new ArrayList<>());
    		// AtomicInteger로 원자적 증가 보장
    		AtomicInteger successCount = new AtomicInteger();
    		
    		for (int i = 0; i < memberCount; i++) {
    		    Long memberId = memberIds.get(i);
    		    Long cartItemId = cartItemIds.get(i);
    		
    		    executor.submit(() -> {
    		        try {
    		            readyLatch.countDown();   // "나 준비됐다"
    		            startLatch.await();       // 출발 신호까지 여기서 정지
    		
    		            orderFacade.createOrder(memberId, reqDto);
    		            successCount.incrementAndGet();
    		        } catch (Exception e) {
    		            failures.add(e);          // 실패 원인(타입)까지 보존
    		        } finally {
    		            doneLatch.countDown();    // 성공/실패 무관 항상 실행
    		        }
    		    });
    		}
    		
    		readyLatch.await();                  // 전원 대기 상태 도달 확인
    		startLatch.countDown();              // 출발 — 5개 동시 재개
    		boolean finished = doneLatch.await(10, TimeUnit.SECONDS); // 타임아웃 필수
    		// 이미 제출된 작업은 유지하고 새 작업만 막는 정상 종료 (강제 중단 아님)
    		executor.shutdown();
    		
    		// ===== 검증 =====
        assertThat(finished).isTrue();
    
        ProductSku result = skuRepository.findById(chairSku.getId()).orElseThrow();
        assertThat(result.getReservedQuantity()).isLessThanOrEqualTo(result.getStockQuantity());
        assertThat(successCnt.get()).isEqualTo(initialStock);
        assertThat(failures).hasSize(memberCnt - initialStock);
    }
    ```
    
- 시나리오 1 “동시 주문 시 재고 초과 판매 여부” 실패
    - 문제: 재고가 3개인데 5명이 1개씩 구매 성공 ⇒ 재고 초과 못 막음
    
    ```bash
    org.opentest4j.AssertionFailedError: 
    expected: 3
     but was: 5
    Expected :3
    Actual   :5
    ```
    
- 정상인 경우
    - 락은 5개 스레드가 순서대로(직렬화되어) 걸림. 만약 데이터가 제대로 최신화되고 있었다면, 스레드마다 자기 차례가 왔을 때 앞선 스레드가 이미 커밋한 예약 수량을 봤어야 함
    - 스레드마다 순서대로 락을 획득하면서 `reservedQty`가 0 → 1 → 2 → 3으로 점점 올라감
    
    ```
    1번째 스레드: reservedQty=0  (아직 아무도 예약 안 함)
    2번째 스레드: reservedQty=1  (1번째가 예약한 게 보임)
    3번째 스레드: reservedQty=2  (1~2번째가 예약한 게 보임)
    4번째 스레드: reservedQty=3  (재고 다 참 → 실패)
    5번째 스레드: reservedQty=3  (재고 다 참 → 실패)
    ```
    
- 검증 시점에 재고 확인 Log
    - 5개 스레드 전부 `reservedQty=0`으로 동일하게 찍힘
        
        ⇒ 5개 스레드 모두 1단계(락 걸기 전) 쿼리 시점의 스냅샷을 그대로 들고 재고 체크를 했음
        
    
    ```java
    public void validateCartItems(List<CartItem> cartItems) {
        List<ItemError> errors = new ArrayList<>();
    
        for (CartItem cartItem : cartItems) {
            ProductSku sku = cartItem.getProductSku();
            String productName = sku.getProduct().getName() + "(" + sku.getSkuIdentifier() + ")";
    
            log.info("thread={}, reservedQty={}, stockQty={}",
                Thread.currentThread().getName(),
                sku.getReservedQuantity(),
                sku.getStockQuantity());
            ...
    }
    ```
    
    ```java
    thread=pool-4-thread-1, reservedQty=0, stockQty=3
    thread=pool-4-thread-4, reservedQty=0, stockQty=3
    thread=pool-4-thread-2, reservedQty=0, stockQty=3
    thread=pool-4-thread-5, reservedQty=0, stockQty=3
    thread=pool-4-thread-3, reservedQty=0, stockQty=3
    ```
    

#### 원인

- Hibernate 영속성 컨텍스트가 "락 걸린 최신 데이터"로 갱신 안 됨
- Hibernate는 같은 트랜잭션(영속성 컨텍스트) 안에서 같은 id의 엔티티를 두 번 조회하면, 두 번째 조회 결과로 필드값을 덮어쓰지 않음
- 이미 1단계에서 로딩된 `ProductSku` 객체가 "그 인스턴스"로 자리 잡고 있으면, 
2단계 쿼리가 최신 DB 값을 가져와도 그 값으로 필드를 갱신하지 않고 그냥 기존(1단계 때 읽은) 객체를 그대로 돌려줌

⇒ 락은 DB 차원에서는 제대로 걸리지만, 자바 객체의 `reservedQuantity`/`stockQuantity`는 락 걸기 전 시점의 스냅샷 그대로

```
1. cartItem + productSku + product 조회 (락 없음)  ← 여기서 ProductSku가 "세션에 로딩"됨
2. productSku만 다시 조회 + FOR UPDATE (락 걺)
```

#### 동시 주문 시 재고 초과 판매된 문제 흐름

1. 스레드 5개가 거의 동시에 1단계 쿼리를 날림 
→ "재고 3, 예약 0"이라는 락 걸리기 전 스냅샷을 각자의 세션에 캐싱
2. 2단계에서 락은 순서대로(직렬화되어) 제대로 걸림
3. 재고 체크 로직은 여전히 1단계에서 캐싱된 오래된 값을 보고 판단함
→ 아무도 다른 스레드가 이미 예약한 걸 못 봄 
→ 5개 다 "재고 있음"으로 판단 
4. 5개 다 성공

⇒ 락은 걸렸는데 정작 그 락으로 보호하려던 데이터는 최신화가 안 된, "락은 걸었지만 읽은 값은 stale"인 버그

---

## 최종안

### 락을 먼저 걸고, 그다음에 조회

- ProductSku가 이 트랜잭션에서 처음 로딩되는 시점이 락을 건 쿼리여야 함
    
    → 그 이후 어떤 쿼리로 다시 조회하든 Hibernate가 "이미 세션에 있는, 락 걸린 최신 인스턴스"를 그대로 재사용하게 됨
    
- `productSku`가 이 트랜잭션에서 처음 로딩되는 시점 자체가 락 건 쿼리라서, 락 걸린 직후의 최신 `reservedQuantity`가 세션에 캐싱됨
- 모든 테스트 통과

```java
@Override
public List<CartItem> findAllWithSkuForUpdate(List<Long> itemIds, Long memberId) {
    // 1.sku_id만 가볍게 조회 (엔티티 로딩 없음, 프로젝션이라 영속성 컨텍스트에 안 올라감)
    List<Long> skuIds = queryFactory
        .select(cartItem.productSku.id)
        .from(cartItem)
        .where(
            cartItem.id.in(itemIds),
            cartItem.cart.memberId.eq(memberId)
        )
        .fetch();

    if (skuIds.isEmpty()) {
        return List.of();
    }

    List<Long> sortedSkuIds = skuIds.stream().distinct().sorted().toList();

    // 2. 이 시점에 ProductSku를 "처음" 로딩 → 락 걸린 채로 최신 데이터가 세션에 캐싱됨
    queryFactory
        .selectFrom(productSku)
        .where(productSku.id.in(sortedSkuIds))
        .orderBy(productSku.id.asc())
        .setLockMode(LockModeType.PESSIMISTIC_WRITE)
        .fetch();

    // 3. 이제 fetchJoin으로 조회해도, productSku는 이미 세션에 "락 걸린 최신 상태"로 있음
    // 그 인스턴스를 그대로 재사용함 (여기서 다시 로딩되며 값이 틀어질 일 없음)
    return queryFactory
        .selectFrom(cartItem)
        .join(cartItem.productSku, productSku).fetchJoin()
        .join(productSku.product, product).fetchJoin()
        .where(
            cartItem.id.in(itemIds),
            cartItem.cart.memberId.eq(memberId),
            productSku.status.ne(SkuStatus.ARCHIVED),
            product.status.ne(ProductStatus.ARCHIVED)
        )
        .orderBy(productSku.id.asc())
        .fetch();
}
```

---
## 정리

- 락 범위를 좁히는 정공법은 `FOR UPDATE OF product_skus`지만, JPA 표준 API와 QueryDSL이 특정 테이블만 잠그는 구문을 노출하지 않아 택하지 못했다. 네이티브 쿼리로 우회하는 대신, 조회와 락을 쿼리 두 개로 나눴다.
- 분리 후 `EXPLAIN`·`performance_schema`로 다시 확인하니 `product_skus` 한 테이블만 잠겼고, PK 범위 스캔이라 filesort 없이 인덱스 순서가 곧 락 순서가 됐다. 락 범위(문제 B)와 락 순서(문제 A)가 함께 정리됐다.
- 그런데 재고 초과 판매 테스트가 깨졌다. 락은 순서대로 걸렸지만, 조회 쿼리가 락 쿼리보다 먼저 `ProductSku`를 세션에 올려서, 애플리케이션이 든 `reservedQuantity`는 락 걸기 전 값 그대로였다.
- `ProductSku`가 이 트랜잭션에서 처음 로딩되는 시점 자체를 락 쿼리로 만들면, 이후 재조회는 세션에 있는 락 걸린 최신 인스턴스를 그대로 쓴다. 쿼리 순서를 그렇게 바꿔 모든 테스트를 통과시켰다.

락 범위를 좁히려던 리팩토링이 stale read라는 새 버그를 불렀고, 그건 DB 락과 ORM이 보는 값이 별개라는 데서 왔다. 시리즈 전체의 타임라인과 교훈은 총정리에서 한 번에 정리한다.