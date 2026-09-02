---
title: "[주문 동시성 문제 해결기 3-2] 락 범위 확산 문제 확인 (EXPLAIN, performance_schema)"
date: 2026-08-20 10:00:00 +0900
categories: [Backend, Troubleshooting]
tags: [동시성, 트러블슈팅, EXPLAIN, performance_schema, Lock, 데드락]
---

> 주문 동시성 문제 해결기 (4/7)
{: .prompt-tip }

## 현재 상태

### 시리즈에서 다루는 문제

2편은 SKU id 오름차순 정렬로 데드락(문제 A)을 피했다고 판단하고 넘어갔다. 그런데 그 판단에는 검증되지 않은 전제가 있었다. 정렬된 결과 집합의 순서가 실제 락 획득 순서와 같다는 보장이다. 여기에 더해, fetch join + FOR UPDATE가 product_skus뿐 아니라 조인된 products까지 잠근다는 문제 B도 아직 확인 전이었다.   
[2편 - 문제 A. 락의 획득 순서로 인한 데드락 위험]({% link _posts/troubleshooting/2026-08-18-concurrency2.md %})


### 이 글에서 다루는 것

앞 글에서 만든 데이터로 실제 락 상태를 들여다본다. EXPLAIN, EXPLAIN FORMAT=TREE, EXPLAIN FORMAT=JSON으로 이 쿼리에서 ORDER BY가 실행 계획의 어느 위치에 있는지 보고, performance_schema.data_locks로 트랜잭션을 열어둔 채 어떤 테이블의 어떤 행이 실제로 잠기는지 관찰한다. 이 과정에서 문제 B(락 범위 확산)를 확인하는 동시에, 2편에서 내린 문제 A의 판단이 실제로 유효했는지도 다시 확인한다.


### 환경

Spring Boot 3.x / JPA(Hibernate) / QueryDSL / MySQL 8.0 (InnoDB, REPEATABLE READ)

---

## 문제 B. FOR UPDATE + JOIN 구조 자체의 문제

- 여러 테이블을 JOIN한 상태로 `FOR UPDATE`를 걸면, InnoDB는 조인에서 실제로 매칭되는 세 테이블의 행 모두에 락을 걸게 됨
- 서로 다른 SKU를 서로 다른 사용자가 동시에 주문하려 할 때, 결국 같은 Product row에 락 경합이 생겨 동시성이 떨어짐
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
            .orderBy(productSku.id.asc()) // sku id 기준 정렬
            .setLockMode(LockModeType.PESSIMISTIC_WRITE) // 비관적 락
            .fetch();  // ← 이 시점에 SELECT ... FOR UPDATE가 즉시 DB로 나감
    }
    ```
    

### 엔티티

<details markdown="1">
<summary>엔티티 4개 전체 보기 (CartItem, Cart, ProductSku, Product)</summary>

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

- `Cart`

```java
    //...
    public class Cart extends BaseEntity {

        @Column(nullable = false, unique = true)
        private Long memberId;
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
        private Integer stockQuantity = 0;

        @Column(nullable = false)
        private Integer reservedQuantity = 0;
        //...
    }
```

- `Product`

```java
    //...
    public class Product extends BaseEntity {

        @ManyToOne(fetch = FetchType.LAZY)
        @JoinColumn(name = "category_id", nullable = false)
        private Category category;

        @Column(nullable = false, length = 255)
        private String name;
        //...
    }
