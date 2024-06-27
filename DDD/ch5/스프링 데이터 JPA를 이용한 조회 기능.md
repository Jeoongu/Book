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

### 5.5 스펙 조합
- 스프링 데이터 JPA가 제공하는 스펙 인터페이스는 스펙을 조합할 수 있는 `and()`와 `or()`메서드이다.
- `and()` 메서드는 두 스펙을 모두 충족하는 조건을 표현하는 스펙을 생성한다.
- `or()` 메서드는 두 스펙 중 하나 이상 충족하는 조건을 표현하는 스펙을 생성한다.
- `not()` 메서드도 제공한다. 정적 메서드로 조건을 반대로 적용할 때 사용한다.
- `where()` 메서드를 사용하면 null 여부 검사가 쉬워진다.

### 5.6 정렬 지정하기
- 스프링 데이터 JPA는 두 가지 방법을 사용해서 정렬을 지정할 수 있다.
    - 메서드 이름에 OrderBy를 사용해서 정렬 기준 지정
    ```java
    List<OrderSummary> findByOrdererIdOrderByNumberDesc(String ordererId);
    ```
    - Sort를 인자로 전달
    ```java
    List<OrderSummary> findByOrdererId(String ordererId, Sort sort);
    ...
    Sort sort = Sort.by("number").ascending();
    List<OrderSummary> results = orderSummaryDao.findByOrdererId("user1", sort);
    ```

### 5.7 페이징 처리하기
- 목록을 보여줄 때 전체 데이터 중 일부만 보여주는 페이징 처리는 기본이다.
- 스프링 데이터 JPA는 페이징 처리를 위해 Pageable 타입을 이용한다. Sort 타입과 마찬가지로 find 메서드에 Pageable 타입 파라미터를 사용하면 페이징을 자동으로 처리해 준다.
```java
// JpaRepository를 상속받는 interface
List<MemberData> findByNameLike(String name, Pageable pageable);
```
```java
Sort sort = Sort.by("name").descending;
PageRequest req = PageRequest.of(1, 10, sort); // 첫 번째 인자는 페이지 번호, 두 번째 인자는 한 페이지의 개수
List<MemberData> user = memberDataDao.findByNameLike("사용자%", req);
```
- Page 타입을 사용하면 데이터 목록뿐만 아니라 조건에 해당하는 전체 개수도 구할 수 있다.
```java
Page<MemberData> findByBlocked(boolean blocked, Pageable pageable);
```
- Pageable을 사용하는 메서드의 리턴 타입이 Page일 경우, 스프링 데이터 JPA는 목록 조회 쿼리와 함께 COUNT 쿼리도 실행해서 조건에 해당하는 데이터 개수를 구한다.
```java
PageRequest req = PageRequest.of(2, 3);
Page<MemberData> page = memberDataDao.findByBlocked(false, req);
// 이후 page.get~~()로 페이징 처리에 필요한 데이터를 얻을 수 있다.
```
- 하지만 페이징 처리와 관련된 정보가 필요 업다면 Page 리턴 타입이 아닌 List를 사용해서 불필요한 COUNT 쿼리를 실행하지 않도록 한다.
- 처음부터 N개의 데이터가 필요하다면 Pageable을 사용하지 않고 `findFirstN()` 형식의 메서드를 사용할 수 도 있다.

### 5.8 스펙 조합을 위한 스펙 빌더 클래스
- 스펙을 생성하다 보면 스펙을 조합해야 할 때가 있다.
- 단점을 보완하기 위해 `SpecBuilder`를 사용한다.
```java
Specfication<MemberData> spec = SpecBuilder.builder(MemberData.class)
    .ifTrue(searchRequest.isOnlyNotBlocked(),
        () -> MemberDataSpecs.nonBlocked())
    .ifHasText(searchRequest.getName(),
        name -> MemberDataSpecs.nameLike(searchRequest.getName()))
    .toSpec();
List<MemberData> result = memberDataDao.findAll(spec, PageRequest.of(0, 5));
```
- 스펙 빌더는 `and()`, `ifHasNext()`, `ifTure()`, `toSpec()` 메서드가 있는데, 이외에 필요한 메서드를 추가해서 사용하면 된다.

### 5.9 동적 인스턴스 생성
- **JPA는 쿼리 결과에서 임의의 객체를 동적으로 생성할 수 있는 기능을 제공하고 있다.**
- JPQL의 select절에 new 키워드를 사용하여 동적 인스턴스를 생성할 수 있다. new 키워드 뒤에 생성할 인스턴스의 완전한 클래스 이름을 지정하고 괄호 안에 생성자에 인자로 전달할 값을 지정한다.
- 조회 전용 모델을 만드는 이유는 표현 영역을 통해 사용자에게 데이터를 보여주기 위함이다.
- 동적 인스턴스의 장점은 JPQL을 그대로 사용하므로 객체 기준으로 쿼리를 작성하면서도 동시에 지연/즉시 로디과 같은 고민 없이 원하는 모습으로 데이터를 조회할 수 있다는 점이다.

### 5.10 하이버네이트 @Subselect 사용
- 하이버네이트는 JPA 확장 기능으로 `@Subselect`를 제공한다.
    - `@Subselect`는 쿼리 결과를 `@Entity`로 매핑할 수 있는 유용한 기능이다.
    ```java
    @Entity
    @Immutable
    @Subselect(
        ......
        select 어쩌구 저쩌구...
        ......
    )
    @Synchronize({"purchase_order", "order_line", "product"})
    public class OrderSummary{
        @Id
        ......
    }
    ```
- `@Immutable`, `@Subselect`, `@Synchronize`는 하이버네이트 전용 애너테이션인데 이 태그를 사용하면 테이블이 아닌 쿼리 결과를 `@Entity`로 매핑할 수 있다.
- `@Subselect`는 조회(select)쿼리를 값으로 갖는다. 하이버네이트는 이 select 쿼리의 결과를 매핑할 테이블처럼 사용한다.
- DBMS가 뷰를 사용하는 것처럼 `@Subselect`를 사용하면 쿼리 실행 결과를 매핑할 테이블처럼 사용한다.
- 뷰를 수정할 수 없듯이 `@Subselect`로 조회한 `@Entity` 역시 수정할 수 없다. 실수로 `@Subselect`를 이용한 `@Entity`의 매핑 필드를 수정하면 하이버네이트는 변경 내역을 반영하는 update 쿼리를 실행할 것이다. 그런데 매핑 한 테이블이 없으므로 에러가 발생한다.
    - 이런 문제를 방지하기 위해 `@Immutable`을 사용한다.
    - `@Immutable`을 사용하면 하이버네이트는 해당 엔티티의 매핑 필드/프로퍼티가 변경되도 DB에 반영하지 않고 무시한다.
- `@Synchronize`는 해당 엔티티와 관련된 테이블 목록을 명시한다. 하이버네이트는 엔티티를 로딩하기 전에 지정한 테이블과 관련된 변경이 발생하면 플러시(flush)를 먼저 한다. 따라서 OrderSummary를 로딩하는 시점에서는 변경 내역이 반영된다.