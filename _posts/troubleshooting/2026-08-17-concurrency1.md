---
title: "[주문 동시성 문제 해결기 1] 장바구니 조회 N+1 해결 - fetch join으로 재조회까지 제거하기"
date: 2026-08-17 10:00:00 +0900
categories: [Backend, Troubleshooting]
tags: [트러블슈팅, JPA, n+1, querydsl]
---

> 주문 동시성 문제 해결기 (1/7)
{: .prompt-tip }

## 현재 상태

### 시리즈에서 다루는 문제

재고 10개인 상품에 5개씩 동시 주문 두 건이 들어왔을 때, 둘 다 성공했지만 reservedQuantity는 5로만 남았다.
에러도 없고 로그상 둘 다 성공이었다.

### 이 글에서 다루는 것

해결책으로 비관적 락(`FOR UPDATE`)을 적용하려면 락을 거는 조회를 한 번으로 모아야 했다.
그런데 기존 구조는 LAZY 로딩으로 N+1이 발생하고, 락 획득을 위한 재조회까지 더해지는 상태였다.
락을 적용하기 전에 조회 구조부터 정리한다.

### 환경

Spring Boot 3.x / JPA(Hibernate) / QueryDSL / MySQL 8.0 (InnoDB, REPEATABLE READ)

---

## N+1 문제

1. 기존 N+1 문제
    - `ProductSku`, `Product`는 `LAZY`라 목록 조회 시점엔 안 불러옴
2. 비관적 락을 위한 재조회까지 더해지는 상황
- 실제 나가는 쿼리
    
    ```sql
    SELECT ci.* 
    FROM cart_items ci
    JOIN carts c ON ci.cart_id = c.id
    WHERE ci.id IN (?, ?, ?) AND c.member_id = ?
    ```
    
- 실제 쿼리 흐름
    
    ```text
    findAllByIdInAndCart_MemberId()
    └─ CartItem SELECT ... IN (...) 쿼리 1
    
    validateCartItems() 에서 cartItem.getProductSku() 접근
    └─ SKU lazy load → 아이템 수만큼 SELECT (N+1)
    
    sku.getProduct().getName() 접근
    └─ Product lazy load → 아이템 수만큼 SELECT (N+1)
    
    createOrderItem() 에서 findByIdForUpdate(skuId)
    └─ SELECT ... FOR UPDATE (락 획득용, 또 조회)
    ```
    

---

### 기존 코드

#### `CartItem` 엔티티

- `ProductSku`, `Product`는 `LAZY`

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

#### 조회

- `CartService`
    
    ```java
    public List<CartItem> getItemsByIdAndMember(List<Long> itemIds, Long memberId) {
        List<Long> distinctIds = itemIds.stream().distinct().toList();
        List<CartItem> cartItems = cartItemRepository.findAllByIdInAndCart_MemberId(distinctIds, memberId);
        // ...
    }
    ```
    
- `CartItemRepository`
    - 연관 엔티티는 함께 fetch 하지 않음
    - fetch join을 명시하지 않으면 LAZY 연관 엔티티는 조회되지 않음
    
    ```java
    List<CartItem> findAllByIdInAndCart_MemberId(List<Long> itemIds, Long memberId);
    ```
    

#### 연관 엔티티 호출

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
    
- `OrderService`
    
    ```java
    public void validateCartItems(List<CartItem> cartItems) {
        List<ItemError> errors = new ArrayList<>();
    
        for (CartItem cartItem : cartItems) {
            ProductSku sku = cartItem.getProductSku();
            String productName = sku.getProduct().getName() + "(" + sku.getSkuIdentifier() + ")";
            //...
        }
        // ...
    }
    ```
    

---

### EAGER 사용?

- 항상 sku의 이름 등이 필요하므로 Eager로 변경?
    
    ⇒ 실제로는 그렇지 않음
    
- 지금 CartItem을 쓰는 케이스들
    
    ```text
    getCart()           → QueryDSL fetch join으로 직접 DTO 조회 (엔티티 탐색 안 함)
    validateCartItems() → SKU, Product 필요
    createOrderItem()   → SKU, Product 필요
    updateCartItem()    → quantity만 변경, SKU 불필요
    deleteCartItem()    → id만 필요, SKU 불필요
    deleteCartItems()   → id만 필요, SKU 불필요
    ```
    

#### EAGER로 변경 시 문제

- EAGER로 바꾸면 항상 SKU, Product JOIN이 따라붙음
    
    ⇒ 필요 없는 데이터를 항상 끌고 옴
    
- ProductSku에 연관관계가 더 붙으면 통제가 어려워짐
- `@ManyToOne` EAGER는 연쇄적으로 문제가 커짐
    - `CartItem → ProductSku`만 EAGER로 바꿔도, 
    `ProductSku → Product`는 여전히 LAZY라 `product.getName()` 접근 시 또 별도 쿼리가 나감
    - 결국 EAGER 전환으로 N+1을 해결하려면 연쇄적으로 두 곳 다 바꿔야 하는 상황이 됨
        
        ⇒ LAZY + fetch join보다 좋지 않음
        

---

## N+1 해결

### LAZY 유지 + 필요한 쿼리에만 fetch join

- `findAllByIdInAndCart_MemberId`를 QueryDSL로 변경
- SKU, Product를 fetch join + `FOR UPDATE`까지 한 번에 처리

#### 조회

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
            .orderBy(cartItem.id.asc())
            .fetch();
    }
    ```

---

## 정리

- LAZY 유지 + 필요한 조회에만 fetch join으로 N+1 제거
- EAGER는 연관관계가 늘어날수록 통제가 어려워 선택하지 않음
- 조회를 한 번으로 모으면서 락 획득용 재조회도 제거
- 다음 글에서 이 조회에 `FOR UPDATE`를 적용