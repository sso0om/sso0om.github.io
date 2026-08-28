---
title: "[주문 동시성 문제 해결기 2]  비관적 락 적용, 데드락 회피"
date: 2026-08-18 10:00:00 +0900
categories: [Backend, Troubleshooting]
tags: [동시성, 트러블슈팅, 트랜잭션, Lock, 데드락]
---

> 주문 동시성 문제 해결기 (2/7)
{: .prompt-tip }

## 현재 상태

### 시리즈에서 다루는 문제

재고 10개인 상품에 5개씩 동시 주문 두 건이 들어왔을 때, 둘 다 성공했는데 reservedQuantity는 5로만 남았다. 
1편에서는 이 문제에 락을 걸기 위한 준비로 조회 구조부터 손봤다(N+1 제거). 
이번 글은 그 현상이 왜 생겼는지, 원인 자체를 다룬다.


### 이 글에서 다루는 것

이 현상의 원인이 Lost Update(갱신 손실)임을 확인하고, 트랜잭션 격리 수준(REPEATABLE READ)만으로 왜 안 막히는지 본다. 그다음 낙관적 락과 비관적 락을 저울질해 비관적 락(FOR UPDATE)을 고르고, 1편에서 만든 fetch join 조회에 락을 건다.

락을 걸고 나면 새 문제가 따라온다. 한 주문에 여러 SKU가 담길 수 있는 도메인 특성상 여러 행에 순차로 락이 걸리고, 이게 데드락 조건을 만든다(문제 A). fetch join 상태에서 FOR UPDATE를 걸면 락이 조인된 여러 테이블로 번지기도 한다(문제 B).

이번 글은 문제 A를 락 순서 정렬로 잡아보는 데까지다. 그 판단이 실제로 맞는지 실측하는 건 다음 편으로 넘긴다.


### 환경

Spring Boot 3.x / JPA(Hibernate) / QueryDSL / MySQL 8.0 (InnoDB, REPEATABLE READ)

---

## 주문 시 동시성 문제

### 핵심 원인: Lost Update (갱신 손실)

- Lost Update
    - 여러 트랜잭션이 같은 데이터를 동시에 수정할 때, 나중에 커밋된 트랜잭션의 쓰기가
    앞서 커밋된 트랜잭션의 변경 내용을 덮어써서 그 변경분이 흔적도 없이 사라지는 현상
    - 각 트랜잭션은 개별적으로는 정상적으로 커밋됨. 에러도 안 나고 롤백도 안 됨.
    - 최종 데이터에는 그중 하나의 결과만 반영되고 나머지는 사라진다는 게 이 문제를 발견하기 어렵게 만드는 지점
    - 로그상으로는 둘 다 "성공"이라고 나옴
- "읽고(Read) → 판단하고 → 쓰는(Write)" 구조
    - 3단계
        1. 현재 값을 조회한다 (Read)
        2. 그 값을 기준으로 새 값을 계산한다 (Compute)
        3. 계산된 새 값을 저장한다 (Write)
    - 서로 분리된 별개의 작업(SELECT 한 번 → 애플리케이션 메모리에서의 연산 → UPDATE 한 번)으로 쪼개져 있음
    - 세 단계 사이에 락이 없으면 여러 트랜잭션이 같은 시점의 값을 동시에 읽어서 서로 "내가 읽은 시점의 값"을 기준으로 각자 독립적으로 새 값을 계산하고, 둘 중 나중에 커밋되는 쪽이 먼저 커밋된 쪽의 결과를 그대로 덮어씀
        
        ⇒ Lost Update(갱신 손실)
        
