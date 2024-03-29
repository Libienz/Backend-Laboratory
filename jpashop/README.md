## 실전 스프링 부트와 JPA 활용2 - API 개발과 성능 최적화
<details>
<summary>Section 01: API 개발 기본 </summary>
<div markdown="1">

### 회원 등록 API V1
- 회원 등록 API를 다음과 같이 만들어보자
```java
    @PostMapping("api/v1/members") 
    public CreateMemberResponse saveMemberV1(@RequestBody @Valid Member member) {
        Long id = memberService.join(member);
        return new CreateMemberResponse(id);
    } 
```
- 엔티티를 RequestBody에 직접 매핑했다
  - 문제점 
    - 엔티티에 프레젠테이션 계층을 위한 로직이 추가된다. (프로젝션 얼마나 할지!)
    - 엔티티에 API 검증을 위한 로직이 들어간다.(@NotEmpty 등등)
    - 실무에서는 회원 엔티티를 위한 API가 다양하게 만들어지는데, 한 엔티티에 각각의 API를 위한 모든 요구사항을 담기는 어렵다.
    - 엔티티가 변경되면 API 스펙이 변한다.
  - 결론
    - API 요청 스펙에 맞추어 별도의 DTO를 파라미터로 받자!
    - 실무에서는 엔티티를 API 스펙에 노출하지 말자!

### 회원 등록 API V2
- 회원 등록 API를 다음과 같이 만들어보자
```java
    @PostMapping("/api/v2/members")
    public CreateMemberResponse saveMemberV2(@RequestBody @Valid CreateMemberRequest request) {
        Member member = new Member();
        member.setName(request.getName());
        Long id = memberService.join(member);
        return new CreateMemberResponse(id);
    } 
```
- CreateMemberRequest를 Member 엔티티 대신에 RequestBody와 매핑한다.
- 엔티티와 프레젠테이션 계층을 위한 로직을 분리할 수 있다.
- 에닡티와 API 스펙을 명확하게 분리할 수 잇다.
- 엔티티가 변해도 API 스펙이 변하지 않는다.

### 회원 수정 API
- 회원 수정 api를 다음과 같이 만들어보자 
```java
/**
 * 수정 API
 */
@PutMapping("/api/v2/members/{id}")
public UpdateMemberResponse updateMemberV2(
        @PathVariable("id") Long id,
        @RequestBody @Valid UpdateMemberRequest request) {
      memberService.update(id, request.getName());
      Member findMember = memberService.findOne(id);
      return new UpdateMemberResponse(findMember.getId(), findMember.getName());
      } 
```
- 회원 수정도 DTO를 요청 파라미터에 매핑한 것을 확인할 수 있다.
- 다음으로 변경 감지를 이용한 회원 수정 서비스 코드를 살펴보자 
```java
public class MemberService {
  private final MemberRepository memberRepository;
  /**
   * 회원 수정
   */
  @Transactional
  public void update(Long id, String name) {
    Member member = memberRepository.findOne(id);
    member.setName(name);
  }
} 
```
- 명시적인 영속화나 update 쿼리 없이 변경 감지 (Dirty checking)을 통하여 데이터를 수정하고 있다
- 수정은 변경감지를 이용하라고 했다!

### 회원 조회 API V1
- 회원조회 API V1을 살펴보자 
```java
 @GetMapping("/api/v1/members")
 public List<Member> membersV1() {
    return memberService.findMembers();
 }
```
- 위와 같이 naive하게 하면 다음의 문제점들이 있따.
  - 엔티티에 프레젠테이션 계층을 위한 로직이 추가된다.
  - 기본적으로 엔티티의 모든 값이 노출된다.
  - 응답 스펙을 맞추기 위한 로직이 추가된다(@JsonIgnore, 별도의 뷰 로직 등등)
  - 실무에서는 같은 엔티티에 대해 API가 용도에 따라 다양하게 만들어지는데 하나의 엔티티는 이 모든 것을 감당할 수 없다.
  - 컬렉션을 직접 반환하면 향후 API 스펙을 변경하기 어렵다.

### 회원 조회 V2: 응답 값으로 엔티티가 아닌 별도의 DTO 사용
- 회원 조회 V2를 살펴보자 
```java
/**
 * 조회 V2: 응답 값으로 엔티티가 아닌 별도의 DTO를 반환한다.
 */
@GetMapping("/api/v2/members")
public Result membersV2() {
      List<Member> findMembers = memberService.findMembers();
      //엔티티 -> DTO 변환
      List<MemberDto> collect = findMembers.stream()
          .map(m -> new MemberDto(m.getName()))
          .collect(Collectors.toList());
      return new Result(collect);
}
@Data
@AllArgsConstructor
static class Result<T> {
  private T data;
}
@Data
@AllArgsConstructor
static class MemberDto {
  private String name;
}
```
- 엔티티를 DTO로 변환해서 반환한다. 
- 엔티티가 변해도 API 스펙이 변경되지 않는다.
- 추가로 Result 클래스로 컬렉션을 감싸서 향후 필요한 필드를 추가할 수 있다.(count라던지 이런 것들 무조건 추가된다! 유지 보수를 위해서는 감싸라)


