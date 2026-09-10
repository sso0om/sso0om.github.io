---
title: "[주문 동시성 문제 해결기 3-3] 락 범위 확산 문제 확인 (동시성 통합 테스트)"
date: 2026-08-24 10:00:00 +0900
categories: [Backend, Troubleshooting]
tags: [동시성, 트러블슈팅, 멀티스레드, CountDownLatch, ExecutorService, JUnit]
---

> 주문 동시성 문제 해결기 (5/7)
{: .prompt-tip }

## 현재 상태

### 시리즈에서 다루는 문제

앞 글에서 `EXPLAIN`과 `performance_schema.data_locks`로 두 가지를 확인했다. 정렬(`orderBy(productSku.id.asc())`)은 조인이 끝난 뒤 적용되는 후처리라 실제 락 순서에 관여하지 못한다는 것(문제 A), 그리고 `FOR UPDATE` + JOIN이 `products` 행까지 잠근다는 것(문제 B)이다. 다만 이건 트랜잭션 하나를 열어두고 관찰한 결과였다.  
[2편 - 문제 A. 락의 획득 순서로 인한 데드락 위험]({% link _posts/troubleshooting/2026-08-18-concurrency2.md %})  
[3-2편 - 문제 B. FOR UPDATE + JOIN 구조 자체의 문제]({% link _posts/troubleshooting/2026-08-20-concurrency4.md %})

### 이 글에서 다루는 것

동시 실행으로 재현한다. 여러 스레드가 같은 트랜잭션을 공유하지 않도록 테스트 환경을 따로 구성하고, 세 가지를 각각 검증한다. 재고보다 많은 동시 주문이 초과 판매로 이어지지 않는지, 서로 다른 순서로 담긴 카트를 동시 주문할 때 데드락이 나는지, 같은 상품의 다른 SKU 주문이 Product 행에서 서로 블로킹되는지다. 앞 글의 실측이 실제 동시 상황에서도 같은 결론으로 이어지는지 본다.

### 환경

Spring Boot 3.x / JPA(Hibernate) / QueryDSL / MySQL 8.0 (InnoDB, REPEATABLE READ) / JUnit 5
   

---

## 기존 통합 테스트 환경

### IntegrationTestBase
- 프로젝트의 통합 테스트는 `IntegrationTestBase`를 상속해서 사용
- `IntegrationTestBase`가 클래스 레벨에 `@Transactional`을 걸어두고 있음
- 이 클래스를 상속한 테스트는 별도로 신경 쓸 것 없이 테스트가 끝나면 자동으로 롤백됨
- 테스트마다 데이터를 직접 지울 필요가 없어 평소엔 편함

```java
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
@Transactional
public class IntegrationTestBase {

    @Autowired
    private JwtTokenProvider jwtTokenProvider;

    static final MySQLContainer<?> mysql =
        new MySQLContainer<>("mysql:8.0")
            .withDatabaseName("fittura_test")
            .withReuse(true);

    static {
        mysql.start();
    }

    @DynamicPropertySource
    static void overrideProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
        registry.add("spring.datasource.driver-class-name", () -> "com.mysql.cj.jdbc.Driver");
    }

    // ========== 헬퍼 메서드 ==========

    protected String userBearerToken(Long memberId) {
        return "Bearer " + jwtTokenProvider.generateAccessToken(memberId, Set.of("ROLE_USER"));
    }

    protected String adminBearerToken(Long memberId) {
        return "Bearer " + jwtTokenProvider.generateAccessToken(memberId, Set.of("ROLE_ADMIN"));
    }
}
```

```java
class OrderControllerTest extends IntegrationTestBase { ... }
```

### 기존 테스트 형식으로는 검증이 안 되는 이유

- 문제 원인: 이 클래스 레벨 `@Transactional`
  - 스프링의 `@Transactional` 테스트: 테스트 메서드 하나를 **하나의 트랜잭션**으로 감싸는데, 그 안에서 실행된 **스레드들이 같은 트랜잭션** 컨텍스트를 공유하게 됨
  - 동시성 검증: **여러 스레드가 각자 독립된 트랜잭션**(별도 커넥션)으로 DB에 실제 커밋까지 도달해야 락 경합이 재현됨
     
  => 트랜잭션이 하나로 묶여 있으면 애초에 경합이 일어날 수가 없음

### IntegrationTestBase 상속 받아야 하는 이유