- 애플리케이션 코드가 `stockQuantity - reservedQuantity`를 자바 객체의 필드값으로 계산하기 때문에 문제에 노출
    - DB의 원자적 연산(예: `UPDATE ... SET x = x + 1`)을 쓰는 게 아님
    - 엔티티를 메모리에 읽어와서 계산한 뒤 다시 통째로 쓰는 방식(JPA의 dirty checking 기반 UPDATE)임
    - 3단계
        1. 엔티티를 SELECT로 읽어와 자바 객체 필드(`reservedQuantity`)에 담고, 
        2. 그 필드값에 자바 코드로 수량을 더한 뒤
        3. 트랜잭션 커밋 시점에 Hibernate가 변경을 감지해 최종 필드값을 통째로 UPDATE 문에 담아 보내는 구조
    - "지금 DB에 실제로 저장돼 있는 값"이 아니라 "내가 트랜잭션 도중 읽어와서 들고 있던 값"을 기준으로 새 값을 계산하게 됨
    - 그 사이 다른 트랜잭션이 값을 이미 바꿔놨어도 내 계산에는 전혀 반영되지 않음
- `ProductSku`의 재고 예약 로직:
    
    ```java
    public void reserveQuantity(Integer quantity) {
        if(!isStockValid(quantity)) {
            throw new ServiceException(ProductErrorCode.STOCK_NOT_VALID);
        }
        this.reservedQuantity += quantity;
    }
    
    public boolean isStockValid(Integer orderQuantity) {
        if (orderQuantity == null || orderQuantity < 1) {
            return false;
        }
        return this.stockQuantity - reservedQuantity - orderQuantity >= 0;
    }
    ```
    
- 재고 예약 시나리오
    
    ```
    SKU 재고 10개, 예약 0개
    
    트랜잭션 A: reservedQuantity 읽음 (0) → isStockValid(5) 계산: 10-0-5 ≥ 0 → true
    트랜잭션 B: reservedQuantity 읽음 (0) → isStockValid(5) 계산: 10-0-5 ≥ 0 → true
                                            (A, B 둘 다 같은 시점 값을 읽어 둘 다 통과)
    
    트랜잭션 A: reservedQuantity = 0+5 = 5 로 저장, 커밋
    트랜잭션 B: reservedQuantity = 0+5 = 5 로 저장, 커밋 ← A가 반영한 값을 못 보고 덮어씀
    
    결과: 
    	  실제로는 10개 주문이 들어왔는데 reservedQuantity는 5로 남음
     → 이후 남은 재고가 5개인 것처럼 계산되어, 실제로는 이미 소진된 재고를 추가로 판매 가능
    ```
    

#### 기존 코드

- `CartItem`
    
    ```java
    //...
    public class CartItem extends BaseEntity {
    
        @ManyToOne(fetch = FetchType.LAZY)
        @JoinColumn(name = "cart_id", nullable = false)
        private Cart cart;
    
        @ManyToOne(fetch = FetchType.LAZY)
        @JoinColumn(name = "sku_id", nullable = false)
        private ProductSku productSku;
    //...
    }
    ```
    
- `ProductSku`
    
    ```java
    //...
    public class ProductSku extends BaseEntity {
    
        @ManyToOne(fetch = FetchType.LAZY)
        @JoinColumn(name = "product_id", nullable = false)
        private Product product;
    
        @Column(nullable = false)
        private Long price;
    
        @Column(nullable = false)
        private Integer stockQuantity = 0;
    
        @Column(nullable = false)
        private Integer reservedQuantity = 0;
    		//...
    		public boolean isStockValid(Integer orderQuantity) {
            if (orderQuantity == null || orderQuantity < 1) {
                return false;
            }
            return this.stockQuantity - reservedQuantity - orderQuantity >= 0;
        }
        //...
    }
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
        cartService.deleteCartItems(cartItems);
        orderService.createOrderAddress(order, reqDto.orderAddress());
        orderService.calcAmount(order);
    
        return order.getId();
    }
    ```
    

#### 트랜잭션 격리 수준만으로는 해결이 안 되는 이유

- MySQL InnoDB의 기본 격리 수준은 REPEATABLE READ
    - "한 트랜잭션 내에서 같은 쿼리를 반복 실행해도 같은 결과를 본다"는 것을 보장
    - 서로 다른 트랜잭션이 같은 row를 동시에 읽고 각자 쓰는 것 자체를 막아주지는 않음