</div>
</details>

<details>
<summary>Section 02: API 개발 고급 - 준비 </summary>
<div markdown="1">

### 샘플 데이터 입력
```java
package jpabook.jpashop;

import jakarta.annotation.PostConstruct;
import jakarta.persistence.EntityManager;
import jpabook.jpashop.domain.*;
import jpabook.jpashop.domain.item.Book;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;


@Component
@RequiredArgsConstructor
public class InitDb {

    private final InitService initService;

    @PostConstruct
    public void init() {
        initService.dbInit1();
    }
    @Component
    @Transactional
    @RequiredArgsConstructor
    static class InitService {
        private final EntityManager em;
        public void dbInit1() {
            Member member = createMember("userA", "서울", "1", "1111");
            em.persist(member);

            Book book1 = createBook("JPA1 BOOK", 10000, 100);
            em.persist(book1);

            Book book2 = createBook("JPA2 BOOK", 20000, 100);
            em.persist(book2);

            OrderItem orderItem1 = OrderItem.createOrderItem(book1, 10000, 1);
            OrderItem orderItem2 = OrderItem.createOrderItem(book2, 20000, 2);

            Delivery delivery = createDelivery(member);
            Order order = Order.createOrder(member, delivery, orderItem1, orderItem2);
            em.persist(order);
        }

        public void dbInit2() {
            Member member = createMember("userB", "부산", "2", "2222");
            em.persist(member);

            Book book1 = createBook("SPRING1 BOOK", 20000, 200);
            em.persist(book1);

            Book book2 = createBook("SPRING2 BOOK", 40000, 300);
            em.persist(book2);

            OrderItem orderItem1 = OrderItem.createOrderItem(book1, 20000, 3);
            OrderItem orderItem2 = OrderItem.createOrderItem(book2, 40000, 4);

            Delivery delivery = createDelivery(member);
            Order order = Order.createOrder(member, delivery, orderItem1, orderItem2);
            em.persist(order);
        }
        private static Book createBook(String name, int price, int stockQuantity) {
            Book book1 = new Book();
            book1.setName(name);
            book1.setPrice(price);
            book1.setStockQuantity(stockQuantity);
            return book1;
        }
        private Member createMember(String name, String city, String street, String zipcode) {
            Member member = new Member();
            member.setName(name);
            member.setAddress(new Address(city, street, zipcode));
            return member;
        }
        private static Delivery createDelivery(Member member) {
            Delivery delivery = new Delivery();
            delivery.setAddress(member.getAddress());
            return delivery;
        }
    }


}

```

</div>
</details>


<details>
<summary>Section 03: API 개발 고급 - 지연 로딩과 조회 성능 최적화</summary>
<div markdown="1">

### 주문 조회 V1
- 간단한 주문 조회를 살펴보자
```java
 @GetMapping("/api/v1/simple-orders")
 public List<Order> ordersV1() {
 List<Order> all = orderRepository.findAllByString(new OrderSearch());
   for (Order order : all) {
     order.getMember().getName(); //Lazy 강제 초기화
     order.getDelivery().getAddress(); //Lazy 강제 초기환
   }
   return all;
 }
```
- 문제점들
  - 엔티티를 직접 노출하는 것은 좋지 않다.
  - order -> member 와 order -> address는 지연 로딩! 따라서 실제 엔티티 대신에 프록시가 존재
  - jackson 라이브러리는 기본적으로 이 프록시 객체를 json으로 어떻게 생성해야 하는 지 모름 -> 예외 발생
  - Hibernate5Module을 스프링 빈으로 등록하면 프록시 무제를 해결할 수는 있음
  - 양방향 연관관계에서 한곳을 @JsonIgnore처리하지 않으면 양쪽을 서로 호출하면서 무한 루프가 걸릴 수 있다

- 주의! 
  - 지연로딩을 피하기 위해 즉시 로딩으로 설정하면 안된다. 
  - 즉시 로딩으로 설정하면 성능 튜닝을 어렵게 한다
  - 연관관계가 필요 없는 경우에도 항상 조회하기에 성능 문제가 발생할 수 있는 것. 
  - 항상 지연 로딩을 기본으로 하고, 성능 최적화가 필요한 경우에는 페치 조인을 사용하시오

### 주문 조회 V2
- 엔티티를 DTO로 변환!