- `IntegrationTestBase`에는 테스트 DB 연결 역할도 하고 있음
  - Testcontainers로 `MySQLContainer`를 띄우고, `@DynamicPropertySource`로 그 컨테이너의 접속 정보(URL, 계정)를 스프링 설정에 주입하는 부분
  - 이게 없으면 테스트가 붙을 DB 자체가 없음
- DB 연결과 트랜잭션 롤백, 두 역할을 아예 분리한 베이스 클래스를 새로 만드는 방법도 고려했지만 채택하지 않음
  - 이 시점에서는 트랜잭션 롤백이 필요 없는 테스트가 이거 하나뿐이라 과한 설계로 보임
  - 이 패턴이 다른 도메인(결제 등)에서도 반복되면 그때 분리하는 쪽이 맞다고 봄


---

## 확인 3. 동시성 통합 테스트

#### 동시성 테스트 방식
  1. (`OrderConcurrencyTest`)는 `IntegrationTestBase`를 상속
  2. `@Transactional(propagation = Propagation.NOT_SUPPORTED)`로 클래스 레벨 트랜잭션을 걷어냄
    - 클래스 레벨에서 트랜잭션 전파 방식을 다시 지정 => `@Transactional` 무력화
  3. 각 테스트에서 `@AfterEach`로 데이터를 직접 정리

```java
@SpringBootTest
@ActiveProfiles("test")
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public class OrderConcurrencyTest extends IntegrationTestBase {

    @Autowired private OrderFacade orderFacade;
    @Autowired private OrderAddressRepository addressRepository;
    //...

    private Category category;

    @BeforeEach
    void setUp() {
        category = CategoryFixture.rootActive();
        categoryRepository.save(category);
    }

    @AfterEach
    void tearDown() {
        addressRepository.deleteAll();
        orderItemRepository.deleteAll();
        orderRepository.deleteAll();
        cartItemRepository.deleteAll();
        cartRepository.deleteAll();
        //...
    }
}
```

### 스레드 동시 실행에 쓰는 도구들

- **`ExecutorService`**: 
  - 스레드를 직접 만들고 관리하는 대신, 스레드 풀에 작업을 던지고 실행을 맡기는 도구
  - `Executors.newFixedThreadPool(n)`으로 스레드 n개짜리 풀을 만들고, `executor.submit(...)`으로 각 작업을 등록함
- **`CountDownLatch`**: 
  - 카운터 하나를 두고, `countDown()`으로 카운터를 하나씩 줄이다가 0이 되면 `await()`로 대기 중이던 스레드들이 풀려나는 동기화 도구
- **`Collections.synchronizedList`**: 
  - 일반 `ArrayList`는 여러 스레드가 동시에 `add()`를 호출하면 데이터가 깨질 수 있음
  - 여러 스레드가 실패 원인(`Throwable`)을 동시에 이 리스트에 담아야 하므로 스레드 안전한 리스트로 감싸서 사용
- **`AtomicInteger`**: 
  - 일반 `int`에 여러 스레드가 동시에 `++`를 하면 연산이 겹쳐 개수가 틀리게 셀 수 있음
  - `AtomicInteger`는 증가 연산이 중간에 끊기지 않고 한 번에 처리되므로, 여러 스레드가 동시에 세어도 정확한 값이 나옴



### 스레드 동기화 방식

세 테스트 모두 같은 패턴을 사용. `CountDownLatch` 세 개로 스레드들의 시작 시점을 맞춤

- `readyLatch`: 워커 스레드 전원이 "준비 완료" 상태에 도달했는지 확인
- `startLatch`: 카운터 1개짜리라, `countDown()` 한 번으로 대기 중이던 스레드 전부가 동시에 풀림
- `doneLatch`: 워커 스레드 전원이 작업(성공/실패 무관)을 끝냈는지 확인

각 스레드는 `readyLatch.countDown()` → `startLatch.await()`로 대기 → 신호가 오면 동시에 실행 → `doneLatch.countDown()`으로 종료를 알리는 순서를 따름. 이렇게 해야 스레드들이 "거의 동시에" 실행돼 실제 동시 주문 상황을 재현할 수 있음


---

### 1) 동시 주문시 재고 초과 판매 여부

- 재고 초과 판매가 발생하지 않는지 검증
- 테스트 결과 성공 - 해당 시나리오는 기존 문제와 상관없었음. 성공하는 게 맞음

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