```

</details>

---

### 발생하는 부작용

#### **1) `cart_items` 락**

- 해당 장바구니 아이템을 수정/삭제하려는 다른 요청
- ex) 사용자의 장바구니 수량 변경)이 블로킹됨 (이건 주문 처리 중이므로 허용할 수 있는 수준)

#### **2) `products` 락**

- 가장 큰 문제
- 동일한 상위 상품(`Product`)에 속한 다른 `ProductSku`(ex. 다른 사이즈나 색상)를 주문하려는 다른 사용자들이 전부 블로킹되어 동시성이 크게 떨어짐
- 인기 상품일수록 이 경합이 실질적인 처리량 저하로 이어질 가능성이 있음
- `Product`와 `ProductSku`는 1:N 관계
    1. 같은 상품의 SKU가 여러 개 있을 때 SKU A를 주문하는 트랜잭션과 
    SKU B를 주문하는 트랜잭션이 동일한 `Product` row를 각자 조인해서 락을 시도
    2. 두 트랜잭션이 서로 다른 SKU를 사더라도, 같은 상품(Product row)에 락을 걸려는 시점에 경합이 생김

#### **3) 데드락 위험**

- `orderBy(productSku.id.asc())`로 정렬을 했지만, 이는 `product_skus` 기준일 뿐
- 데이터베이스가 내부적으로 조인을 풀어나가는 과정에서 `cart_items` 인덱스와 `product_skus` 인덱스를 타는 순서가 꼬이면 여전히 데드락이 발생할 수 있음

---

## 확인 1. 실행 계획

- 실제로 문제가 발생하는 지 확인하기 위해 EXPLAIN 실행
- `findAllWithSkuForUpdate` 쿼리 EXPLAIN
    
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
    
- EXPLAIN 실행 결과:
    - `Using filesort` 문제 확인
- EXPLAIN FORMAT=TREE 실행 결과:
    - `carts` 테이블이 트리에 아예 안 보임
    - `ORDER BY`는 애초에 락 순서에 영향을 줄 수 없는 위치에 있음
- EXPLAIN FORMAT=JSON 실행 결과:
    - `ordering_operation`이 `nested_loop` 전체를 감싸는 구조

⇒ 이미 모든 락이 걸린 뒤에 실행되는 후 정렬

---

### EXPLAIN 실행 결과

- **`table`**: `c(carts) → ci(cart_items) → ps(product_skus) → p(products)` 순서로 접근
- `key`: 전부 인덱스를 통해 접근
- **`Extra`**: `Using filesort` -> “정렬 순서와 락 획득 순서 불일치" 가능성 존재

```text
+----+-------+--------+-----------------------+-----------------------+------+----------------------------------------------+
| id | table | type   | key                   | ref                   | rows | Extra                                        |
+----+-------+--------+-----------------------+-----------------------+------+----------------------------------------------+
|  1 | c     | const  | UKj43ag...            | const                 |    1 | Using index; Using temporary; Using filesort |
|  1 | ci    | ref    | uk_cart_item_cart_sku | const                 |    4 | Using index condition                        |
|  1 | ps    | eq_ref | PRIMARY               | fittura.ci.sku_id     |    1 | Using where                                  |
|  1 | p     | eq_ref | PRIMARY               | fittura.ps.product_id |    1 | Using where                                  |
+----+-------+--------+-----------------------+-----------------------+------+----------------------------------------------+
```

#### 조인 순서(join order)

- 순서: `carts → cart_items → product_skus → products`
    1. `carts`
        - `member_id`에 걸린 유니크 인덱스(`UKj43ag...`)로 정확히 1행(`type=const`)을 찾음
        
        ```sql
        WHERE ci.cart_id IN (SELECT c.id FROM carts c WHERE c.member_id = 1)
        ```
        
    2. `cart_items`
        - `cart_id` 값을 들고 `cart_items`의 `uk_cart_item_cart_sku` 인덱스로 넘어가서(`ref=const`, 예상 4행) SKU들을 찾음
    3. `product_skus` PK 기준 `eq_ref`(정확히 1행 매칭)
    4. `products` PK 기준 `eq_ref`(정확히 1행 매칭)
- 풀스캔(`ALL`)이 하나도 없고, 4단계 전부 인덱스를 타고 있음 ⇒ 인덱스 설계는 정상 작동

---

### 근거 1. Using filesort 문제

- 첫 번째 행(`c`)의 `Extra`에 `Using index; Using temporary; Using filesort`
- `ORDER BY`가 인덱스로 처리되지 않고 별도 정렬 단계를 거침
    
    ⇒ “정렬 순서와 락 획득 순서 불일치" 가능성 존재
    
- 정상인 상황
    - `ORDER BY ps.id ASC`가 인덱스에 의해 정렬 없이 이미 정렬된 순서로 데이터가 나옴)
        
        -> `filesort`는 안 나타나야 정상
        
- 현재 상황
    - `filesort`가 찍힘
    - MySQL이 조인을 다 실행해서 행들을 모은 다음, 별도의 정렬 단계를 거쳐야 `ps.id ASC` 순서를 만들 수 있다는 뜻

#### 실제 락 획득 순서 ≠ 정렬 순서

1. 조인이 실행되면서 각 행에 락이 걸리는 시점(runtime 접근 순서)
    - 조인 순서(`carts → cart_items → product_skus → products`
    - `uk_cart_item_cart_sku` 인덱스가 반환하는 물리적 순서
2. 정렬 순서
    - `ORDER BY ps.id ASC`: 조인 후 별도로 적용되는 정렬일 뿐

---

### EXPLAIN FORMAT=TREE 실행 결과

- `Using temporary` 원인을 좀 더 확정하기 위해 `EXPLAIN FORMAT=TREE` 실행
- 어느 단계에서 임시 테이블이 만들어지는지를 트리 구조로 더 명시적으로 확인
- 문법
    - 실행 순서
        1. 들여쓰기가 깊을수록(안쪽에 있을수록) 먼저 실행되는 단계
        2. 그 결과가 바깥쪽(위쪽) 노드로 흘러 들어 감
    - `Nested loop inner join`:
        - 자식이 두 개
        - 첫 번째 자식은 "기준(driving) 테이블"로 먼저 스캔됨
        - 그 결과 한 행 한 행에 대해 두 번째 자식이 반복 실행

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

#### 실행 순서

1. **`ci`(cart_items) 스캔**: 
    1. `uk_cart_item_cart_sku` 인덱스로 `cart_id='1'`을 찾음
    2. `ci.id IN (8,9,10,11)` 조건은 인덱스 조건 푸시다운(index condition)으로 걸러가며 4행을 뽑음
2. **`ps`(product_skus)**
    1. 그 4행 하나하나에 대해 **`ps`(product_skus)를 PK로 단건 조회**(`Single-row index lookup`)
    2.  `status <> 'ARCHIVED'` 필터를 적용
3. **`p`(products)**
    1. 남은 행 각각에 대해 **`p`(products)를 PK로 단건 조회**
    2. 같은 방식으로 필터를 적용
4. 이 모든 조인이 끝난 결과를 `Stream results`로 흘려보냄
5. 맨 마지막(최상위)에서 `Sort: ps.id`로 정렬

#### `carts`는 락 잠기지 않음

- `EXPLAIN`: `c`(carts)가 `type=const`인 별도 행으로 나왔음
- `EXPLAIN FORMAT=TREE`:
    - `carts` 테이블이 트리에 안 보임
    - `cart_id='1'`이라는 리터럴로 접혀 있음
- 근거
    1. `member_id`가 UNIQUE라서 정확히 1행만 나오는 조회
        
        → 옵티마이저가 이 값을 상수로 취급해 실행 계획 자체를 단순화한 것으로 보임
        
    2. MySQL 8.0 공식 문서: `t2`는 잠기지 않는다고 명시되어 있음
        
        ```sql
        SELECT * 
        FROM t1 
        WHERE c1 = (SELECT c1 FROM t2) 
        FOR UPDATE;
        ```
        
- 확인 사항
    - 서브쿼리(`carts` 조회) 자체엔 `FOR UPDATE`가 없어서 바깥의 `FOR UPDATE`는 `carts`엔 적용되지 않음
    - `cart_items, product_skus, products` 세 테이블에 락이 걸림
    - `carts`는 락 잠기지 않음
    
    ```sql
    WHERE ci.cart_id IN (SELECT c.id FROM carts c WHERE c.member_id = 1)
    ...
    FOR UPDATE;
    ```
    

---

### 근거 2. `ORDER BY`는 애초에 락 순서에 영향을 줄 수 없는 위치에 있음

- 락은 각 인덱스 레코드를 스캔하는(읽는) 시점에 걸림
- `Sort` 노드는 가장 바깥쪽(최상위), 즉 가장 마지막에 실행되는 단계
- `ORDER BY ps.id ASC`가 붙어 있든 없든
    1. cart_items/product_skus/products에 대한 락은 이미 그 이전 단계(`ci → ps → p` 순서의 nested loop 스캔)에서 전부 걸려버린 뒤
    2. `ORDER BY`는 최종 반환되는 "행의 나열 순서"만 바꿀 뿐, 락이 걸리는 시점이나 순서에는 애초에 관여할 수 없는 위치에서 실행
    
    ⇒  애초에 정렬이 실행되는 시점 자체가 모든 락 획득이 끝난 이후
    

---

### EXPLAIN FORMAT=JSON

- traditional 포맷 EXPLAIN의 Extra 컬럼만으론 `Using temporary/Using filesort`가
어느 처리 단계에 귀속된 것인지 특정 불가
    
    → `EXPLAIN FORMAT=JSON`으로 교차검증
    
- JSON 결과
    <details markdown="1">
    <summary>EXPLAIN FORMAT=JSON 전체 결과 보기</summary>

    ```json
    {
      "query_block": {
        "select_id": 1,
        "cost_info": {
          "query_cost": "5.60"
        },
        "ordering_operation": { // ORDER BY 처리를 나타내는 노드
          "using_temporary_table": true,
          "using_filesort": true,
          "cost_info": {
            "sort_cost": "2.25"
          },
          "nested_loop": [ // 조인 전체
            {
              "table": {
                "table_name": "c",
                "access_type": "const",
                "possible_keys": [
                  "PRIMARY",
                  "UKj43ag4hc9ceo08tnwsnol207h"
                ],
                "key": "UKj43ag4hc9ceo08tnwsnol207h",
                "used_key_parts": [
                  "member_id"
                ],
                "key_length": "8",
                "ref": [
                  "const"
                ],
                "rows_examined_per_scan": 1,
                "rows_produced_per_join": 1,
                "filtered": "100.00",
                "using_index": true,
                "cost_info": {
                  "read_cost": "0.00",
                  "eval_cost": "0.10",
                  "prefix_cost": "0.00",
                  "data_read_per_join": "40"
                },
                "used_columns": [
                  "id",
                  "member_id"
                ]
              }
            },
            {
              "table": {
                "table_name": "ci",
                "access_type": "ref",
                "possible_keys": [
                  "PRIMARY",
                  "uk_cart_item_cart_sku",
                  "FKia2f7yq196srm6ya3v96wafy6"
                ],
                "key": "uk_cart_item_cart_sku",
                "used_key_parts": [
                  "cart_id"
                ],
                "key_length": "8",
                "ref": [
                  "const"
                ],
                "rows_examined_per_scan": 4,
                "rows_produced_per_join": 4,
                "filtered": "100.00",
                "index_condition": "(`fittura`.`ci`.`id` in (8,9,10,11))",
                "cost_info": {
                  "read_cost": "0.50",
                  "eval_cost": "0.40",
                  "prefix_cost": "0.90",
                  "data_read_per_join": "192"
                },
                "used_columns": [
                  "quantity",
                  "cart_id",
                  "created_date",
                  "id",
                  "modified_date",
                  "sku_id"
                ]
              }
            },
            {
              "table": {
                "table_name": "ps",
                "access_type": "eq_ref",
                "possible_keys": [
                  "PRIMARY",
                  "FKgfjst7dvihycy15ceiruv9roo"
                ],
                "key": "PRIMARY",
                "used_key_parts": [
                  "id"
                ],
                "key_length": "8",
                "ref": [
                  "fittura.ci.sku_id"
                ],
                "rows_examined_per_scan": 1,
                "rows_produced_per_join": 3,
                "filtered": "75.00",
                "cost_info": {
                  "read_cost": "1.00",
                  "eval_cost": "0.30",
                  "prefix_cost": "2.30",
                  "data_read_per_join": "1K"
                },
                "used_columns": [
                  "reserved_quantity",
                  "stock_quantity",
                  "created_date",
                  "id",
                  "modified_date",
                  "price",
                  "product_id",
                  "color",
                  "material",
                  "status"
                ],
                "attached_condition": "(`fittura`.`ps`.`status` <> 'ARCHIVED')"
              }
            },
            {
              "table": {
                "table_name": "p",
                "access_type": "eq_ref",
                "possible_keys": [
                  "PRIMARY"
                ],
                "key": "PRIMARY",
                "used_key_parts": [
                  "id"
                ],
                "key_length": "8",
                "ref": [
                  "fittura.ps.product_id"
                ],
                "rows_examined_per_scan": 1,
                "rows_produced_per_join": 2,
                "filtered": "75.00",
                "cost_info": {
                  "read_cost": "0.75",
                  "eval_cost": "0.23",
                  "prefix_cost": "3.35",
                  "data_read_per_join": "2K"
                },
                "used_columns": [
                  "depth",
                  "height",
                  "weight",
                  "width",
                  "base_price",
                  "category_id",
                  "created_date",
                  "id",
                  "modified_date",
                  "description",
                  "name",
                  "product_type",
                  "status"
                ],
                "attached_condition": "(`fittura`.`p`.`status` <> 'ARCHIVED')"
              }
            }
          ]
        }
      }
    }
    ```

    </details>
    
- `ordering_operation`
    - `ORDER BY` 처리를 나타내는 노드
    - 노드가 `nested_loop`(조인 전체)를 감싸고 있다는 구조
        
        ⇒ 조인이 다 끝난 결과에 대해 정렬을 적용
        
- `nested_loop` 배열
    - 배열의 순서 = 조인 실행 순서(join order)
    - 각 원소가 테이블 하나씩
    
    ```text
    "ordering_operation": {
        "using_temporary_table": true,
        "using_filesort": true,
        "nested_loop": [ {c}, {ci}, {ps}, {p} ]
    }
    ```
    

### 근거 3. `ordering_operation`이 `nested_loop` 전체를 감싸는 구조

- 인덱스로 `ORDER BY`를 해결할 수 없는 조건임
    - EXPLAIN 결과에서 `c`(carts)는 `access_type: const`라 상수 테이블로 제외됨
    - 그다음 첫 번째 non-const 테이블은 `ci`(cart_items)
    - `ORDER BY`의 대상은 그다음 조인되는 `product_skus` 테이블의 컬럼
- 임시 테이블 + filesort 사용된 이유
    1. 조인 전체 결과를 일단 임시 테이블에 모은 뒤(`using_temporary_table`) 
    2. 그걸 다시 정렬(`using_filesort`)해야 함
- 테이블 조인이 전부 끝나야 그 결과가 `ordering_operation`으로 넘어갈 수 있음
    - 조인이 끝났다는 건 곧 `cart_items`/`product_skus`/`products`에 대한 `FOR UPDATE` 락이 이미 다 걸렸다는 뜻
    
    **=> `ORDER BY ps.id ASC`:** 
    
    - 이미 모든 락이 걸린 뒤에 실행되는 후처리 단계
    - 락 획득 순서에 관여할 수 없음

---

---

## 확인 2. performance_schema

- `performance_schema.data_locks`으로 실제 락 상태 확인
- 트랜잭션을 열어둔 상태에서(커밋 전) 다른 세션에서 아래 쿼리로 실제 어떤 테이블의 어떤 row에 락이 걸려 있는지 확인
- `performance_schema.data_locks`:
    - 쿼리를 실행하는 그 순간에 실제로 걸려 있는 락 상태를 즉석에서 보여주는 테이블
    - 실행 조건: 락을 쥔 채 커밋하지 않고 열어둔 트랜잭션이 다른 세션에 실제로 존재해야 함

```sql
SELECT * FROM performance_schema.data_locks;
```

### 확인 절차

- DBeaver 기준으로, SQL 에디터 탭 두 개(=서로 다른 세션) 필요
1. 탭 A (락을 거는 쪽)
    
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
    
    -- 여기서 COMMIT이나 ROLLBACK을 아직 실행하지 마세요. 트랜잭션을 열어둔 채로 둡니다.
    ```
    