```java
/**
 * V2. 엔티티를 조회해서 DTO로 변환(fetch join 사용X)
 * - 단점: 지연로딩으로 쿼리 N번 호출
 */
@GetMapping("/api/v2/simple-orders")
public List<SimpleOrderDto> ordersV2() {
  List<Order> orders = orderRepository.findAll();
  List<SimpleOrderDto> result = orders.stream()
    .map(o -> new SimpleOrderDto(o))
    .collect(toList());
  return result;
}
```
- 엔티티를 DTO로 변환하는 일반적인 방법이다.
- 여기에서 List를 한 꺼풀 더 씌우면 더 좋은 설계 (해당 예제에선 생략)
- 문제점
  - 쿼리가 총 1 + N + N번 실행된다.
  - Order 조회 1번 (order 조회 결과 row 수가 N이 된다.)
  - order -> member 지연 로딩 조회 N번
  - order -> delivery 지연 로딩 조회 N번
  - 예) order의 결과가 4개면 최악의 경우 1 + 4 + 4번 실행된다 (최악의 경우)
    - 지연로딩은 영속성 컨텍스트에서 조회함으로 이미 조회된 경우 쿼리를 생략한다
    - 따라서 위에서 최악의 경우라고 명시한 것!

### 주문 조회 V3
- 페치 조인 최적화

```java
public List<Order> findAllWithMemberDelivery() {
      return em.createQuery(
      "select o from Order o" +
      " join fetch o.member m" +
      " join fetch o.delivery d", Order.class
      ).getResultList();
}
```
- 엔티티를 페치 조인을 사용해서 쿼리 1번에 조회
- 페치 조인으로 order -> member, order -> delivery는 이미 조회된 상태임으로 지연로딩 X

### 주문 조회 V4
- JPA에서 DTO로 바로 조회

```java

public List<OrderSimpleQueryDto> findOrderDtos() {
      return em.createQuery(
      "select new jpabook.jpashop.repository.OrderSimpleQueryDto(o.id, m.name, o.orderDate, o.orderStatus, d.address)" +
      " from Order o" +
      " join o.member m" +
      " join o.delivery d", OrderSimpleQueryDto.class
      ).getResultList();
}
```
- 일반적인 SQL을 사용할 때 처럼 원하는 값을 선택해서 조회
- new 명령어를 사용해서 JPQL의 결과를 DTO로 즉시 변환!
- SELECT절에서 원하는 데이터를 직접 선택하므로 DB -> 애플리케이션 네트웍 용량 최적화 (생각보다 미비)
- 리포지토리 재사용성 떨어짐, API 스펙에 맞춘 코드가 리포지토리에 들어가는 꼴
- 엔티티를 DTO로 변환하거나(V3) DTO로 바로 조회하는(V4) 두가지 방법은 각각 장단점이 있다.
  - V3: 리포지토리 코드 재사용성 굳. repository가 엔티티만을 조회할 수 있도록 일관성을 준다.
  - V4: 조회성능 미비하게 끌어올린다. (엔티티를 통째로 받아오지 않고 join한 테이블에서 내가 원하는 데이터만 냠냠 가능)

- 잠깐 잠깐 넘어가기 전에 join과 fetch join의 차이점 다시 짚고 가자!
- https://cobbybb.tistory.com/18

</div>
</details>

<details>
<summary>Section 04: API 개발 고급 - 컬렉션 조회 최적화 </summary>
<div markdown="1">

- 주문내역에서 추가로 주문한 상품 정보(OrderItem)를 추가로 조회하자
- Order 기준으로 컬렉션인 OrderItem과 Item이 필요하다.
- 앞의 예제에서는 toOne관계만 있었다. 이번에는 컬렉션인 일대다 관계를 조회하고 최적화 하는 방법을 알아보자

### 주문 조회 V1: 엔티티 직접 노출
```java
    @GetMapping("/api/v1/orders")
    public List<Order> ordersV1() {
        List<Order> all = orderRepository.findAllByString(new OrderSearch());
        // PROXY TOUCH
        for (Order order : all) {
            order.getMember().getName();
            order.getDelivery().getAddress();
            List<OrderItem> orderItems = order.getOrderItems();
            orderItems.stream().forEach(o -> o.getItem().getName());

        }
        return all;
    }
```
- 문제점 
  - 엔티티를 직접 노출 한다.
  - 양방향 연관관계면 무한 루프에 걸리지 않게 한곳에 @JsonIgnore를 추가해야 한다.
  - 프록시를 터치하여 초기화 해야 한다.

### 주문 조회 V2: 엔티티를 DTO로 변환
```java
    @GetMapping("/api/v2/orders")
    public List<OrderDto> ordersV2() {
        List<Order> orders = orderRepository.findAllByString(new OrderSearch());
        List<OrderDto> collect = orders.stream()
                .map(o -> new OrderDto(o))
                .collect((Collectors.toList()));

        return collect;
    }

    @Getter
    static class OrderDto {
        private Long orderId;
        private String name;
        private LocalDateTime orderDate;
        private Address address;
        private List<OrderItemDto> orderItems;

        public OrderDto(Order order) {
            orderId = order.getId();
            name = order.getMember().getName();
            orderDate = order.getOrderDate();
            address = order.getDelivery().getAddress();
            orderItems = order.getOrderItems().stream()
                    .map(orderItem -> new OrderItemDto(orderItem))
                    .collect(Collectors.toList());

        }
    }

    static class OrderItemDto {
        private String itemName;
        private int orderPrice;
        private int count;

        public OrderItemDto(OrderItem orderItem) {
            itemName = orderItem.getItem().getName();
            orderPrice = orderItem.getOrderPrice();
            count = orderItem.getCount();
        }
    }

```
- 문제점 
  - 지연 로딩으로 너무 많은 SQL이 실행된다.
  - SQL 실행 수
    - order 1번 (order 조회 수 N)
    - member, address N번
    - orderItem N번
    - item N번 (orderItem 조회 수 N)