---

### 2) 데드락 회피 (락 순서 정렬 검증)

- 동시에 주문 실행 → `orderBy(productSku.id.asc())`가 실제로 락 순서를 정렬해주는지 확인
    - EXPLAIN/performance_schema 분석 결과, ORDER BY는 조인이 다 끝난 뒤(post-join)에 적용되는 정렬이라 실제 락 획득 시점에는 관여하지 못함
- `PessimisticLockingFailureException`
    - "비관적 락 획득 실패" 전체를 아우르는 상위 카테고리
    - Spring 공식 문서 자체가 "특정 서브클래스에 의존하지 말고 `PessimisticLockingFailureException` 자체를 처리하는 걸 권장한다"고 명시
    - `CannotAcquireLockException`
        - Spring Framework의 데이터 접근 예외(DataAccessException) 계층에 속한 예외
        - "SELECT FOR UPDATE 같은 락 획득 시도 자체가 실패했을 때" 던져지는 구체적인 하위 타입
        - MySQL의 `Deadlock found...` 에러가 이 예외로 변환되어 올라옴
    
    ```text
    DataAccessException
     └─ NonTransientDataAccessException
          └─ ConcurrencyFailureException
               └─ PessimisticLockingFailureException   ← 추상적인 상위 카테고리
                    ├─ CannotAcquireLockException        ← 구체적인 하위 타입
                    ├─ CannotSerializeTransactionException (deprecated)
                    └─ DeadlockLoserDataAccessException (deprecated)
    ```
    
- 테스트 결과 성공 - 이번 실행에서는 인덱스 구조 덕분에 우연히 안 걸린 것으로 추정
    - uk_cart_item_cart_sku (cart_id, sku_id) 인덱스 특성상 삽입 순서와 무관하게
    sku_id 오름차순으로 스캔되어 우연히 락 순서가 맞아떨어진 것으로 보임
    - 인덱스가 나중에 바뀌거나, 옵티마이저가 다른 실행 계획을 택하는 상황(데이터량 증가, 쿼리 조건 변경 등)이 오면 이 우연한 보호가 깨질 수 있음
    - 이 테스트가 통과한다고 데드락 위험이 해소된 건 아님

```java
@Test
@DisplayName("서로 다른 순서로 담긴 카트를 동시 주문해도 데드락이 발생하지 않음")
void concurrentOrdersWithReversedCartOrderDoNotDeadlock() throws InterruptedException {
    int stockEach = 5;

    // 서로 다른 상품에 속한 SKU 2개 준비 (같은 재고, 경합 여부는 검증 대상 아님)
    ProductSku skuA = getProductSku("의자A", stockEach);
    ProductSku skuB = getProductSku("의자B", stockEach);

    Long memberX = 91001L;
    Long memberY = 91002L;

    // 회원 X: 카트에 [skuA, skuB] 순서로 담음
    Cart cartX = Cart.create(memberX);
    cartRepository.save(cartX);
    CartItem itemXA = CartItem.create(cartX, skuA, 1);
    cartItemRepository.save(itemXA);
    CartItem itemXB = CartItem.create(cartX, skuB, 1);
    cartItemRepository.save(itemXB);
    List<Long> cartItemIdsX = List.of(itemXA.getId(), itemXB.getId());

    // 회원 Y: 카트에 [skuB, skuA] 순서로 담음 (X와 반대 순서 — 락 순서 충돌 유도 목적)
    Cart cartY = Cart.create(memberY);
    cartRepository.save(cartY);
    CartItem itemYB = CartItem.create(cartY, skuB, 1);
    cartItemRepository.save(itemYB);
    CartItem itemYA = CartItem.create(cartY, skuA, 1);
    cartItemRepository.save(itemYA);
    List<Long> cartItemIdsY = List.of(itemYB.getId(), itemYA.getId());

    // 반복되는 스레드 로직을 하나로 묶기 위한 (memberId, cartItemIds) 페어
    List<OrderTask> tasks = List.of(
        new OrderTask(memberX, cartItemIdsX),
        new OrderTask(memberY, cartItemIdsY)
    );
    AddressCreateReqDto address = addressDto();

    // ===== 동시 실행 =====
    ExecutorService executor = Executors.newFixedThreadPool(tasks.size());
    CountDownLatch readyLatch = new CountDownLatch(tasks.size());
    CountDownLatch startLatch = new CountDownLatch(1);
    CountDownLatch doneLatch = new CountDownLatch(tasks.size());
    List<Throwable> failures = Collections.synchronizedList(new ArrayList<>());

    for (OrderTask task : tasks) {
        executor.submit(() -> {
            try {
                readyLatch.countDown();   // "나 준비됐다"
                startLatch.await();       // 출발 신호까지 대기 → 동시 재개
                orderFacade.createOrder(task.memberId(), new OrderCreateReqDto(task.cartItemIds(), 0L, address));
            } catch (Throwable e) {
                failures.add(e);          // 실패 원인(타입) 보존 — 데드락 계열인지 이후 검사
            } finally {
                doneLatch.countDown();
            }
        });
    }

    readyLatch.await();       // 전원 대기 상태 도달 확인
    startLatch.countDown();   // 출발 — 동시 재개
    boolean finished = doneLatch.await(10, TimeUnit.SECONDS); // 락 대기가 무한정 걸릴 가능성 대비 타임아웃 필수
    executor.shutdown();

    // ===== 검증 =====
    // 타임아웃 없이 정상 종료됐는지 (진짜 락 대기가 걸려있으면 여기서 false)
    assertThat(finished).isTrue();

    // 실패가 있었다면 실제 예외 타입을 눈으로 확인 (디버깅/첫 실행 확인용)
    if (!failures.isEmpty()) {
        failures.forEach(e -> System.out.println(
            "failure type: " + e.getClass().getName()
                + (e.getCause() != null ? " / cause: " + e.getCause().getClass().getName() : "")
        ));
    }

    // 핵심 검증: 실패 원인 중 데드락 계열 예외가 있는지 여부
    boolean hasDeadlock = failures.stream().anyMatch(this::isDeadlockRelated);
    assertThat(hasDeadlock).isFalse();
}

private boolean isDeadlockRelated(Throwable e) {
    for (Throwable t = e; t != null; t = t.getCause()) {
        if (t instanceof PessimisticLockingFailureException) {
            return true;
        }
    }
    return false;
}
```

