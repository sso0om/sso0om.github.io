---
title: "[주문 동시성 문제 해결기 총 정리] 재고 정합성, 데드락, 락 범위 확산"
date: 2026-08-26 10:00:00 +0900
categories: [Backend, Troubleshooting]
tags: [동시성, 트러블슈팅, Lost Update, 데드락, 락범위, 영속성컨텍스트]
---

> 주문 동시성 문제 해결기 (7/7)
{: .prompt-tip }

## 주문 동시성 문제 해결기

### 배경

가구 쇼핑몰(Fittura) 프로젝트에서, 상품은 완제품 또는 상판·하판·다리 등 구성품 단위(SKU)로도 구매할 수 있다. 여러 사용자가 같은 SKU 혹은 같은 상품의 서로 다른 SKU를 동시에 주문하는 상황에서, 재고 정합성이 실제로 보장되는지 검증하고 그 과정에서 드러난 문제들을 순차적으로 해결한 과정을 정리한다.

---

※ [1편 - 장바구니 조회 N+1 해결]({% link _posts/troubleshooting/2026-08-17-concurrency1.md %})   

### 0. 리팩토링 전

- 테이블간 관계
    
    ```text
    Category
      └─ Product (상품)
           └─ ProductSku (색상·재질별 판매 단위, 재고 보유)   ← 1:N
    Member
      └─ Cart (회원당 1개, member_id UNIQUE)
           └─ CartItem (담은 SKU + 수량)   → ProductSku 참조
    ```
    
- `OrderFacade`
    
    ```java
    @Transactional
    public Long createOrder(Long memberId, OrderCreateReqDto reqDto) {
        List<CartItem> cartItems = cartService.getItemsByIdAndMember(reqDto.cartItems(), memberId);
        orderService.validateCartItems(cartItems);
    
        Order order = orderService.createOrder(memberId, reqDto);
        for(CartItem cartItem : cartItems) {
            orderService.createOrderItem(cartItem, order);
        }
        // ...
    }
    ```
    
- `CartItemRepository`
    
    ```java
    List<CartItem> cartItems = cartItemRepository.findAllByIdInAndCart_MemberId(distinctIds, memberId);
    ```
    

### 1. 시작점: N+1 쿼리

장바구니 아이템을 조회할 때 `ProductSku`, `Product` 연관관계가 `LAZY`로 설정돼 있어, 검증 로직에서 각 필드에 접근할 때마다 추가 쿼리가 발생하는 N+1 문제가 있었다. 

- 기각한 방식: `EAGER` 전환
    - 이유: `CartItem → ProductSku → Product`로 이어지는 연쇄 참조 구조상 항상 불필요한 조인을 끌고 오게 됨
- **선택 방식:** 필요한 지점에서만 QueryDSL `fetchJoin()`으로 명시적으로 가져오기

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
        .orderBy(cartItem.id.asc())
        .fetch();
}
```

 

---

※ [2편 - 비관적 락 적용, 데드락 회피]({% link _posts/troubleshooting/2026-08-18-concurrency2.md %})   

### 2. 재고 동시성 문제: Lost Update

재고 검증 로직은 자바 객체에 값을 읽어와 계산한 뒤 다시 저장하는 구조였다. 이런 "읽고 → 판단하고 → 쓰는" 구조는 여러 트랜잭션이 같은 시점의 값을 동시에 읽고 각자 유효하다 판단해 서로의 쓰기 결과를 덮어쓰는 Lost Update에 노출된다. 

```java
public boolean isStockValid(Integer orderQuantity) {
    return this.stockQuantity - reservedQuantity - orderQuantity >= 0;
}
```

MySQL InnoDB의 기본 격리 수준인 REPEATABLE READ는 "내가 읽은 값이 트랜잭션 도중 바뀌어 보이지 않는 것"만 보장할 뿐, "다른 트랜잭션이 내가 읽은 값을 바꾸지 못하게" 막아주지는 않는다. 별도의 명시적 락이 필요했다.

- 기각한 방식: 낙관적 락(`@Version`)
    - 이유: 낙관적 락은 재고가 남아있어도 버전 충돌로 실패하고 재시도 로직이 추가로 필요해지기 때문
- **선택한 방식:** 비관적 락(`FOR UPDATE`)
    - 이유: 재고처럼 "남은 수량이 있으면 성공, 없으면 정상 실패"가 되어야 하는 도메인에는 대기(blocking) 후 순차 처리가 가능한 비관적 락이 더 적합

```java
//...
.orderBy(cartItem.id.asc()) // 초기 버전 -> 데드락 문제 예상
.setLockMode(LockModeType.PESSIMISTIC_WRITE) // 비관적 락
.fetch();
```

### 3. 락이 유발한 데드락 위험

한 주문에 여러 SKU가 담길 수 있는 도메인 특성상, 여러 행에 순차적으로 락을 거는 구조 자체가 데드락 성립 조건(상호 배제, 점유 대기, 비선점, 순환 대기)이 갖춰져 있었다.

```
회원 A 카트: [SKU 101, SKU 202]
회원 B 카트: [SKU 202, SKU 101]  ← 같은 두 SKU, 담은 순서만 반대