### 주문 조회 V3: 엔티티를 DTO로 변환 - 페치 조인 최적화
```java
    public List<Order> findAllWithItem() {
        return em.createQuery(
                        "select o from Order o" +
                                " join fetch o.member m" +
                                " join fetch o.delivery d" +
                                " join fetch o.orderItems oi" +
                                " join fetch oi.item i", Order.class)
                .getResultList();
    }
```
- 페치 조인으로 SQL이 1번만 실행됨
- distinct를 사용한 이유는 1대다 조인이 있음으로 데이터베이스 row가 증가하기 때문
  - 그 결과 같은 order엔티티의 조회 수도 증가하게 된다. 
  - JPA의 distinct는 SQL에 distinct를 추가하고, 더해서 같은 엔티티가 조회되면, 애플리케이션에서 중복을 걸러준다.
  - 이 예에서 order가 컬렉션 페이 조인 때문에 중복 조회 되는 것을 막아준다.
- 단점 - 페이징 불가능!
  - 컬렉션 페치 조인을 사용하면 페이징이 불가능하다. 하이버네이트는 경고 로그를 남기면서 모든 데이터를 DB에서 읽어오고, 메모리에서 페이징 해버린다
  - 즉 페이징 sql이 작성되지 않는 것
    - Why! 
      - DB에서의 테이블은 distinct를 멕여도 모든 col이 똑같지 않다면 중복을 제거하지 않는다. 
      - 즉 페이징을 하고자 할 때 distinct가 먹지 않은 상태로 들어가기에 싱크가 맞지 않기에 DB에 요청할 수 없는 것 

  - 메모리에 퍼올린다음 애플리케이션레벨에서 페이징이 들어가는 것인데 매우매우 위험하다 (메모리 터질수도)
- 참고 : 컬렉션 페치 조인은 1개만 사용할 수 있다. 컬렉션 둘 이상에 페치 조인을 사용하면 안된다. 데이터가 부정합하게 조회될 수 있다. 

### 주문 조회 V3.1: 엔티티를 DTO로 변환 - 페이징과 한계 돌파

- 컬렉션을 페치 조인하면 페이징이 불가능하다.
  - 컬렉션을 페치 조인하면 일대다 조인이 발생함으로 데이터가 예측할 수 없이 증가
  - 일대다에서 일을 기준으로 페이징을 하는 것이 목적이지만 데이터는 다를 기준으로 생성된다.
  - Order를 기준으로 페이징 하고 싶은데, 다인 OrderItem을 조인하면 OrderItem이 기준이 되어 버린다.
- 이 경우 하이버네이트는 경고 로그를 남기고 모든 DB 데이터를 읽어서 메모리에서 페이징을 시도한다. 최악의 경우 장애로 이어질 수 있다

- 그러면 페이징 + 컬렉션 엔티티를 함께 조회하려면 어떻게 해야할까?
  - 먼저 ToOne 관계는 모두 페치조인한다. (ToOne관계는 row수를 증가시키지 않음으로 페이징 쿼리에 영향을 주지 않는다.)
  - 컬렉션은 지연로딩으로 조회한다.
  - 지연 로딩 성능 최적화를 위해 default_batch_fetch_size, @BatchSize를 적용한다. 
    - 이 옵션을 사용하면 컬렉션이나, 프록시 객체를 한꺼번에 설정한 Size만큼 IN 쿼리로 조회한다.

```java
    //OrderRepository
    public List<Order> findAllWithMemberDelivery(int offset, int limit) {
        return em.createQuery(
                        "select o from Order o" +
                                " join fetch o.member m" +
                                " join fetch o.delivery d", Order.class
                )
                .setFirstResult(offset)
                .setMaxResults(limit).
                getResultList();
    }
```
```java
    //Controller
    @GetMapping("/api/v3.1/orders")
    public List<OrderDto> orderV3_page(
            @RequestParam(value = "offset", defaultValue = "0") int offset,
            @RequestParam(value = "limit", defaultValue = "100") int limit) {
        List<Order> orders = orderRepository.findAllWithMemberDelivery(offset, limit);

        List<OrderDto> result = orders.stream()
                .map(o -> new OrderDto(o))
                .collect((Collectors.toList()));
        return result;
    }
```
- 장점 
  - 쿼리 호출 수가 1+N에서 1+1로 최적화된다.
  - 조인보다 DB 데이터 전송량이 최적화 된다.
    - Order와 OrderItem을 조인하면 Order가 OrderItem만큼 중복해서 조회된다. 하지만 이 방법은 각각 조회함으로 전송해야할 중복 데이터가 없다.
  - 페치 조인 방식과 비교해서 쿼리 호출 수가 약간 증가하지만 DB데이터 전송량이 감소한다.
  - 컬렉션 페치 조인은 페이징이 불가능하지만 이 방법은 페이징이 가능하다.
