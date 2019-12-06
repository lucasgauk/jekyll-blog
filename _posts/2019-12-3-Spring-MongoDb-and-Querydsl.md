---
layout: post
title: Spring, MongoDb, and Querydsl
---

Front end applications often need to display filtered information to their users, and writing boilerplate 
endpoints to filter different properties on your collections can get tiresome. 
Today I'll take a quick look
at how you might use Querydsl to allow your API users to query your MongoDb collections themselves.
 
A repository containing a working copy of all of the code discussed in this post can be found 
[here](https://github.com/lucasgauk/spring-query-dsl).

##  The Dependencies

You'll need to add the following to your pom file
 
```xml

  <properties>
    <query-dsl.version>4.2.1</query-dsl.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>com.querydsl</groupId>
      <artifactId>querydsl-mongodb</artifactId>
      <version>${query-dsl.version}</version>
    </dependency>
    <dependency>
      <groupId>com.querydsl</groupId>
      <artifactId>querydsl-apt</artifactId>
      <version>${query-dsl.version}</version>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>com.mysema.maven</groupId>
        <artifactId>apt-maven-plugin</artifactId>
        <version>1.1.3</version>
        <executions>
          <execution>
            <phase>generate-sources</phase>
            <goals>
              <goal>process</goal>
            </goals>
            <configuration>
              <outputDirectory>${project.build.directory}/generated-sources</outputDirectory>
              <processor>org.springframework.data.mongodb.repository.support.MongoAnnotationProcessor</processor>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
```

## The Collection
Today we're going to be emulating a simple order API that tracks orders and payments taken against those orders.

I like to define a base entity class that defines a few useful items that each collection in my database will share.

For demonstration purposes the base entity class is going to look as follows (note that I am using [lombok](https://projectlombok.org/))

```java
@Data
public class BaseEntity {

 @Id
 private ObjectId id;
 @CreatedDate
 private LocalDateTime createdAt;

}
```

The main order class will look as follows

```java
@Getter
@Setter
@QueryEntity
@Document
@TypeAlias("order")
@AllArgsConstructor
@NoArgsConstructor
public class Order extends BaseEntity {

private String customerName;
private BigDecimal orderTotal;
private String orderStatus;
private List<Payment> payments = new ArrayList<>();
private LocalDateTime orderClosedAt;

public enum OrderStatus {
 IN_PROGRESS, COMPLETED
}

public static Order from(OrderRequest request) {
 return new Order(request.getCustomerName(),
                  request.getOrderTotal(),
                  OrderStatus.IN_PROGRESS.toString(),
                  new ArrayList<>(),
                  null);
}

}
```

Notice that I am using a `String` for the `orderStatus` rather than the `OrderStatus` directly.
This is absolutely bad practice but you'll see why I'm doing it later. 

API users will be able to add payments to the order. The payment class will look as follows

```java
@Data
@AllArgsConstructor
public class Payment {

    LocalDateTime processedAt;
    BigDecimal amount;

    public static Payment from(PaymentRequest request) {
        return new Payment(LocalDateTime.now(), request.getAmount());
    }

}

```

With these three classes we've defined a very basic order tracking system. Users can add orders and then
add payments against those orders. Orders can be closed if they have been paid in full.

I've skipped over a few small pieces, but they aren't critical to the point of this blog post.

## The Repository

Next we will have to add a repository for our new collection. It should look as follows

```java 
@Repository
public interface OrderRepository extends MongoRepository<Order, String>, QuerydslPredicateExecutor<Order> {
}
```

The `QueryDslPredicateExecutor` interface exposes some very useful methods for our collection.
The one we will be focusing on today is the `Iterable<T> findAll(Predicate var1)` method.

Our code today will essentially boil down to allowing our users to create a `Predicate` 
such that they can search through our collection themselves.

## Creating the Predicate

Now that we have this repository, all we really need to do is allow our users to build their own
`Predicate` classes to search for whatever they want on our collection. The following is my adaptation
of a blog post I found a while back regarding this. I wish I could link to it but I can no longer find
the original article. If any of the following looks familiar, shoot me an email!

This is the part where this becomes flexible. My implementation isn't great, I would recommend using
this as a starting point and refactoring it to suit your application better.

The idea boils down to allowing the user to create their own query strings defining a list 
of `AND` operations they wish to perform on the collection. Our job will be to parse this query
and create the `Predicate` for them. The following will be the format of the query:

```
<property name><operator><match object>,<property name><operator>...
```

We will define a single search criteria as follows

```java
@Data
@AllArgsConstructor
class SearchCriteria {

  private static final List<String> OPERATIONS = Arrays.asList("<", ">", ":");

  private String key;
  private String operation;
  private String value;

  Boolean isValid() {
    return key != null && OPERATIONS.contains(operation) && value != null;
  }

} 

```

As you can see, we accept three operations - less than, greater than, and equals. We will define a class
to create a `Predicate` from a `SearchCriteria` as follows

```java 
class BasicPredicate<T> {

  private SearchCriteria criteria;

  BasicPredicate(SearchCriteria criteria) {
    this.criteria = criteria;
  }

  BooleanExpression getPredicate(Class<T> tClass, String collectionName) {
    PathBuilder<T> entityPath = new PathBuilder<>(tClass, collectionName);

    // Date criteria
    if (this.isDate(criteria.getValue())) {
      DatePath<LocalDate> datePath = entityPath.getDate(criteria.getKey(), LocalDate.class);
      switch (criteria.getOperation()) {
        case ":":
          return datePath.eq(LocalDate.parse(criteria.getValue()));
        case ">":
          return datePath.gt(LocalDate.parse(criteria.getValue()));
        case "<":
          return datePath.lt(LocalDate.parse(criteria.getValue()));
      }
    }

    if (this.isNumber(criteria.getValue())) {
      NumberPath<BigDecimal> numPath = entityPath.getNumber(criteria.getKey(), BigDecimal.class);
      switch (criteria.getOperation()) {
        case ":":
          return numPath.eq(new BigDecimal(criteria.getValue()));
        case ">":
          return numPath.gt(new BigDecimal(criteria.getValue()));
        case "<":
          return numPath.lt(new BigDecimal(criteria.getValue()));
      }
    }

    StringPath stringPath = entityPath.getString(criteria.getKey());
    if (criteria.getOperation().equalsIgnoreCase(":")) {
      return stringPath.containsIgnoreCase(criteria.getValue());
    }

    return null;
  }

  private boolean isDate(String value) {
    try {
      LocalDate.parse(value);
    } catch (DateTimeParseException e) {
      return false;
    }
    return true;
  }

  private boolean isNumber(String value) {
    try {
      new BigDecimal(value);
    } catch (NumberFormatException e) {
      return false;
    }
    return true;
  }
}
```

Lets break this down. As you can see, the `BasicPredicate` has a SearchCriteria. Calling the 
`getPredicate` method will generate a new `BooleanExpression` predicate based on the SearchCriteria
contained by the class.

To do this I am using implementations of the dsl `Path` class. These classes contain a reference
to the property on the object you are looking to query, and can be given comparison operators to 
build a `Predicate`.

As you can see, I'm assuming that if I can parse the value as a date or a Number that the path
should be a `DatePath` or a `NumberPath` respectively. This is pretty bad. For the purposes
of this little demo it works, but it would maybe be best to use reflection or another method of
attempting to determine the class of the property on the object that this is used on. Or hell, maybe
it would be best if this wasn't generic. It would probably be safest for each class to have it's own
Predicate builder BasicPredicate -> OrderPredicate maybe. Anyways, take the idea and run with it.

Moving along, now what we need is a way to parse a whole query string into a bunch of `SearchCriteria`
and then build a combined Predicate from the entire query. That class might look like the following

```java 
public class BasicPredicateBuilder<T> {

  private List<SearchCriteria> criteria;
  private Class<T> tClass;
  private String collectionName;

  public BasicPredicateBuilder(Class<T> tClass, String collectionName) {
    criteria = new ArrayList<>();
    this.tClass = tClass;
    this.collectionName = collectionName;
  }

  public void from(String search) throws IllegalArgumentException {
    if (search != null) {
      Pattern pattern = Pattern.compile("([\\w.]+?)([:<>])([\\w.\\- ]+?),");
      Matcher matcher = pattern.matcher(search + ",");
      while (matcher.find()) {
        SearchCriteria c = new SearchCriteria(matcher.group(1), matcher.group(2), matcher.group(3));
        if (!c.isValid()) {
          throw new IllegalArgumentException("Invalid query: " + c.toString());
        }
        this.criteria.add(c);
      }
      if (this.criteria.size() == 0) {
        throw new IllegalArgumentException("Invalid query: " + search);
      }
    }
  }

  public void with(String key, String operation, String value) {
    this.criteria.add(new SearchCriteria(key, operation, value));
  }

  public BooleanExpression build() {
    if (this.criteria.size() == 0) {
      return null;
    }

    List<BooleanExpression> predicates = criteria.stream().map(param -> {
      BasicPredicate<T> predicate = new BasicPredicate<>(param);
      return predicate.getPredicate(this.tClass, this.collectionName);
    }).filter(Objects::nonNull).collect(Collectors.toList());

    BooleanExpression expression = predicates.remove(0);

    for (BooleanExpression item : predicates) {
      expression = expression.and(item);
    }

    return expression;

  }
}
```

For now the important method is the `from(String search)` method. This method uses regex to create
SearchCriteria from a query string. Then the build takes each of the criteria, uses the `BasicPredicate`
to create Predicates and then combines them all together as `AND` operations, returning a final
Predicate. This predicate can then be sent straight to our repository!

So that's the goofy stuff done. Let's write a bit more boilerplate code to let our users access these 
features.

## The Service and the Controller

I always like to abstract the repository away from the user with a service. In this case the service
also handles operations on the Order like adding a payment and closing the order.

A very simple service for Order manipulation might look like the following 

```java 
@Service
public class OrderServiceImpl implements OrderService {

    private final OrderRepository repository;

    public OrderServiceImpl(OrderRepository repository) {
        this.repository = repository;
    }

    public List<Order> search(Predicate p) {
        return Lists.newArrayList(this.repository.findAll(p));
    }

    public Order save(Order order) {
        return this.repository.save(order);
    }

    public void saveAll(List<Order> orders) {
        this.repository.saveAll(orders);
    }

    public Order addPayment(String orderId, Payment payment) {
        Order order = this.repository.findById(orderId).orElseThrow(() -> new OrderNotFoundException(orderId));
        order.getPayments().add(payment);
        BigDecimal paymentTotal = order.getPayments()
                                       .stream()
                                       .map(Payment::getAmount)
                                       .reduce(BigDecimal::add)
                                       .orElse(BigDecimal.ZERO);
        if (paymentTotal.compareTo(order.getOrderTotal()) >= 0) {
            this.closeOrder(order);
        }
        return this.save(order);
    }

    private void closeOrder(Order order) {
        order.setOrderStatus(OrderStatus.COMPLETED.toString());
        order.setOrderClosedAt(LocalDateTime.now());
    }
}
```

As you can see, we have a few boilerplate Order methods and a search method that sends the 
predicate straight to the Repository and creates a more manageable List with the result.

Now lets let our users go nuts 

```java 
@RestController
@RequestMapping
@CrossOrigin(origins = "*", maxAge = 3600)
public class OrderController {

  private final OrderService orderService;

  public OrderController(OrderService orderService) {
    this.orderService = orderService;
  }

  @GetMapping("/query")
  @ResponseBody
  public List<Order> query(@RequestParam String search) {
    BasicPredicateBuilder<Order> builder = new BasicPredicateBuilder<>(Order.class, "order");

    builder.from(search);
    BooleanExpression exp = builder.build();

    return this.orderService.search(exp);
  }

  @PostMapping
  @ResponseBody
  public Order create(@RequestBody OrderRequest request) {
    return this.orderService.save(Order.from(request));
  }

  @PostMapping("/{orderId}/payment")
  @ResponseBody
  public Order addPayment(@PathVariable String orderId, @RequestBody PaymentRequest request) {
    return this.orderService.addPayment(orderId, Payment.from(request));
  }

}
```

This controller exposes all of the stuff we've done so far as an API. Users can create Orders,
add Payments, and then search for them using our custom query language.

So what might this look like? Let's take a look at some tests.

## Tests and Final Results

Let's try searching for a couple things at once

```java 
    @Test
    public void searchForOrderByStatusAndCreatedDate() {
        Order o1 = new Order();
        o1.setOrderStatus(OrderStatus.COMPLETED.toString());
        Order o2 = new Order();
        o2.setOrderStatus(OrderStatus.IN_PROGRESS.toString());
        Order o3 = new Order();
        o3.setOrderStatus(OrderStatus.COMPLETED.toString());

        this.orderService.saveAll(Arrays.asList(o1, o2, o3));

        o1.setCreatedAt(o1.getCreatedAt().minusDays(10));
        o2.setCreatedAt(o2.getCreatedAt().minusDays(5));

        this.orderService.saveAll(Arrays.asList(o1, o2));

        String query1 = "orderStatus:IN_PROGRESS,createdAt>" + LocalDate.now().minusDays(6).toString();
        String query2 = "orderStatus:COMPLETED,createdAt>" + LocalDate.now().minusDays(6).toString();
        String query3 = "orderStatus:COMPLETED,createdAt<" + LocalDate.now().minusDays(1).toString();

        BasicPredicateBuilder<Order> builder1 = new BasicPredicateBuilder<>(Order.class, "order");
        builder1.from(query1);
        List<Order> query1Orders = this.orderService.search(builder1.build());

        assertThat(query1Orders.size()).isEqualTo(1);
        assertThat(query1Orders.get(0).getId()).isEqualTo(o2.getId());

        BasicPredicateBuilder<Order> builder2 = new BasicPredicateBuilder<>(Order.class, "order");
        builder2.from(query2);
        List<Order> query2Orders = this.orderService.search(builder2.build());

        assertThat(query2Orders.size()).isEqualTo(1);
        assertThat(query2Orders.get(0).getId()).isEqualTo(o3.getId());

        BasicPredicateBuilder<Order> builder3 = new BasicPredicateBuilder<>(Order.class, "order");
        builder3.from(query3);
        List<Order> query3Orders = this.orderService.search(builder3.build());

        assertThat(query3Orders.size()).isEqualTo(1);
        assertThat(query3Orders.get(0).getId()).isEqualTo(o1.getId());
    }
```

We can also search for properties on nested objects in a List! It will return any object that contains
a child object that matches that property.

```java 
    @Test
    public void searchForOrderByPayment() {
        Order o1 = new Order();
        o1.setOrderTotal(BigDecimal.ZERO);
        Order o2 = new Order();
        o2.setOrderTotal(BigDecimal.ONE);
        Order o3 = new Order();
        o3.setOrderTotal(BigDecimal.TEN);

        PaymentRequest p1 = new PaymentRequest(BigDecimal.ZERO);
        PaymentRequest p2 = new PaymentRequest(BigDecimal.ONE);
        PaymentRequest p3 = new PaymentRequest(BigDecimal.TEN);

        this.orderService.saveAll(Arrays.asList(o1, o2, o3));

        this.orderService.addPayment(o1.getId().toString(), Payment.from(p1));
        this.orderService.addPayment(o2.getId().toString(), Payment.from(p2));
        this.orderService.addPayment(o3.getId().toString(), Payment.from(p3));


        String query1 = "payments.amount>1";
        String query2 = "payments.amount:0";
        String query3 = "payments.amount<5";

        BasicPredicateBuilder<Order> builder1 = new BasicPredicateBuilder<>(Order.class, "order");
        builder1.from(query1);
        List<Order> query1Orders = this.orderService.search(builder1.build());

        assertThat(query1Orders.size()).isEqualTo(1);
        assertThat(query1Orders.get(0).getId()).isEqualTo(o3.getId());

        BasicPredicateBuilder<Order> builder2 = new BasicPredicateBuilder<>(Order.class, "order");
        builder2.from(query2);
        List<Order> query2Orders = this.orderService.search(builder2.build());

        assertThat(query2Orders.size()).isEqualTo(1);
        assertThat(query2Orders.get(0).getId()).isEqualTo(o1.getId());

        BasicPredicateBuilder<Order> builder3 = new BasicPredicateBuilder<>(Order.class, "order");
        builder3.from(query3);
        List<Order> query3Orders = this.orderService.search(builder3.build());

        assertThat(query3Orders.size()).isEqualTo(2);
    }
```

Pretty cool right?

## Wrap Up

So what have we done? We've essentially created a reusable query language that can be slapped onto
any basic collection to allow users to search for items that match any combination of basic criteria.

Some of the code in here is pretty sloppy but I hope I helped you get an idea of how you might implement this
in your own projects. 

Have fun, and happy coding!