- "내가 읽은 게 중간에 바뀌어 보이지 않게" 해주는 것 O
- "남이 내가 읽은 값을 바꾸지 못하게" 막아주는 것 X
    - 명시적인 락(`SELECT ... FOR UPDATE`)이나 버전 검증(`@Version`)이 별도로 필요

---

### 프로젝트에서 문제가 더 두드러지는 지점

#### **1) 검증과 예약이 분리된 두 단계로 실행됨**

- 단계
    1. `validateCartItems()`에서 재고를 확인
    2. 루프에서 실제로 `reserveQuantity()`를 호출
- 단계 사이의 시간 간격이 넓을수록(카트 아이템이 많을수록, 다른 로직이 끼어들수록) 동시성 충돌이 발생할 수 있는 시간 창(window)이 넓어짐
    - 이 자체가 별도의 버그는 아님 (같은 트랜잭션 안에서 실행되므로)
    - 락 없이 실행된다면 이 구조가 문제를 더 자주 드러나게 만드는 요인

```java
// OrderFacade.createOrder()
orderService.validateCartItems(cartItems);   // 1단계: 검증
...
orderService.createOrderItem(cartItem, order); // 2단계: 예약 (내부에서 reserveQuantity 호출)
```

#### **2) 한 주문에 여러 SKU가 포함될 수 있는 도메인 특성**

- 단계
    1. 카트에 여러 SKU를 담아 일괄 주문
    2. 한 번의 주문 처리(트랜잭션)가 여러 SKU row에 순차적으로 접근
- 여러 row에 순차적으로 락을 거는 구조 자체가 데드락 조건(Coffman의 점유 대기 조건)을 만들어낼 잠재 요인이 됨

#### **3) 재고 관련 필드가 두 개로 분리되어 있음**

- 재고 관련 필드
    1. `stockQuantity`: 실 재고
    2. `reservedQuantity`: 예약 재고
- 실제 판매 가능 여부:
    - 차감 결과(`stockQuantity - reservedQuantity`)로 판단
    - 필드가 하나가 아니라 두 개의 조합으로 판매 가능 여부가 결정되는 구조

#### 원인 요약

| 원인 | 설명 |
| --- | --- |
| Lost Update | 재고 필드를 애플리케이션에서 읽고 계산 후 다시 쓰는 구조
  → 동시 읽기 시 갱신 손실 발생 |
| 격리 수준의 한계 | REPEATABLE READ는 동시 쓰기 충돌 자체를 막지 못함
  → 별도 락 필요 |
| 검증-예약 분리 | 재고 확인과 실제 예약이 별개 호출로 이뤄져 충돌 가능 |
| 다중 SKU 주문 | 한 트랜잭션이 여러 SKU row에 순차 접근
  → 데드락 조건 성립 가능 |

---

### 해결 방법

#### 1) 엔티티 기반 + 비관적 락

- **채택**:
    - 현재 트래픽 규모에서는 처리량보다 도메인 로직 응집이 우선이라 판단
    - 대규모 트래픽 상황이 되면 방식 B 재검토 여지 있음
- 엔티티 안에서 계산하는 방식
- `FOR UPDATE`
    - 비관적 락
    - "충돌이 날 것"이라고 가정하고, 데이터를 읽는 시점에 바로 락을 걸어서 다른 트랜잭션이 손 못 대게 막는 방식
    - MySQL 8.0부터는 `FOR UPDATE OF 테이블별칭` 문법을 지원
    - 조인을 하더라도 특정 테이블의 레코드에만 락을 한정할 수 있음
- 계산이 이뤄지는 구간에 락이 걸려 있음
    1. `FOR UPDATE`로 해당 row를 잠금
    2. 트랜잭션이 끝날 때까지 다른 트랜잭션은 같은 row를 읽지 못하고 대기
    
    ⇒ "동시에 같은 값을 읽어서 서로 덮어쓰는" Lost Update 상황 자체가 원천 차단
    