2. 탭 B (락 상태를 관찰하는 쪽, 탭 A가 열려있는 동안 실행)
    - `OBJECT_SCHEMA = '프로젝트명'`로 필터링 권장
    - `performance_schema` 테이블에는 다른 세션이나 시스템 스키마의 락까지 다 나올 수 있어서, 필터 없이 보면 노이즈가 섞임
    
    ```sql
    SELECT * 
    FROM performance_schema.data_locks
    WHERE OBJECT_SCHEMA = 'fittura';
    ```
    
3. 탭 A
    - 확인이 끝나면 탭 A로 돌아가서 `ROLLBACK;`(또는 `COMMIT;`)을 실행해서 락을 풀어줌
    - 계속 열어두면 다른 테스트에 영향을 줌
    
    ```sql
    ROLLBACK;
    ```
    

#### 결과

```text
+--------------+-----------------------+-----------+---------------+-------------+------------------------+
| OBJECT_NAME  | INDEX_NAME            | LOCK_TYPE | LOCK_MODE     | LOCK_STATUS | LOCK_DATA              |
+--------------+-----------------------+-----------+---------------+-------------+------------------------+
| products     | NULL                  | TABLE     | IX            | GRANTED     | NULL                   |
| product_skus | NULL                  | TABLE     | IX            | GRANTED     | NULL                   |
| cart_items   | NULL                  | TABLE     | IX            | GRANTED     | NULL                   |
| cart_items   | uk_cart_item_cart_sku | RECORD    | X             | GRANTED     | supremum pseudo-record |
| cart_items   | uk_cart_item_cart_sku | RECORD    | X             | GRANTED     | 1, 2048, 8             |
| cart_items   | uk_cart_item_cart_sku | RECORD    | X             | GRANTED     | 1, 2049, 9             |
| cart_items   | uk_cart_item_cart_sku | RECORD    | X             | GRANTED     | 1, 2050, 10            |
| cart_items   | uk_cart_item_cart_sku | RECORD    | X             | GRANTED     | 1, 2051, 11            |
| cart_items   | PRIMARY               | RECORD    | X,REC_NOT_GAP | GRANTED     | 8                      |
| cart_items   | PRIMARY               | RECORD    | X,REC_NOT_GAP | GRANTED     | 9                      |
| cart_items   | PRIMARY               | RECORD    | X,REC_NOT_GAP | GRANTED     | 10                     |
| cart_items   | PRIMARY               | RECORD    | X,REC_NOT_GAP | GRANTED     | 11                     |
| products     | PRIMARY               | RECORD    | X,REC_NOT_GAP | GRANTED     | 512                    |
| product_skus | PRIMARY               | RECORD    | X,REC_NOT_GAP | GRANTED     | 2048                   |
| product_skus | PRIMARY               | RECORD    | X,REC_NOT_GAP | GRANTED     | 2049                   |
| product_skus | PRIMARY               | RECORD    | X,REC_NOT_GAP | GRANTED     | 2050                   |
| product_skus | PRIMARY               | RECORD    | X,REC_NOT_GAP | GRANTED     | 2051                   |
+--------------+-----------------------+-----------+---------------+-------------+------------------------+
```