---

### 3) Product Row 락 경합 (락 범위 검증)

- 같은 Product에 속한 서로 다른 SKU를 동시 주문했을 때, Product row 레벨에서 락 경합이 발생해
직렬화되는지 확인
- 정확성 assertion이 아니라 타이밍 기반 관찰
- 스레드 스케줄링/CI 환경에 따라 flaky할 수 있음
- 상시 CI보다는 로컬에서 Product row 경합 존재를 증명하는 1회성 검증 용도
- 이 테스트는 "블로킹이 실제로 관찰되는가"라는 증상만 확인함
    - Product row가 구체적으로 잠긴다는 사실 자체는 이미 EXPLAIN + performance_schema.data_locks로 별도 확인된 것이며, 이 테스트가 그 원인을 직접 증명하지는 않음
    - 커넥션 풀 크기 등 다른 요인으로도 유사한 블로킹 증상이 나타날 수 있음에 유의
- 테스트 결과 실패 - 현재 구조(FOR UPDATE + JOIN)에서는 실패하는 게 정상
    - B의 실행 시간이 약 2077ms로 측정됨 → A의 락 점유 시간(holdMillis=2000ms)과 거의 일치
    - B가 A의 커밋 시점까지 대기했다는 뜻이며, Product row 경합 가설과 일치하는 결과
    - FOR UPDATE를 SKU 단위로 좁히는 리팩토링 후에는 이 assertion이 통과해야 함
    (통과 여부로 리팩토링의 실제 효과를 판단)