- 장점:
    - 도메인 로직(재고 검증, 상태 전이)을 엔티티 메서드에 응집시킬 수 있음
- 단점:
    - 락을 오래 점유(트랜잭션 전체 구간)하므로 대기 시간이 길어질 수 있음
    - 락 대상 row 수가 많아질수록 대기 큐가 길어져 처리량이 떨어짐

```java
// 1) FOR UPDATE로 조회 (락 획득)
List<CartItem> items = cartItemRepository.findAllWithSkuForUpdate(...);

// 2) 자바 객체 필드로 계산
sku.reserveQuantity(quantity);  // this.reservedQuantity += quantity

// 3) 트랜잭션 커밋 시 JPA dirty checking이 UPDATE 발행
```

#### 2) DB 원자적 UPDATE (락 없이 조건부 쿼리)

- `SELECT`로 값을 미리 읽어서 판단하는 과정 자체를 생략
- "재고 확인 + 차감"을 하나의 UPDATE 문 안에서 원자적으로 처리
    1. `WHERE` 절의 조건이 만족되지 않으면 UPDATE 자체가 0 row에 적용됨
    2. 반환 된 영향 받은 row 수(`int`)가 0이면 재고 부족으로 판단
- 장점:
    - 락 대기 시간이 짧음(UPDATE 문 실행 시간만큼만 row 락 점유)
    - 처리량 측면에서 비관적 락보다 유리한 경우가 많음
- 단점:
    - 도메인 로직이 JPQL/SQL 문자열 안으로 들어감
    - `reserveQuantity()`가 더 이상 엔티티 메서드 호출로 표현되지 않고 벌크 쿼리 실행이 되어버림
    - 검증 실패 시 구체적인 사유(재고 얼마나 부족한지, SKU가 비활성인지 등)를 얻으려면 이 UPDATE 실행 전후로 별도 조회가 필요해질 수 있음

```java
@Modifying
@Query("""
    update ProductSku s
    set s.reservedQuantity = s.reservedQuantity + :quantity
    where s.id = :id
    and s.stockQuantity - s.reservedQuantity >= :quantity
    """)
int reserveStock(@Param("id") Long id, @Param("quantity") Integer quantity);
```

#### 3) 낙관적 락 VS 비관적 락

- 판매 결정되는 구조: 필드 두 개의 조합(`stockQuantity - reservedQuantity`)
- 비관적 락이 재고에 적합: blocking(대기) 떄문
- 예시: 사용자 A, B, C가 동시에 재고 10개짜리 SKU에 각각 5개 주문
1. 낙관적 락
    - B와 C 입장에서는 재고가 5개 남아있는데 주문이 실패
    - 단순 `@Version` 낙관적 락으로 감지하더라도(둘 중 하나만 바뀌어도 버전이 올라가므로) 
    근본적 해결이 아니라 재시도 로직이 추가로 필요해짐
        
        ⇒ 낙관적 락이 재고 도메인에 부적합
        
    
    ```text
    A, B, C 모두 동시에 재고 읽음 (version=5, 재고=10개)
    A → 커밋 성공 (version 5→6)
    B → 커밋 시도: WHERE version=5 → 이미 6이라 실패 → OptimisticLockException X
    C → 커밋 시도: WHERE version=5 → 이미 6이라 실패 → OptimisticLockException X
    
    결과: A만 성공, B와 C는 재고가 남아있어도 실패
    ```
    
2. 비관적 락
    - B와 C 입장에서는 재고가 5개 남아있는 걸 대기 시켜서 순서대로 처리
    - 재고 있으면 성공, 없으면 정상 실패가 보장
    
    ```text
    A → FOR UPDATE로 락 획득, 재고 확인(10개), 5개 예약 → 커밋 → 락 해제 (재고: 5개)
    B → 락 대기 중...    → 락 획득, 재고 확인(5개), 5개 예약 → 커밋 → 락 해제 (재고: 0개)
    C → 락 대기 중...           → 락 획득, 재고 확인(0개) → 재고 부족 예외
    
    결과: A, B 성공 / C 재고 부족으로 정상 실패
    ```
    