---

### 근거 4. 테이블 레벨 `IX`(Intention Exclusive) 락

- `IX`(Intention Exclusive):
    - "이 테이블 안의 특정 행에 배타 락을 걸겠다"는 의도를 미리 테이블 단위로 표시해두는 락
    - 행 단위 락을 걸기 전에 항상 선행되는 InnoDB의 표준 메커니즘
    - 여러 트랜잭션이 동시에 같은 테이블에 각자 `IX`를 갖는 건 문제 없음
    - 진짜 충돌은 `LOCK TABLES`나 DDL 같은 진짜 테이블 전체 락과만 발생
- "이 세 테이블에서 행 단위 락이 실제로 발생했다"는 것을 뒷받침하는 근거 정도로 보면 됨

```text
+--------------+-----------+-----------+-------------+
| OBJECT_NAME  | LOCK_TYPE | LOCK_MODE | LOCK_STATUS |
+--------------+-----------+-----------+-------------+
| products     | TABLE     | IX        | GRANTED     |
| product_skus | TABLE     | IX        | GRANTED     |
| cart_items   | TABLE     | IX        | GRANTED     |
+--------------+-----------+-----------+-------------+
```

### 근거 5. `cart_items` 두 인덱스에 걸쳐 락이 걸림

- `LOCK_DATA`의 `1, 2048, 8`은 `(cart_id, sku_id, PK)` 순으로 읽으면 됨
- PRIMARY 인덱스
    - `REC_NOT_GAP`이 붙어 있음
    - “해당 레코드만 잠그고, 그 앞의 간격(gap)은 안 잠근다"는 뜻
    - PK로 정확히 매칭되는 단건 조회라 간격 잠금이 필요 없는 경우