- 결론
  - ToOne관계는 페치조인해도 페이징에 영향을 주지 않는다. 따라서 ToOne관계는 페치조인으로 쿼리 수를 줄이고 해결하고, 나머지는 batch size로 최적화하자.
- 참고
  - default_batch_fetch_size의 크기는 100~1000사이를 선택하면 된다.
    - 이 전략은 SQL IN절을 사용하는데, 데이터베이스에 따라 IN절 파라미터를 1000으로 제한하기도 한다.
    - 1000으로 잡으면 한번에 1000개를 DB에서 애플리케이션에 불러옴으로 DB에 순간적 부하가 증가할 수 잇다.
    - 하지만 애플리케이션은 100이든 1000이든 결국 전체 데이터를 로딩해야 함으로 메모리 사용량이 같다.
    - 1000으로 설정하는 것이 성능상 가장 좋지만, 결국 DB든 애플리케이션이든 순간 부하를 어디까지 견딜 수 있는지로 결정하면 된다.

### 주문 조회 V4: JPA에서 DTO 직접 조회 
```java

    public List<OrderQueryDto> findOrderQueryDtos() {
        List<OrderQueryDto> result = findOrders();
        result.forEach(o -> {
            List<OrderItemQueryDto> orderItems = findOrderItems(o.getOrderId());
            o.setOrderItems(orderItems);
        });

        return result;
    }

    private List<OrderItemQueryDto> findOrderItems(Long orderId) {
        return em.createQuery(
                        "select new jpabook.jpashop.repository.order.query.OrderItemQueryDto(oi.order.id, i.name, oi.orderPrice, oi.count)" +
                                " from OrderItem oi" +
                                " join oi.item i" +
                                " where oi.order.id = :orderId", OrderItemQueryDto.class
                )
                .setParameter("orderId", orderId)
                .getResultList();
    }
```
- Query: 루트 1번, 컬렉션 N번 실행
- ToOne 관계들을 먼저 조회하고, ToMany 관계는 각각 별도로 처리한다.
  - 이런 방식을 선택한 이유는 다음과 같다.
  - ToOne관계는 조인해도 데이터 row수가 증가하지 않는다.
  - ToMany 관계는 조인하면 row수가 증가한다.
- row수가 증가하지 않는 ToOne관계는 조인으로 최적화 하기 쉬움으로 한번에 조회하고 ToMany관계는 최적화 하기 어려움으로 findorderItems같은 별도의 메서드로 조회한 것

### 주문 조회 V5: JPA에서 DTO 직접 조회 - 컬렉션 조회 최적화
- OrderApiController
```java

    @GetMapping("/api/v5/orders")
    public List<OrderQueryDto> orderV5() {
        return orderQueryRepository.findAllByDto_optimization();
    }
```
- OrderQueryRepository

```java
    public List<OrderQueryDto> findAllByDto_optimization() {
        List<OrderQueryDto> result = findOrders();

        List<Long> orderIds = toOrderIds(result);
        Map<Long, List<OrderItemQueryDto>> orderItemMap = findOrderItemMap(orderIds);

        result.forEach(o -> o.setOrderItems(orderItemMap.get(o.getOrderId())));

        return result;
    }

    private Map<Long, List<OrderItemQueryDto>> findOrderItemMap(List<Long> orderIds) {
        List<OrderItemQueryDto> orderItems = em.createQuery(
                        "select new jpabook.jpashop.repository.order.query.OrderItemQueryDto(oi.order.id, i.name, oi.orderPrice, oi.count)" +
                                " from OrderItem oi" +
                                " join oi.item i" +
                                " where oi.order.id in :orderIds", OrderItemQueryDto.class
                )
                .setParameter("orderIds", orderIds)
                .getResultList();

        Map<Long, List<OrderItemQueryDto>> orderItemMap = orderItems.stream()
                .collect(Collectors.groupingBy(OrderItemQueryDto::getOrderId));
        return orderItemMap;
    }

    private static List<Long> toOrderIds(List<OrderQueryDto> result) {
        List<Long> orderIds = result.stream()
                .map(o -> o.getOrderId())
                .collect(Collectors.toList());
        return orderIds;
    }
```
- Query: 루트 1번 컬렉션 1번
- ToOne 관계들을 먼저 조회하고, 여기서 얻은 식별자 orderId로 ToMany 관계인 orderItem을 명시적 IN Query로 한번에 조회
- Map을 이용해서 result에 추가한다. 

