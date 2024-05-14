# ch5. 스프링 데이터 JPA를 이용한 조회 기능

### 5.1 시작에 앞서
- CQRS는 명령(Command)모델과 조회(Query)모델을 분리하는 패턴이다.
- 명령 모델은 상태를 변경하는 기능을 구현할 때 사용한다.
- 조회 모델은 데이터를 조회하는 기능을 구현할 때 사용한다.
- 엔티티, 애그리거트, 리포지터리 등 앞에서 살펴봤던 모델은 상태를 주로 변경한다. **즉, 도메인 모델은 명령 모델로 주로 사용된다.**
- 하지만 ch5에서 설명할 정렬, 페이징, 검색 지정 조건과 같은 기능은 조회 기능에 사용된다. 즉, 이 장에서 살펴볼 구현 방법은 조회 모델을 구현할 때 주로 사용한다.
- 이 장의 예제 코드는 리포지터리(도메인 모델에 속한)와 DAO(데이터 접근을 의미하는)라는 이름을 혼용해서 사용한다.
> 모든 DB 연동 코드를 JPA만 사용해서 구현해야 한다고 생각하지는 말자.

### 5.2 검색을 위한 스펙
- 검색 조건이 고정되어 있고 단순하면 특정 조건으로 조회하는 기능을 만들면 되지만, 목록 조회와 같은 기능은 다양한 검색 조건을 조합해야 할 때가 있다.
- 필요한 조합마다 find 메서드를 정의할 수도 있지만 이건 좋은 방법이 아니다. 조합이 증가할수록 정의해야 할 find 메서드도 함께 증가하기 때문이다.
- 이렇게 검색 조건을 다양하게 조합해야 할 때 사용할 수 있는 것이 **스펙(Specification)**이다.
    - 스펙은 애그리거트가 특정 조건을 충족하는지 검사할 때 사용하는 인터페이스다.
    ```java
    public interface Specficiation<T>{
        public boolean isSatisfiedBy(T agg);
    }
    ```
    - `isSatisfiedBy()` 메서드의 agg 파라미터는 검사 대상이 되는 개체이다.
    - 스펙을 리포지터리에 사용하면 agg는 애그리거트 루트가 되고, 스펙을 DAO에 사용하면 agg는 검색 결과로 리턴할 데이터 객체가 된다.
    - `isSatisfiedBy()` 메서드는 검사 대상 객체가 조건을 충족하면 true, 아니면 false를 리턴한다.
    ```java
    public class OrdererSpec implements Specification<Order>{
        private String ordererId;
        ...
        public boolean isSatisfiedBy(Order agg){
            return agg.getOrdererId().getMemberId().getId().equals(ordererId);
        }
    }
    ```
    - 리포지터리나 DAO는 검색 대상을 걸러내는 용도로 스펙을 사용한다.
    ```java
    public class MemoryOrderRepository implements OrderRepository {
        public List<Order> findAll(Specification<Order> spec){
            List<Order> allOrders = findAll();
            return allOrders.stream().
                            .filter(order -> spec.isSatisfiedBy(order))
                            .toList();
        }
    }
    ```
    - 리포지터리가 스펙을 이용해서 검색 대상을 걸러주므로 특정 조건을 충족하는 애그리거트를 찾고 싶으면 원하는 스펙을 생성해서 리포지터리에 전달해 주기만 하면 된다.
    ```java
    // 검색 조건을 표현하는 스펙을 사용해서
    Specification<Order> ordererSpec = new OrdererSpec("madvirus");

    // 리포지터리에 전달
    List<Order> orders = orderRepository.findAll(ordererSpec);
    ```
    - 하지만 실제 스펙은 사용하는 기술에 맞춰 구현하게 된다.

### 5.3 스프링 데이터 JPA를 이용한 스펙 구현
- Specification 인터페이스에서 제네릭 타입 파라미터 T는 JPA 엔티티 타입을 의미한다.
- toPredicate() 메서드는 JPA criteria API에서 조건을 표현하는 Predicate을 생성한다.

- JPA 정적 메타 모델
    - 구현 시 `@StaticMetamodel`와 같이 애너테이션을 사용한다. 메타 모델 클래스는 모델 클래스의 이름 뒤에 _을 붙인 이름을 갖는다.

- 5.3 다시 공부 필요

### 5.4 리포지터리/DAO에서 스펙 사용하기
- 스펙을 충족하는 엔티티를 검색하고 싶다면 `findAll()` 메서드를 사용하면 된다. `findAll()`메서드는 스펙 인터페이스를 파라미터로 갖는다.
```java
public interface OrderSummaryDao extends Repository<OrderSummary, String>{
    List<OrderSummary> findAll(Specification<OrderSummary> spec);
}
```
- 위의 findAll() 메서드는 OrderSummary에 대한 검색 조건을 표현하는 스펙 인터페이스를 파라미터로 갖는다.
- 이 메서드와 앞서 작성한 스펙 구현체를 사용하면 특정 조건을 충족하는 엔티티를 검색할 수 있다.
```java
// 스펙 객체 생성하고
Specification<OrderSummary> spec = new OrdererIdSpec("user1");
// findAll() 메서드를 이용해서 검색
List<OrderSummary> results = orderSummaryDao.findAll(spec);
```