- `supremum pseudo-record`
    - 인덱스에서 조건에 맞는 행들의 범위를 확정하기 위해, 그 범위의 끝(가장 큰 값 쪽 경계)까지 포함해서 잠그는 것
    - `cart_id=1`이라는 조건으로 `ref` 방식 스캔을 했기 때문에 발생한 것으로 보임
    
    ⇒ 같은 `cart_id=1`에 새로운 `cart_items` 행을 삽입하려는 다른 트랜잭션도 이 트랜잭션이 끝날 때까지 대기할 수 있음
    

```text
+-----------------------+-----------+---------------+-------------+------------------------+
| INDEX_NAME            | LOCK_TYPE | LOCK_MODE     | LOCK_STATUS | LOCK_DATA              |
+-----------------------+-----------+---------------+-------------+------------------------+
| uk_cart_item_cart_sku | RECORD    | X             | GRANTED     | supremum pseudo-record |
| uk_cart_item_cart_sku | RECORD    | X             | GRANTED     | 1, 2048, 8             |
| uk_cart_item_cart_sku | RECORD    | X             | GRANTED     | 1, 2049, 9             |
| uk_cart_item_cart_sku | RECORD    | X             | GRANTED     | 1, 2050, 10            |
| uk_cart_item_cart_sku | RECORD    | X             | GRANTED     | 1, 2051, 11            |
| PRIMARY               | RECORD    | X,REC_NOT_GAP | GRANTED     | 8                      |
| PRIMARY               | RECORD    | X,REC_NOT_GAP | GRANTED     | 9                      |
| PRIMARY               | RECORD    | X,REC_NOT_GAP | GRANTED     | 10                     |
| PRIMARY               | RECORD    | X,REC_NOT_GAP | GRANTED     | 11                     |
+-----------------------+-----------+---------------+-------------+------------------------+
```

