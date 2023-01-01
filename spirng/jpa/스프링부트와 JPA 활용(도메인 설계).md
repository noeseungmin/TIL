# 스프링부트와 JPA 활용

> [[실전! 스프링 부트와 JPA 활용1 - 웹 애플리케이션 개발]](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-%ED%99%9C%EC%9A%A9-1/dashboard) 강의 학습 내용입니다.

## 도메인 설계

* 회원 기능
  * 회원 등록
  * 회원 조회
* 상품 기능
  * 상품 등록
  * 상품 수정
  * 상품 조회
* 주문 기능
  * 상품 주문
  * 주문 내역 조회
  * 주문 취소
* 기타 요구사항
  * 상품은 재고 관리가 필요하다.
  * 상품의 종류는 도서, 음반, 영화가 있다.
  * 상품을 카테고리로 구분할 수 있다.
  * 상품 주문시 배송 정보를 입력 할 수 있다.

## 엔티티 클래스 개발

### 회원 엔티티
```java
@Entity
@Getter @Setter
public class Member {

    @Id @GeneratedValue
    @Column(name = "member_id")
    private Long id;

    private String username;

    @Embedded //내장타입이다.
    private Address address;

    @OneToMany(mappedBy = "member") //order 테이블에 있는 member 필드에 의해 매핑되었다는걸 의미
    private List<Order> orders = new ArrayList<>();

}
```
### 주문 
```java
@Entity
@Table(name = "orders")
@Getter @Setter
public class Order {

    @Id @Generated
    @Column(name = "order_id")
    private Long id;

    @ManyToOne(fetch = LAZY) //연관관계 설정
    @JoinColumn(name = "member_id")
    private Member member;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> orderItems = new ArrayList<>();

    @OneToOne(cascade = CascadeType.ALL, fetch = LAZY)
    @JoinColumn(name = "delivery_id")
    private Delivery delivery;

    private LocalDateTime orderDate; //주문시간

    @Enumerated(EnumType.STRING)
    private OrderStatus status; //ENUM 형식 주문 상태 표시 [ORDER, CANCEL]

    //연관관계 메서드
    public void setMember(Member member){
        this.member = member;
        member.getOrders().add(this);
    }

    public static void main(String[] args) {
        Member member = new Member();
        Order order = new Order();

        order.setMember(member);
    }

    public  void addOrderItem(OrderItem orderItem){
        orderItems.add(orderItem);
        orderItem.setOrder(this);
    }

    public void setDelivery(Delivery delivery){
        this.delivery = delivery;
        delivery.setOrder(this);
    }
}
```

### 주문상태
```java
public enum OrderStatus {
    ORDER, CANCEL
}

```

### 주문상품 엔티티
```java
@Entity
@Getter @Setter
public class OrderItem {

    @Id @GeneratedValue
    @Column(name = "order_item_id")
    private Long id;

    @ManyToOne(fetch = LAZY)
    @JoinColumn(name = "item_id")
    private Item Item;

    @ManyToOne(fetch = LAZY)
    @JoinColumn(name = "order_id")
    private Order order;

    private int orderPrice;
    private int count;
}
```

### 상품 엔티티
```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED) //상속관계 전략 부모테이블에 지정 1.joined 2.SINGLE_TABLE 3.TABLE_PER_CLASS
@DiscriminatorColumn(name = "DTYPE")
@Getter @Setter
public abstract class Item {

    @Id
    @GeneratedValue
    @Column(name = "item_id")
    private Long id;

    private String name;
    private int price;
    private int stockQuantity;


    @ManyToMany(mappedBy = "items")
    private List<Category> categories = new ArrayList<>();
}
```

### 상품 - 도서
```java
@Entity
@DiscriminatorValue("B")
@Getter
@Setter
public class Book extends Item{

    private String author;
    private String isbn;

}
```

### 상품 - 음반
```java
@Entity
@DiscriminatorValue("A")
@Getter
@Setter
public class Album extends Item{

    private String artist;
    private String etc;
}
```

### 상품 - 영화
```java
@Entity
@DiscriminatorValue("M")
@Getter @Setter
public class Movie extends Item{

    private String director;
    private String actor;
}
```

### 배송 엔티티
```java
@Entity
@Getter @Setter
public class Delivery {

    @Id @GeneratedValue
    @Column(name = "delivery_id")
    private Long id;

    @OneToOne(mappedBy = "delivery", fetch = LAZY)
    private Order order;

    @Embedded
    private  Address address;

    @Enumerated(EnumType.STRING) //ORDINAL 숫자, STRING
    private DeliveryStaus status; //READY, COMP
}
```

### 배송 상태
```java
public enum DeliveryStaus {
    READY, COMP
}
```

### 카테고리 엔티티
```java
@Entity
@Getter @Setter
public class Category {

    @Id
    @GeneratedValue
    @Column(name = "category_id")
    private Long id;

    private String name;

    @ManyToMany(fetch = LAZY)
    @JoinTable(name = "category_item",
            joinColumns = @JoinColumn(name = "category_id"),
            inverseJoinColumns = @JoinColumn(name = "item_id"))
    private List<Item> items = new ArrayList<>();

    @ManyToOne(fetch = LAZY)
    @JoinColumn(name = "parent_id")
    private Category parent;

    @OneToMany(mappedBy = "parent")
    private List<Category> child = new ArrayList<>();

    //연관관계 메서드
    public void addChildCategory(Category child){
        this.child.add(child);
        child.setParent(this);
    }
}
```

### 주소 값 타입
```java
@Embeddable //어딘가에 내장 될 수 있다.
@Getter
public class Address {

    private String city;
    private String street;
    private String zipcode;

    protected Address() {
    }

    public Address(String city, String street, String zipcode) {
        this.city = city;
        this.street = street;
        this.zipcode = zipcode;
    }
}
```