---

## 시도 1. 목록에 비관적 락 적용

- SKU, Product를 fetch join + `FOR UPDATE`까지 한 번에 처리
- `FOR UPDATE`는 장바구니 목록 조회 시점에 같이 걸기
    
    ⇒  validate에서도 락 걸린 엔티티를 쓰고, createOrderItem에서 재조회도 없어짐
    

### 조회

- `CartService`
    
    ```java
    public List<CartItem> getItemsByIdAndMember(List<Long> itemIds, Long memberId) {
        List<Long> distinctIds = itemIds.stream().distinct().toList();
        List<CartItem> cartItems = cartItemRepository.findAllWithSkuForUpdate(distinctIds, memberId);
        // ...
    }
    ```
    
- `CartItemRepositoryCustom`
    
    ```java
    List<CartItem> findAllWithSkuForUpdate(List<Long> itemIds, Long memberId);
    ```
    
- `CartItemRepositoryImpl`
    
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
            .orderBy(cartItem.id.asc()) // 초기 버전 -> 데드락 문제 예상
            .setLockMode(LockModeType.PESSIMISTIC_WRITE) // 비관적 락
            .fetch();
    }
    ```
    

### 결과

- 문제 A. 락의 획득 순서로 인한 데드락 위험
    - 여러 트랜잭션이 여러 행에 락을 걸 때, **트랜잭션마다 락을 거는 순서가 제각각이면** 교착 상태(데드락)가 발생할 수 있음
- 문제 B. FOR UPDATE + JOIN 구조 자체의 문제
    - 여러 테이블을 JOIN한 상태로 `FOR UPDATE`를 걸면, InnoDB는 조인에서 실제로 매칭되는 세 테이블의 행 모두에 락을 걸게 됨
    - 서로 다른 SKU를 서로 다른 사용자가 동시에 주문하려 할 때, 결국 같은 Product row에 락 경합이 생겨 동시성이 떨어짐

---

## 문제 A. 락의 획득 순서로 인한 데드락 위험

### 데드락(Deadlock)

- 둘 이상의 프로세스(또는 트랜잭션)가 서로 상대방이 점유한 자원의 해제를 기다리며 무한정 대기하는 상태
- 데드락 발생4가지 필요 조건
    1. **상호 배제 (Mutual Exclusion)**: 자원은 한 번에 하나의 트랜잭션만 점유 가능
    2. **점유 대기 (Hold and Wait)**: 자원을 점유한 채로 다른 자원을 추가로 기다림
    3. **비선점 (No Preemption)**: 다른 트랜잭션이 점유한 자원을 강제로 뺏을 수 없음
    4. **순환 대기 (Circular Wait)**: 트랜잭션들이 자원을 기다리는 관계가 원형으로 이어짐 (A는 B가 가진 자원을 기다리고, B는 A가 가진 자원을 기다리는 식)
- 데드락 회피 기법 대부분은 **순환 대기 조건을 깨는 것**을 목표로 함

#### RDBMS에서의 데드락

- MySQL InnoDB, PostgreSQL 등 주요 RDBMS는 데드락을 **자동으로 감지**하는 메커니즘(Deadlock Detection)을 갖고 있음
- 감지되면 관련 트랜잭션 중 하나(보통 처리 비용이 더 적은 쪽)를 DB 엔진이 강제로 롤백시키고, 해당 트랜잭션에 에러를 반환함
- MySQL의 경우 아래와 같은 에러가 발생
    
    ```bash
    Deadlock found when trying to get lock; try restarting transaction
    ```
    

---

### 주문 시 데드락 발생 원인

- 락을 요청하는 순서가 트랜잭션마다 다르다는 데 있음
- `cartItems`가 카트에 담긴 순서(생성 시점) 그대로 순회되면서 락을 거는데, 
이 순서는 사용자가 상품을 담은 순서에 따라 제각각임

#### 주문 시나리오

```
카트 아이템:
		회원 A의 카트: 의자A(SKU=101), 책상B(SKU=202)
		회원 B의 카트: 책상B(SKU=202), 의자A(SKU=101)   ← 같은 두 상품, 담은 순서만 다름