### 근거 6. `products` 1개 행 락 잠김

- 4개의 서로 다른 SKU(2048, 2049, 2050, 2051)를 잠갔지만 Product 행은 단 하나(512)
- 같은 상위 상품에 속한 다른 SKU를 주문하려는 트랜잭션들이 결국 같은 Product row에서 경합
- 만약 다른 트랜잭션이 `id=512` 상품의 SKU를 (지금 잠긴 4개와 다른 SKU라도) 주문하려 한다면, 이 `products` 행 락 때문에 대기해야 함

```text
+------------+-----------+---------------+-------------+-----------+
| INDEX_NAME | LOCK_TYPE | LOCK_MODE     | LOCK_STATUS | LOCK_DATA |
+------------+-----------+---------------+-------------+-----------+
| PRIMARY    | RECORD    | X,REC_NOT_GAP | GRANTED     | 512       |
+------------+-----------+---------------+-------------+-----------+
```

### 근거 7. `product_skus` 각각 락 잠김

- `cart_items`가 `(cart_id, sku_id)` 인덱스로 스캔되면서 `sku_id` 오름차순으로 처리
    - 2048 < 2049 < 2050 < 2051
    - 실제 잠금 순서와 일치
    - 한 트랜잭션의 스냅샷 하나로 확인한 정황 증거
    - 여러 트랜잭션이 동시에 실행될 때도 항상 이 순서를 지킨다는 확정은 아님