### 주문 조회 V6: JPA에서 DTO로 직접 조회 플랫 데이터 최적화
- OrderApiController

```java

    @GetMapping("/api/v6/orders")
    public List<OrderQueryDto> orderV6() {
        List<OrderFlatDto> flats = orderQueryRepository.findAllByDto_flat();

        return flats.stream()
                .collect(groupingBy(o -> new OrderQueryDto(o.getOrderId(),
                                o.getName(), o.getOrderDate(), o.getOrderStatus(), o.getAddress()),
                        mapping(o -> new OrderItemQueryDto(o.getOrderId(),
                                o.getItemName(), o.getOrderPrice(), o.getCount()), toList())
                )).entrySet().stream()
                .map(e -> new OrderQueryDto(e.getKey().getOrderId(),
                        e.getKey().getName(), e.getKey().getOrderDate(), e.getKey().getOrderStatus(),
                        e.getKey().getAddress(), e.getValue()))
                .collect(toList());
    }
```

- OrderQueryRepositroy

```java

    public List<OrderFlatDto> findAllByDto_flat() {
        return em.createQuery(
                "select new jpabook.jpashop.repository.order.query.OrderFlatDto(o.id, m.name, o.orderDate, o.status, d.address, oi.orderPrice, oi.count)" +
                        " from Order o" +
                        " join o.member m" +
                        " join o.delivery d" +
                        " join o.orderItems oi" +
                        " join oi.item i", OrderFlatDto.class)
                .getResultList();

    }
```
- Query가 단 1번 나간다.
- 테이블을 join해서 flat하게 만든 테이블에서 row를 돌며 애플리케이션 단에서 커스텀 하는 것이 V6
- 애플리케이션에서 작업이 많고 Order 입장의 페이징이 불가능하다는 단점이 있으며 중복 데이터 역시 별도의 처리가 필요하다.

### API 개발 고급 정리
### 엔티티 조회
  - 엔티티를 조회해서 그대로 반환: V1
  - 엔티티 조회 후 DTO로 변환: V2
  - 페치 조인으로 쿼리 수 최적화: V3
  - 컬렉션 페이징과 한계 돌파: V3.1
    - 컬렉션은 페치 조인 시 페이징이 불가능
    - ToOne 관계는 페치 조인으로 쿼리 수 최적화
    - 컬렉션은 페치 조인 대신에 지연 로딩을 유지하고, @BatchSize로 최적화
### DTO로 직접 조회
  - JPA에서 DTO를 직접 조회: V4
  - 컬렉션 조회 최적화 - 일대다 관계인 컬렉션은 IN절을 활용해서 메모리에서 미리 조회해서 최적화: V5
  - 플랫 데이터 최적화 - JOIN 결과를 그대로 조회 후 애플리케이션에서 원하는 모양으로 발라낸다: V6

### 권장 순서
- 엔티티 조회 방식으로 우선 접근
  - 성능 최적화가 필요한 부분을 페치 조인으로 쿼리 수를 최적화
  - 컬렉션 최적화 
    - 페이징이 필요한지 살펴보고 필요한 경우 @BatchSize로 최적화 필요 없다면 fetchjoin을 사용하자 (distinct)
- 엔티티 조회 방식으로 해결이 안되면 직접적 SQL에 가까운 DTO 조회 방식을 사용하자 (Select 절을 최적화 하여 네트웤 용량을 최적화 할 수 있다)
- DTO 조회 방식으로도 해결이 안되면 NativeSQL or 스프링 JdbcTemplate 사용
  - 근데 사실 이정도 까지 와서 안되면 Redis 등의 db를 사용해야 하지 않는지 고민해볼것 ..
- 참고
- 엔티티 조회 방식은 페치 조인이나, hibernate.default_batch_fetch_size , @BatchSize 같이 코드를 거의 수정하지 않고, 옵션만 약간 변경해서, 다양한 성능 최적화를 시도할 수 있다.
- 반면에 DTO를 직접 조회하는 방식은 성능을 최적화하거나 성능 최적화 방식을 변경할 때 많은 코드를 변경해야 한다.
- 개발자는 성능 최적화와 코드 복잡도 사이에서 줄타기를 해야 한다. 항상 그런 것은 아니지만, 보통 성능 최적화는 단순한 코드를 복잡한 코드로 몰고간다.
- 엔티티 조회 방식은 JPA가 많은 부분을 최적화 해주기 때문에 단순한 코드를 유지하면서, 성능을 최적화 할 수 있다!
- 엔티티 조회 방식을 즐겨쓰는 것이 괜찮을 듯!

### DTO 조회 방식의 선택지 
- DTO로 조회하는 방법도 각각 장단이 있다. V4, V5, V6에서 단순하게 쿼리가 1번 실행된다고 V6이 항상 좋은 방법인 것은 아니다. 각각의 장단점을 살펴보자
- V4
  - 코드가 단순하다. 특정 주문 한건만 조회해야 되는 경우라면 이 방식을 사용해도 성능이 잘 나온다 (추가 쿼리가 줄어들기 때문)
  - 예를 들어서 Order데이터가 1건이면 OrderItem을 찾기 위한 쿼리도 1번만 실행하면 된다. 