동시에 주문 실행:

트랜잭션 A                          트랜잭션 B
─────────────────────────────────────────────────────
SKU 101 락 획득 (성공)
                                    SKU 202 락 획득 (성공)
SKU 202 락 시도 → B가 보유 중, 대기
                                    SKU 101 락 시도 → A가 보유 중, 대기

A는 B가 SKU 101을 놓아주길 기다리고
B는 A가 SKU 202를 놓아주길 기다림
→ 서로 영원히 대기 (순환 대기, Deadlock)
```

1. `OrderService.createOrderItem()`이 카트 아이템 목록을 순회하며 각 SKU에 `FOR UPDATE` 락을 걸게 됨
2. 두 회원이 같은 두 SKU를 카트에 서로 다른 순서로 담고 동시에 주문하면, 
데드락 4가지 조건이 모두 성립할 수 있음
    - 상호 배제: SKU row의 `FOR UPDATE` 락은 한 트랜잭션만 보유
    - 점유 대기: SKU A를 락 건 채로 SKU B의 락을 기다림
    - 비선점: DB가 다른 트랜잭션의 락을 강제로 뺏지 않음
    - 순환 대기: A는 B가 보유한 자원을, B는 A가 보유한 자원을 서로 기다림
3. MySQL InnoDB는 이 순환 대기 상태를 감지하면 둘 중 한 트랜잭션을 강제로 골라 롤백
4. `Deadlock found when trying to get lock; try restarting transaction` 예외를 던짐
5. 둘 다 실패하는 게 아니라, 하나는 롤백되어 사용자에게 주문 실패로 나타남

---

### 데드락 회피

 

#### 데드락 회피 필요한 이유

- 여러 트랜잭션이 여러 행에 락을 걸 때, **트랜잭션마다 락을 거는 순서가 제각각이면** 교착 상태(데드락)가 발생할 수 있음
- 한 사용자가 카트에 서로 다른 SKU 여러 개를 담아 한 번에 주문할 수 있고, 
그 주문 처리 로직(`createOrderItem`)이 각 SKU에 대해 순차적으로 `FOR UPDATE` 락을 걸게 됨
- 데드락이 자주 발생할지는 트래픽 패턴(동시 접속자 수, 동일 SKU 동시 주문 빈도)에 달려 있음

#### 락 순서 정렬 (Lock Ordering)

- 데드락을 예방하는 방법
- **모든 트랜잭션이 자원에 대해 동일한 순서로 락을 요청하도록 강제**
- 순환 대기 조건을 깨는 접근
- 순환 구조 자체가 성립할 수 없음

#### orderBy가 문제를 막는 방식

1. 모든 트랜잭션이 동일한 기준(SKU ID 오름차순)으로 락을 순서대로 요청하게 강제
2. A든 B든 항상 “SKU 101 → SKU 102" 순서로만 락을 시도함
3. 순환 대기(circular wait)가 성립하지 않으므로 데드락 자체가 발생하지 않음
4. B는 대기하지만, A가 끝나면 정상적으로 진행됨

```
트랜잭션 A                          트랜잭션 B
─────────────────────────────────────────────────────
SKU 101 락 획득 (성공)
                                    SKU 101 락 시도 → A가 보유, 대기
SKU 102 락 획득 (성공, 이미 상판 보유 중이므로 순차 진행)
커밋, 락 전체 해제
                                    SKU 101 락 획득 (성공)
                                    SKU 102 락 획득 (성공)
                                    커밋