```text
+------------+-----------+---------------+-------------+-----------+
| INDEX_NAME | LOCK_TYPE | LOCK_MODE     | LOCK_STATUS | LOCK_DATA |
+------------+-----------+---------------+-------------+-----------+
| PRIMARY    | RECORD    | X,REC_NOT_GAP | GRANTED     | 2048      |
| PRIMARY    | RECORD    | X,REC_NOT_GAP | GRANTED     | 2049      |
| PRIMARY    | RECORD    | X,REC_NOT_GAP | GRANTED     | 2050      |
| PRIMARY    | RECORD    | X,REC_NOT_GAP | GRANTED     | 2051      |
+------------+-----------+---------------+-------------+-----------+
```

---


## 정리

- EXPLAIN 결과, `ORDER BY ps.id`는 인덱스로 처리되지 못하고 `Using filesort`로 나타났다. `FORMAT=JSON`에서도 `ordering_operation`이 `nested_loop`(조인 전체)를 감싸는 구조였다. 정렬이 조인·락이 모두 끝난 뒤 적용되는 후처리라는 뜻이다.
- 즉 이 `ORDER BY`는 결과 나열 순서만 바꿀 뿐, 락이 걸리는 시점·순서에는 관여하지 못한다. 2편에서 "정렬로 데드락을 피했다"고 본 판단은 이 지점에서 성립하지 않는다.
- `performance_schema.data_locks`로 확인하니 `cart_items`, `product_skus`, `products` 세 테이블 모두에 행 락이 걸려 있었다. 4개의 서로 다른 SKU를 잠갔는데 `products`는 단 한 행(512)만 잠겼다. 같은 상품의 다른 SKU를 사려는 트랜잭션은 이 행에서 경합한다. 문제 B가 실재함을 확인했다.
- `carts`는 잠기지 않았다. `member_id`가 UNIQUE라 옵티마이저가 상수로 접었고, 서브쿼리에는 `FOR UPDATE`가 걸리지 않기 때문이다.

정렬이 락 순서를 지켜준다는 가정도, 락이 SKU에만 걸린다는 가정도 실측 앞에서 어긋났다. 문제 A는 아직 해결된 게 아니고, 문제 B는 확인됐다. 다음 글에서는 이 상태를 동시성 통합 테스트로 재현해, 정황이 아니라 실제 동시 실행에서도 같은 결론이 나오는지 본다.