- V5
  - 코드가 V4에 비해 복잡하다. 
  - 하지만 여러 주문을 한꺼번에 조회하는 경우에는 V4대신에 이것을 최적화한 V5방식을 사용하면 꽤나 좋은 성능 최적화를 이루어 낼 수 있다.
  - 예를 들어서 조회한 Order데이터가 1000건인다, V4방식을 그대로 사용하면 쿼리가 총 1 + 1000번 실행된다. 
  - 여기서 1은 Order를 조회한 쿼리고 1000은 조회된 Order의 row 수다. 
  - V5방식으로 최적화 하면 쿼리가 총 1+1번만 실행된다 (명시적 IN절 사용)
  - 상황에 따라 다르겠지만 운영환경에서 100배 이상의 성능 차이가 날 수 있다!
- V6
  - V6는 완전히 다른 접근방식이다. 
  - 쿼리 한번으로 최적화 되기에 상당히 좋아보이지만 Order를 기준으로 페이징이 불가능하다. (Join을 하기 때문)
  - 실무에서는 이정도 데이터면 수백이나, 수천건 단위로 페이징 처리가 꼭 필요함으로 이 경우 선택하기 어려운 방법이다.
  - 그리고 데이터가 많으면 중복 전송이 증가해서 V5와 비교해서 성능 차이도 미비한 편이다

</div>
</details>


<details>
<summary>Section 05: API 개발 고급 - 실무 필수 최적화 OSIV</summary>
<div markdown="1">

### OSIV와 성능 최적화
- Open Session In View: 하이버네이트에서의 엔티티매니저 역할을 하는 것을 Session이라고 한다. (View에서도 커넥션이 열려있는 상태를 암시하는 것을 알 수 있음)
- Open EntityManager In View: JPA
- ![img.png](img.png)
- 어플리케이션을 실행시키면 이상한 WARN로그를 발견할 수 있다.
  - spring.jpa.open-in-view의 default가 true임으로 api응답이 끝날 때 까지 DB커넥션을 물고 있을 수 있다라는 로그를 말하는 것
  - 이 기본값을 뿌리면서 WARN 로그를 남기는 것은 이유가 있다.
  - OSIV 전략은 트랜잭션 시작처럼 최초 데이터베이스 커넥션 시작지점 부터 API 응답이 끝날 때 까지 영속성 컨텍스트와 데이터베이스 커넥션을 유지한다.
  - 그래서 지금까지 View Template이나 API 컨트롤러에서 지연 로딩이 가능했던 것이다.
  - 지연 로딩은 영속성 컨텍스트가 살아있어야 가능하고, 영속성 컨텍스트는 기본적으로 데이터베이스 커넥션을 유지한다.
  - 지연 로딩을 가능하게 하고 있었다니! 이것 자체가 큰 장점이라고 볼 수 있겠다

### 그렇다면 OSIV는 항상 켜놓는 것이 좋을까?
- OSIV 전략은 너무 오랜시간동안 데이터베이스 커넥션 리소스를 사용하기 때문에 실시간 트래픽이 중요한 애플리케이션에서는 커넥션이 모자를 수 있다.
- 이것은 결국 장애로 이어진다..
- 예를 들어서 컨트롤러에서 외부 API를 호출하면 외부 API 대기 시간 만큼 커넥션 리소스를 반환하지 못하고 유지해야 하는 것이다!
- ![img_1.png](img_1.png)
- OSIV를 끄면 트랜잭션을 종료할 때 영속성 컨텍스트를 닫고, 데이터베이스 커넥션도 반환한다. 따라서 커넥션 리소스를 낭비하지 않는다.
- OSIV를 끄면 모든 지연로딩을 트랜잭션 안에서 처리해야 한다. 
- 따라서 지금까지 작성한 많은 지연 로딩 코드를 트랜잭션 안으로 말아 넣어야 한다는 단점이 있다.
- 그리고 view template에서 지연로딩이 동작하지 않는다. 
- 결론적으로 트랜잭션이 끝나기 전에 지연 로딩을 강제로 호출해 두어야 한다.

### 커맨드와 쿼리 분리
- 실무에서 OSIV를 끈 상태로 복잡성을 관리하는 좋은 방법이 있다. 바로 Command와 Query를 분리하는 것이다.
- 보통 비즈니스 로직은 특정 엔티티 몇 개를 등록하거나 수정하는 것이므로 성능이 크게 문제가 되지 않는다.
- 그런데 복잡한 화면을 출력하기 위한 쿼리는 화면에 맞추어 성능을 최적화 하는 것이 중요하다.
- 하지만 그 복잡성에 비해 핵심 비즈니스에 큰 영향을 주는 것은 아니다.
- 그래서 크고 복잡한 애플리케이션을 개발한다면, 이 둘의 관심사를 명확하게 분리하는 선택은 유지보수 관점에서 충분히 의미 있다.
- 단순하게 설명해서 다음처럼 분리하는 것이다.
  - OrderService: 핵심 비즈니스 로직
  - OrderQueryService: 화면이나 API에 맞춘 서비스 (주로 읽기 전용 트랜잭션 사용)