```

---

## 시도 2. orderBy 적용

- CartItem의 id 기준으로 정렬할 경우:
    - SKU 락 순서는 뒤섞일 수 있어 데드락 발생 가능
- SKU의 id 기준으로 정렬할 경우:
    - 락을 거는 대상(row) 그 자체의 식별자임
    - 모든 트랜잭션이 물리적으로 동일한 순서로 락을 요청하게 됨

### SKU의 id 기준으로 정렬

- `CartItemRepositoryImpl`
    
    락은 커밋 시점이 아니라 SELECT 실행 시점에 걸림
    
    1. `setLockMode(LockModeType.PESSIMISTIC_WRITE)`가 붙은 쿼리는 **호출**
    2. **즉시** DB에 락을 요청하는 SQL을 실행
    
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
            .orderBy(productSku.id.asc()) // sku id 기준 정렬
            .setLockMode(LockModeType.PESSIMISTIC_WRITE) // 비관적 락
            .fetch();  // ← 이 시점에 SELECT ... FOR UPDATE가 즉시 DB로 나감
    }
    ```
    
- 실행 쿼리
    
    ```sql
    select
        ci.id, ci.quantity, ci.cart_id, ci.sku_id, ...,
        ps.id, ps.price, ps.stock_quantity, ps.reserved_quantity, ...,
        p.id, p.name, p.status, ...
    from cart_items ci
    inner join product_skus ps on ci.sku_id = ps.id
    inner join products p on ps.product_id = p.id
    where ci.id in (?, ?, ?)
      and ci.cart_id in (select c.id from carts c where c.member_id = ?)
      and ps.status <> 'ARCHIVED'
      and p.status <> 'ARCHIVED'
    order by ps.id asc
    for update
    ```
    
---

### 최종 결과

- SKU의 id 기준으로 정렬하여 데드락은 회피하였음 ⇒ 문제 A 해결
- 문제 B. FOR UPDATE + JOIN 구조 자체의 문제 해결 안됨

---

## 정리

- 재고가 어긋난 원인은 Lost Update였다. 재고 필드를 애플리케이션으로 읽어와 계산한 뒤 도로 쓰는 구조라, 여러 트랜잭션이 같은 시점 값을 읽고 서로의 결과를 덮어썼다.
- REPEATABLE READ가 보장하는 건 "내가 읽은 값이 도중에 바뀌어 보이지 않는 것"까지다. 남이 그 값을 실제로 바꾸는 것까지는 막지 못하므로, 명시적 락이 있어야 했다.
- 재고는 "남으면 성공, 없으면 정상 실패"가 되어야 하는 도메인이다. 이 조건에는 대기 후 순차 처리가 되는 비관적 락이 이번 상황에서 더 단순하고 잘 맞았다. (낙관적 락도 재시도를 붙이면 못 쓸 건 아니지만, 재고 도메인에선 재시도 비용이 커진다.)
- fetch join 조회에 FOR UPDATE를 걸자 데드락 위험(문제 A)이 드러났고, SKU id 오름차순 정렬(orderBy(productSku.id.asc()))로 이 시나리오의 순환 대기를 끊는 쪽으로 방향을 잡았다.
   

문제는 여기서 멈추면 두 가지가 남는다는 점이다.   
첫째, 이 orderBy가 정말 락 획득 순서까지 정렬해 주느냐다. 정렬은 결과를 나열하는 순서일 뿐, DB가 행에 락을 거는 실제 순서와 같다는 보장은 아직 없다.
둘째, fetch join + FOR UPDATE가 product_skus만이 아니라 조인된 products까지 잠그는 락 범위 확산(문제 B)이다.

다음 편부터 이 두 가지를 EXPLAIN과 performance_schema.data_locks로 확인한다. 결론부터 말하자면 이번 편에서 "정렬로 피했다"고 본 문제 A의 판단은 틀렸다.