```java
@Test
@DisplayName("같은 상품의 다른 SKU를 동시 주문해도 Product 락 경합 없이 독립적으로 처리됨")
void concurrentOrdersOnSameProductDifferentSkuAreSerialized() throws Exception {
    long holdMillis = 2000L;

    Product chair = createActiveProduct("의자");
    ProductSku chairRed = createSku(chair, 5, "red");
    ProductSku chairBlue = createSku(chair, 5, "blue");

    Long memberX = 92001L;
    Long memberY = 92002L;

    Cart cartX = Cart.create(memberX);
    cartRepository.save(cartX);
    CartItem itemXRed = CartItem.create(cartX, chairRed, 1);
    cartItemRepository.save(itemXRed);

    Cart cartY = Cart.create(memberY);
    cartRepository.save(cartY);
    CartItem itemYBlue = CartItem.create(cartY, chairBlue, 1);
    cartItemRepository.save(itemYBlue);

    AddressCreateReqDto address = addressDto();
    TransactionTemplate transactionTemplate = new TransactionTemplate(transactionManager);
    ExecutorService executor = Executors.newFixedThreadPool(2);
    CountDownLatch lockAcquiredLatch = new CountDownLatch(1);

    // 스레드 A: chairRed 락을 잡고 holdMillis 동안 트랜잭션을 붙잡고 있음
    Future<?> threadA = executor.submit(() -> {
        transactionTemplate.execute(status -> {
            cartItemRepository.findAllWithSkuForUpdate(List.of(itemXRed.getId()), memberX);
            lockAcquiredLatch.countDown();
            try {
                Thread.sleep(holdMillis);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return null;
        }); // execute()가 리턴하는 시점 = 트랜잭션 커밋 완료 = 락 해제
        return null;
    });

    // B: A가 락을 잡은 직후, 같은 Product의 다른 SKU(blue) 주문 시도 + 실행 시간 측정
    Future<Long> threadB = executor.submit(() -> {
		    // 스레드 A가 lockAcquiredLatch.countDown() 전에 예외를 던지면, 스레드 B의 await()가 무한 대기할 수 있음
        if (!lockAcquiredLatch.await(2, TimeUnit.SECONDS)) {
            // A가 신호를 못 줬다면, 보통 A가 이미 예외로 끝났을 가능성이 높음
            if (threadA.isDone()) {
                threadA.get();
            }
            throw new AssertionError("스레드 A가 락을 획득하지 못했습니다 (A가 아직 실행 중)");
        }
        long start = System.nanoTime();
        orderFacade.createOrder(memberY, new OrderCreateReqDto(List.of(itemYBlue.getId()), 0L, address));
        return TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);
    });

    // 안 끝나는 상황만 막는 안전장치
    Long executionTimeB = threadB.get(3, TimeUnit.SECONDS);

    // 실제 검증: B의 실행 시간이 A의 락 점유 시간(holdMillis)보다 확실히 짧아야 함
    // 지금(FOR UPDATE + JOIN 구조)은 Product row 경합 때문에 holdMillis 근처까지 걸려 실패하는 게 정상
    // FOR UPDATE를 SKU 단위로 좁히는 리팩토링 후에는 이 assertion이 통과해야 함
    assertThat(executionTimeB).isLessThan(holdMillis - 500L);

    threadA.get(10, TimeUnit.SECONDS);
    executor.shutdown();
}
```


---

## 정리

- 동시성 검증은 클래스 레벨 `@Transactional`을 걷어내야 성립한다. 그대로 두면 스레드들이 같은 트랜잭션·커넥션을 공유해 락 경합이 재현되지 않기 때문이다. `CountDownLatch`로 스레드 시작 시점을 맞추고, 실제 DB 커밋까지 도달하도록 구성했다.
- 재고 초과 판매 테스트는 통과했다. 이 시나리오는 애초에 락과 무관하게 정상 동작하던 부분이라 예상된 결과다.
- 데드락 테스트도 통과했지만, 이건 문제 A(락의 획득 순서로 인한 데드락 위험)가 해결돼서가 아니다. `uk_cart_item_cart_sku (cart_id, sku_id)` 인덱스 특성상 스캔이 sku_id 오름차순으로 진행돼 락 순서가 우연히 맞아떨어진 것에 가깝다. 인덱스나 실행 계획이 바뀌면 이 우연한 보호는 깨질 수 있다. 통과가 곧 안전은 아니다.
- Product 행 경합 테스트는 예상대로 실패했다. 앞 스레드가 락을 2초 잡고 있는 동안 뒤 스레드가 거의 그만큼(약 2077ms) 대기했다. 같은 상품의 다른 SKU인데도 Product 행에서 직렬화된다는 뜻으로, 문제 B(FOR UPDATE + JOIN 구조 자체의 문제)가 실제 동시 실행에서도 재현됐다.

실제 동시 실행으로도 결론은 같았다. 문제 A(락의 획득 순서로 인한 데드락 위험)는 인덱스 덕에 우연히 가려져 있을 뿐 여전히 남아 있고, 문제 B(FOR UPDATE + JOIN 구조 자체의 문제)는 확정됐다. 다음 글에서 이 두 가지를 함께 해결한다.  