A: SKU 101 락 획득 → SKU 202 락 시도 (B가 보유, 대기)
B: SKU 202 락 획득 → SKU 101 락 시도 (A가 보유, 대기)
→ 순환 대기 → 데드락
```

- **회피 시도 방법:**  모든 트랜잭션이 SKU id 오름차순으로 락을 요청하도록 강제(`orderBy(productSku.id.asc())`)해 순환 대기 조건을 제거하는 방식으로 접근

```java
//...
.orderBy(productSku.id.asc()) // sku id 기준 정렬
.setLockMode(LockModeType.PESSIMISTIC_WRITE) // 비관적 락
.fetch();
```

---

※ [3-2편 - 락 범위 확산 문제 확인 (EXPLAIN, performance_schema)]({% link _posts/troubleshooting/2026-08-20-concurrency4.md %})

### 4. 가정을 검증

`EXPLAIN`과 `performance_schema.data_locks`로 직접 확인하는 절차를 거쳤다. 결과는 예상과 달랐다.

- `EXPLAIN FORMAT=JSON` 결과:
    - `ordering_operation`(ORDER BY 처리)이 `nested_loop`(조인 전체)를 감싸는 구조
    - 정렬: 조인이 다 끝나고 모든 락이 걸린 뒤에 적용되는 후처리 단계로 쓰여, 실제 락 획득 순서에는 관여하지 못함
    - 같은 이유로 `Using filesort`가 발생
    
    ```text
    "ordering_operation": {
        "using_temporary_table": true,
        "using_filesort": true,
        "nested_loop": [ {c}, {ci}, {ps}, {p} ]
    }
    ```
    
- `performance_schema.data_locks`로 실제 트랜잭션을 열어 확인한 결과:
    - `cart_items`, `product_skus`뿐 아니라 `products` 행까지 락이 걸리고 있었음
    - `FOR UPDATE` + JOIN 구조에서는 조인에 매칭되는 모든 테이블의 행이 잠기기 때문
    - 같은 상품의 서로 다른 SKU를 사려는 트랜잭션끼리도 결국 같은 `Product` 행에서 경합하는 락 확산 문제가 실재함
    
    | OBJECT_NAME | INDEX_NAME | LOCK_TYPE | LOCK_MODE | LOCK_STATUS | LOCK_DATA |
    | --- | --- | --- | --- | --- | --- |
    | products  | PRIMARY  | RECORD  | X,REC_NOT_GAP | GRANTED  | 512 |

가정만으로 "해결됐다"고 판단했다면 놓쳤을 문제를, 실행 계획과 실제 락 상태를 직접 확인하면서 발견할 수 있었다.

---

※ [3-4편 - 락 범위 확산 문제 확인 (동시성 통합 테스트)]({% link _posts/troubleshooting/2026-08-24-concurrency5.md %})

### 5. 동시성 통합 테스트

`EXPLAIN`과 `performance_schema`로 확인한 건 트랜잭션 하나를 열어둔 상태였다. 실제 동시 실행에서도 같은 결론이 나오는지 통합 테스트로 재현했다. `CountDownLatch`로 여러 스레드의 시작 시점을 맞춰 세 시나리오를 검증했다.

1. 재고 초과 판매 여부
    - 결과: 통과
    - 애초에 락과 무관하게 정상 동작하던 부분
2. **데드락 회피**(락 순서 정렬 검증)
    - 결과: 통과
    - 정렬이 실제로 효과가 있어서가 아니라 인덱스 스캔이 우연히 sku_id 오름차순으로 진행돼 락 순서가 맞아떨어진 것에 가까움
    - 인덱스나 실행 계획이 바뀌면 깨질 수 있는 **우연한 보호**
3. **Product 행 락 경합**(락 범위 검증)
    - 결과: **실패**
    - Product 행 경합이 실제 동시 실행에서도 발생
    - 같은 상품의 다른 SKU를 동시 주문했을 때, 뒤 스레드의 실행 시간이 약 2077ms로 측정돼 앞 스레드의 락 점유 시간(2000ms)과 거의 일치
    - 다만, 정확성 검증이 아닌 타이밍 기반 관찰이라 스레드 스케줄링에 따라 결과가 바뀔 수 있음
    - 문제 재현을 확인하는 1회성 검증 용도로 사용


---

※ [4편 - 락 범위 확산 문제 해결]({% link _posts/troubleshooting/2026-08-25-concurrency6.md %})

### 6. 락 범위 확산 해결

MySQL 8.0은 `FOR UPDATE OF table_name` 문법으로 조인된 테이블 중 특정 테이블에만 락을 한정할 수 있지만, JPA 표준 API(`LockModeType`)와 QueryDSL은 이를 지원하지 않는다. 네이티브 쿼리로 우회하는 대신, 조회(락 없음)와 락(SKU 단독)을 쿼리 두 번으로 분리하는 방식을 택했다.

```java
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
```

해결된 사항

- 조인 자체가 없는 단일 테이블 쿼리라 **`products`가 잠길 근거가 사라짐**
- `WHERE id IN (...)`이 PK 범위 조건이라 별도 정렬 단계 없이 인덱스 스캔 순서 자체가 오름차순이 되어 **락 순서 문제 해결**

`EXPLAIN`(`using_filesort: false`)과 `performance_schema.data_locks`(오직 `product_skus`만 락, 정확히 오름차순) 양쪽으로 재확인했다.


### 7. 리팩토링이 만든 새로운 버그

락 범위는 해결됐지만, 기존에 통과하던 **재고 초과 판매 방지 테스트가 갑자기 실패**했다. 재고 3개에 5명이 1개씩 동시 주문했는데 5건 모두 성공한 것이다.

- **원인 = Hibernate 영속성 컨텍스트**
    - 같은 트랜잭션에서 동일 id의 엔티티를 두 번 조회하면, 두 번째 조회가 최신 DB 값을 가져와도 이미 세션에 로딩된 첫 번째 인스턴스의 필드값을 덮어쓰지 않음
    - 조회 쿼리(락 없음)가 락 쿼리보다 먼저 실행되며 `ProductSku`를 세션에 먼저 로딩해버린 게 문제
    - DB 레벨 락은 정상적으로 순서대로 걸렸지만, 자바 객체가 들고 있는 `reservedQuantity`는 락 걸기 전 스냅샷 그대로였음
- 재고 체크 직전에 로그
    
    ```
    thread=pool-4-thread-1, reservedQty=0, stockQty=3
    thread=pool-4-thread-2, reservedQty=0, stockQty=3
    thread=pool-4-thread-3, reservedQty=0, stockQty=3
    thread=pool-4-thread-4, reservedQty=0, stockQty=3
    thread=pool-4-thread-5, reservedQty=0, stockQty=3
    ```
    

락이 순서대로 걸렸다면 스레드마다 `reservedQty`가 0 → 1 → 2로 올라가야 정상인데, 5개 스레드 모두 동일하게 0을 보고 있었다. **stale 데이터**를 쓰고 있다는 직접적인 증거였다.

### 8. 락을 먼저 걸고, 그다음에 조회

해결책은 순서를 다시 바꾸는 것이었다. `ProductSku`가 해당 트랜잭션에서 **처음 로딩되는 시점 자체를 락 쿼리로** 만들면, 이후 어떤 쿼리로 재조회하든 Hibernate가 이미 세션에 있는 락 걸린 최신 인스턴스를 그대로 재사용한다.

```java
// 1.sku_id만 조회 (엔티티 로딩 없음, 프로젝션이라 영속성 컨텍스트에 안 올라감)
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
```

---

## 검증 전략

1. **실행 계획**: `EXPLAIN`, `EXPLAIN FORMAT=JSON`으로 정렬과 락 획득 순서의 관계를 사전 확인
2. **실제 락 상태**: `performance_schema.data_locks`로 트랜잭션을 열어둔 채 어떤 테이블의 어떤 행이 실제로 잠기는지 직접 관찰
3. **동시성 통합 테스트**: `CountDownLatch`로 여러 스레드의 시작 시점을 동기화하고, 실제 DB 커밋까지 도달하는 트랜잭션 환경에서 재고 초과 판매 여부·데드락 발생 여부·락 범위로 인한 지연을 각각 검증

## 결과

- 재고 초과 판매 없이 정확히 재고 수량만큼만 주문 성공 (5건 동시 주문, 재고 3개 → 3건 성공)
- 서로 다른 순서로 담긴 카트를 동시 주문해도 데드락 없이 처리
- 같은 상품의 다른 SKU 동시 주문 시, 서로 블로킹되지 않고 독립적으로 처리 (Product row 락 경합 해소)

## 배운 점

- **가정과 실측은 다르다.** 
    "정렬을 걸었으니 순서가 보장될 것"이라는 이론적으로 그럴듯한 판단이 실제로는 틀렸다. `EXPLAIN`과 `performance_schema`로 직접 확인하지 않았다면 데드락 위험을 안은 채 해결됐다고 착각했을 것이다.
- **DB 레벨 락과 애플리케이션이 보는 데이터는 별개다.** 
    락이 정상적으로 걸려도, ORM의 영속성 컨텍스트 동작 방식에 따라 애플리케이션 코드는 여전히 오래된 값을 볼 수 있다.
- **한 문제의 해결책이 다른 문제를 만들 수 있다.** 
    락 범위를 좁히려는 리팩토링이 의도치 않게 stale read 버그를 만들었다. 리팩토링 후에는 관련 없어 보이는 기존 테스트까지 전부 재실행해 회귀 여부를 확인하는 과정이 반드시 필요했다.