```java

    @GetMapping("/api/v3/orders")
    public List<OrderDto> orderV3() {
        List<Order> orders = orderRepository.findAllWithItem();

        List<OrderDto> result = orders.stream()
                .map(o -> new OrderDto(o))
                .collect((toList()));
        return result;
    }

```
- 해당 코드를 보면 Repository에서 orders를 가져와서 stream과 lambda표현식을 이용해서 api 스펙에 맞추어 리스트를 깎고 있다.
- 만약 OSIV가 false 이면 위의 코드는 오류를 뿜을 것임 (지연로딩을 트랜잭션 밖에서 하고 있기 때문에)
- 위의 코드에서 api 스펙을 위해서 dto를 깎는 부분을 별도의 서비스로 두어 분리하라는 것이다
- 분리된 Service는 트랜잭션을 따로 ReadOnly로 열면 OSIV문제에 구애받지 않게 되고 관심사의 분리도 명확하게 되는 것이다.


</div>
</details>


<details>
<summary>Section 06: 스프링 데이터 JPA와 QueryDSL 소개 </summary>
<div markdown="1">

### 스프링 데이터 JPA 소개

- 스프링 데이터 JPA는 JPA를 사용할 때 지루하게 반복하는 코드를 (Boilerplate code) 자동화 해준다. 

- 원래의 MemberRepository
```java
package jpabook.jpashop.repository;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import jpabook.jpashop.domain.Member;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public class MemberRepository {

    @PersistenceContext
    private EntityManager em;

    public void save(Member member) {
        em.persist(member); //영속성 컨텍스트에 member 삽입, transaction이 커밋되는 순간에 DB에 반영된다.(insert query가 날라가는 시점)
    }

    public Member findOne(Long id) {
        return em.find(Member.class, id);
    }

    public List<Member> findAll() {
        return em.createQuery("select m from Member m", Member.class) //jpql sql과 다른점이있다. sql은 테이블 대상으로 쿼리, jpql은 entitiy객체를 대상으로 쿼리
                .getResultList();
    }

    public List<Member> findByName(String name) {
        return em.createQuery("select m from Member m where m.name = :name", Member.class)
                .setParameter("name", name)
                .getResultList();
    }

}

```
- Spring Data JPA Member Repository

```java
package jpabook.jpashop.repository;
import jpabook.jpashop.domain.Member;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;
public interface MemberRepository extends JpaRepository<Member, Long> {
    List<Member> findByName(String name);
}
```

- 스프링 데이터 JPA는 JpaRepository라는 인터페이스를 제공하는데, 명시적으로 구현하지 않아도 기본적인 CRUD 기능이 모두 제공된다!
- 개발자는 인터페이스만 만들면 구현체는 스프링 데이터 JPA가 어플리케이션 실행 시점에 주입해주는 것이다.
  - 추가적으로 위의 findByName(String name)을 보자 
  - 스프링 데이터 jpa에서 멤버를 이름으로 조회하여 반환하는 것은 기본적으로 제공하지 않는다. (findById는 제공하지만..)
  - 그럼에도 불구하고 signature를 정확히 기입하여 인터페이스에 올려놓으면 스프링 데이터 jpa는 이에 맞는 jpql을 생성해주는 구현체를 만들어준다!
- 스프링 데이터 JPA는 스프링과 JPA를 활용해서 애플리케이션을 만들 때 정말 편리한 기능을 많이 제공한다. 
- 단순히 편리함을 넘어서 때로는 마법을 부리는 것 같을 정도로 놀라운 개발 생산성의 세계로 우리를 이끌어 준다.
- 하지만 스프링 데이터 JPA는 JPA를 사용해서 이런 기능을 제공할 뿐이니 결국 JPA자체를 잘 이해하는 것이 중요하다고 말할 수 있다.

### QueryDSL
- 실무에서는 조건에 따라서 실행되는 쿼리가 달라지는 동적 쿼리를 많이 사용한다.
- QueryDSL을 사용하면 JPQL을 동적으로 편리하게 생성할 수 잇다.
- 꼭 동적 쿼리가 아니라 정적 쿼리인 경우에도 직관적인 문법, 컴파일 시점 오류 발견, 코드 자동완성, 코드 재사용 등을 위해서 QueryDSL을 사용하는 것이 좋다.
- Querydsl은 JPQL을 코드로 만들 수 있도록 도와주는 빌더 역할을 할 뿐이기에 JPQL을 잘 이해하고 있다면 금방 배울 수 있다


</div>
</details>
