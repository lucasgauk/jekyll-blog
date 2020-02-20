---
layout: post
title: GraphQL basics with Spring
---

Designed as an alternative to REST, GraphQL is a fairly recent query language developed by 
Facebook that specifies a new way for clients to interact with and request data from APIs. 
Today we're going to be setting up a basic Spring GraphQL API and in doing so exploring the pros and cons
of this new tool. This one is gonna be a bit long, so bring snacks.

A repository with the full working API and all of the code discussed in this article can be found 
[here](https://github.com/lucasgauk/graphql-api)


##  The Dependencies

To start with, we're going to need to add the following dependencies to our project. 

```xml
<dependency>
  <groupId>com.graphql-java</groupId>
  <artifactId>graphql-spring-boot-starter</artifactId>
  <version>5.0.2</version>
</dependency>
<dependency>
  <groupId>com.graphql-java</groupId>
  <artifactId>graphql-java-tools</artifactId>
  <version>5.2.4</version>
</dependency>

<dependency>
  <groupId>com.graphql-java</groupId>
  <artifactId>graphiql-spring-boot-starter</artifactId>
  <version>5.0.2</version>
</dependency>
```

The last GraphIQL dependency exposes an interactive web based GraphQL IDE at the /graphiql endpoint. 
We will be exploring this more later when we start running queries.    

## The Collection

We're going to be setting up a basic Order / Payment database using MongoDb for this example. Now
normally for something like Orders and Payments I would embed the Payment objects into the 
Order object directly, but for the purposes of exploring GraphQL I am going to keep them in separate
collections. We'll see a bit more about this later.

```java 
@Data
@Document
@TypeAlias("Order")
@Builder
public class Order {

  @Id
  private ObjectId id;
  @Default private OrderStatus status = OrderStatus.IN_PROGRESS;
  private BigDecimal total;
  @CreatedDate private LocalDateTime createdAt;

}
```

```java 
@Data
@Document
@TypeAlias("Payment")
@Builder
public class Payment {

  @Id private ObjectId id;
  private ObjectId orderId;
  private BigDecimal amount;
  @CreatedDate LocalDateTime createdAt;

}
```

I've also created a boilerplate set of repositories and services for accessing and modifying these 
objects. If you would like to see them they are available in the repository linked above but they 
are not critical to the point of this article.  

## Types and Queries

GraphQL as a language revolves around asking for specific fields on an object. In order to do this,
GraphQL needs a way of describing the resources available on the server to the client requesting them.
This is called the `GraphQL Schema Language`.

The most basic component of this language is the `Type`. A Type represents an object that exists on 
the server. Types contain strongly typed fields. For example, our Order object may look like the following

``` 
type Order {
  id: String!
  total: BigDecimal,
  status: OrderStatus
  createdAt: String!
}

enum OrderStatus {
  COMPLETED,
  IN_PROGRESS
}
```

A `String` and `BigDecimal` are two built in GraphQL Scalar types. 
Scalars are the leaf nodes of our queries and will resolve to concrete data. 

The `!` denotes a non nullable field, meaning that the client can always expect that field to 
be non null on the resource.

GraphQL also supports Enums. Enums are a special kind of scalar that is restricted to a specific 
set of possible values.

These Schemas are defined in text files with the `.graphqls` extension and can be anywhere in the 
classpath. They can be split into multiple files but will describe one singular Schema.

Now that we have our basic Order object defined by our schema, we need to define a way of accessing 
this object. This is where the `Query` type comes in. While most Types in the Schema define normal 
objects, the Query Type is special within the Schema and serves as the entry point of every GraphQL 
query. Every GraphQL service needs to have a Query Type.

Lets start by defining a Query Type that allows our users to fetch a group of Orders from the 
database.

``` 
type Query {
  orders(count: Int!, offset: Int!): [Order]!
}
```

Parentheses after a field denote possible arguments for that field. The orders field on our Query
type takes two non null arguments, count and offset, and will return a non null list of Orders.

One thing to note is that GraphQL can only have one root Query defined in the schema. This is bad
if you would like to keep fields related to specific Types within different files. It would
be hard to maintain one large `.graphqls` file that defined every single Type in your system. 
For us that would mean having the Payment Type in the same file as the Order Type and having 
fields related to Payment in the Query along with our Order fields. In GraphQL, Types can 
only be defined once but can be extended infinitely. So a quick workaround for this is to define the root
query in a file then extend it in each separate file. So we would wind up with two files:

root.graphqls
``` 
type Query {

}
```

order.graphqls
``` 
type Order {
  id: String!
  total: BigDecimal,
  status: OrderStatus
  createdAt: String!
}

enum OrderStatus {
  COMPLETED,
  IN_PROGRESS
}

extend type Query {
  orders(count: Int!, offset: Int!): [Order]!
}
```

## Query Resolvers

Now that we have a few Types and a root Query specified in our schema, we need to implement a 
special bean to resolve the fields from our root Query and return concrete data to our client.

This bean must implement `GraphQLQueryResolver`, and should have methods that match the fields (and their
arguments) in our Query Type and resolve them. For example, a Query resolver for our root Query so
far might look something like the following.

```java 
@Component
public class OrderQuery implements GraphQLQueryResolver {

  private final OrderService orderService;
  private final OrderTemplateService orderTemplateService;

  public OrderQuery(OrderService orderService, OrderTemplateService orderTemplateService) {
    this.orderService = orderService;
    this.orderTemplateService = orderTemplateService;
  }

  public List<Order> orders(int count, int offset) {
    return this.orderService.findAll(PageRequest.of(offset, count)).getContent();
  }

}
```

Our `orders` field in the Query Type is resolved by our `orders(int, int)` method in the bean.

Once we have this bean defined we can run our application and start using our new order field.

Populate your database with dummy data and lets try a basic query by navigating to
our /graphiql endpoint and typing something like the following.

```
{
  orders(count: 10, offset: 0) {
    id,
    total,
    createdAt
  }
}
```

Basic queries start by specifying the field on the Query Type they wish to access, the arguments,
then the fields on the response type that they wish to receive. In this case we are taking the first
10 orders in our database and we are interested in the id, the total, and the created date of these orders.

This query might result in the following response

```json
{
  "data": {
    "orders": [
      {
        "id": "5e4cc0d47e17b84d7609fd95",
        "total": 10,
        "createdAt": "2020-02-18T22:00:04.537"
      }
    ]
  }
}
```

So here we see a couple of benefits of GraphQL. 

The most obvious immediate benefit here is limiting network bandwidth used by sending useless fields. 
One thing to notice though is that the API is still fetching every field from the database and
instantiating it into a List. Don't think that GraphQL is somehow a solution to high API memory usage.

Next is the abstraction of the traditional REST
endpoints away in favour of a single /graphql endpoint that resolves depending on the query passed
to the endpoint. We also no longer handle the different `GET/POST/PUT/DELETE` calls, or response codes.
This isn't necessarily a benefit, and may not be right for all applications.

## Field Resolvers

Lets look at what I think might be one of the most interesting GraphQL features, field resolvers.
The GraphQL schema language allows us to enable queries that request multiple resources at once. Lets
explore that a bit.

Say we define a Payment Type for our Payment object.
``` 
type Payment {
  id: String!
  orderId: String!
  amount: BigDecimal
  createdAt: String!
}
```

Then say we modified our Order object to contain a List of Payment objects.

```java 
public class Order {

  ...
  private List<Payment> payments;
  ...

}

```

And then changed our Order Type
``` 
type Order {
  id: String!
  total: BigDecimal,
  status: OrderStatus
  payments: [Payment]!
  createdAt: String!
}
```

We would already be able to send queries asking for specific fields on our payments, or omit 
the payments list altogether, saving bandwidth. Something like the following

``` 
{
  orders(count: 10, offset: 0) {
    id,
    total,
    payments {
      id,
      amount
    }
  }
}
```

But this really isn't all that interesting. Our services are required to fetch the whole object every 
time anyways so we're really only saving bandwidth. This kind of field specificity could be 
implemented with REST with something like `GET /foo?fields=baz,bar`. 

The more interesting way to implement this is to use field resolvers. If a field in a Type is non trivial
to load (not just property on the object with a getter) then a custom field resolver can be implemented.

So if we go back to our original (separate tables) implementation of Order and Payment, but we keep
the new schema defining payments as a field on the Order schema, we can implement a field resolver 
to fetch the payments from the separate table if requested. 

```java 
@Component
public class OrderResolver implements GraphQLResolver<Order> {

  private PaymentService paymentService;

  public OrderResolver(PaymentService paymentService) {
    this.paymentService = paymentService;
  }

  public List<Payment> getPayments(Order order) {
    return this.paymentService.findAllByOrderId(order.getId());
  }

}
```

Our resolver needs to implement `GraphQLResolver` and for the fields that it wishes to resolve must
provide a method signature that matches the field name and arguments from the Type along with the 
object that the field is being requested on.

In this case, when payments is requested on the Order object we do a lookup against the Payment 
collection. Now when payments is not requested we are actually saving server resources
by never instantiating or fetching anything related to payments. 

This sort of resolver structure could enable some really clean single requests for multiple resources and 
drastically reduce the number of calls the API users need to make for related
resources stored in different collections, saving front end dev work and creating more maintainable front end
code.

On the flip side, combining calls to multiple separate resources into one request can be a bit dangerous. 
The request becomes all-or-nothing, with the client forced to wait for lookups to every resource before getting any
information back to render. If part of the query fails it can be much more catastrophic to the user experience, as the whole
query will fail together. Finding errors can get more troublesome as you're never quite as sure what went wrong.

It's up to you to determine whether this is right for your application. I haven't done any performance
testing but I could see this being slower than a traditional MongoDb lookup implemented
at some sort of specific order-with-payments endpoint in REST, but this code is super clean and 
convenient for the API users and maintainers. More hardware is almost always cheaper than more dev work.

## More Arguments

Lets look at field arguments a bit more. We've been introduced to them in our Query Type, but they 
can be applied to any field in any Type in our Schema. For example, in our Order schema we could allow
our users to specify the date format they'd like to receive the createdAt string in.

``` 
type Order {
  ...
  createdAt(format: String = "yyyy-MM-dd HH:mm:ss"): String!
}
```

Arguments can have default values. These are specified by adding an equals sign and giving them
a value like in Typescript.

Now we need to go back to our resolver handle the new argument.

```java 
@Component
public class OrderResolver implements GraphQLResolver<Order> {

  ...
  public String getCreatedAt(Order order, String format) {
    try {
      DateTimeFormatter formatter = DateTimeFormatter.ofPattern(format);
      return formatter.format(order.getCreatedAt());
    } catch (IllegalArgumentException e) {
      return order.getCreatedAt().toString();
    }
  }

}
``` 

Like before, we specify how to resolve the field for a given Order and argument. Every field and 
nested object can have it's own arguments enabling some pretty complex single queries.

## Custom Scalars

We've seen a few of the basic default scalars in GraphQL, but if we want something that isn't supported
out of the box like Dates or something that requires some custom validation like emails then we need a
way of defining custom scalars. Lets add a custom scalar to support LocalDateTime. Our scalar will
follow the ISO standard format of `yyyy-mm-ddThh:mm:ss` and be UTC. 

Custom scalars can be defined in the schema as follows.

``` 
scalar Date
``` 

We then need to provide a bean that will explain how to handle this new scalar. Our bean should 
extend `GraphQLScalarType` and provide a `Coercing` that explains how to deal with the scalar.

Ours might look like the following.

```java 
@Component
public class DateScalarType extends GraphQLScalarType {

  DateScalarType() {

    super("Date", "Date value", new Coercing<LocalDateTime, String>() {
      public String serialize(Object o) throws CoercingSerializeException {
        return serializeLocalDateTime(o);
      }

      public LocalDateTime parseValue(Object o) throws CoercingParseValueException {
        return parseLocalDateTimeFromValue(o);
      }

      public LocalDateTime parseLiteral(Object o) throws CoercingParseLiteralException {
        return parseLocalDateTimeFromLiteral(o);
      }

    });
  }

  private static String serializeLocalDateTime(Object o) {
    if (o instanceof LocalDateTime) {
      LocalDateTime dateTime = (LocalDateTime) o;
      return dateTime.toString();
    }

    throw new CoercingSerializeException("Object not an instance of LocalDateTime");
  }

  private static LocalDateTime parseLocalDateTimeFromValue(Object o) {
    if (o instanceof String) {
      String dateTimeString = (String) o;
      try {
        return LocalDateTime.parse(dateTimeString);
      } catch (DateTimeParseException e) {
        throw new CoercingParseValueException("Unable to create LocalDateTime from String");
      }
    }
    throw new CoercingParseValueException("Object not instance of String");
  }

  private static LocalDateTime parseLocalDateTimeFromLiteral(Object o) {
    if (o instanceof StringValue) {
      StringValue dateTimeString = (StringValue) o;
      try {
        return LocalDateTime.parse(dateTimeString.getValue());
      } catch (DateTimeParseException e) {
        throw new CoercingParseValueException("Unable to create LocalDateTime from StringValue");
      }
    }
    throw new CoercingParseValueException("Object not instance of StringValue");
  }

}
```

Now we can specify a field in our root Query that gets orders between two dates.

``` 
extend type Query {
  ...
  ordersBetween(start: Date, end: Date): [Order]!
}
```

And implement it in our Query resolver as the following.

```java 
@Component
public class OrderQuery implements GraphQLQueryResolver {
  
  ...
  public List<Order> ordersBetween(LocalDateTime start, LocalDateTime end) {
    return this.orderTemplateService.ordersBetween(start, end);
  }

}
```

The query for this endpoint might look like 

``` 
{
  ordersBetween(start: "2019-01-01T00:00:00", end: "2022-01-01T00:00:00") {
    id,
    total,
    createdAt
  }
}
```

## Mutations

So far we've only dealt with data fetching. There's nothing stopping you from writing a field into
your root Query that takes arguments and persists data but that would be a bit like having a `GET`
endpoint that creates data, its bad practice. That's where the Mutation Type comes in. 

Like the Query Type, Mutation is a special Type that contains fields that specify what and how you 
can change the data on the API. This mutation needs resolvers in the same way our root Query does.

Our Mutation for Order and Payment might look a bit like the following

In root.graphqls
``` 
type Mutation {

}
```

In order.graphqls
``` 
extend type Mutation {
  createOrder(total: BigDecimal): Order!
}
```

In payment.graphqls
``` 
extend type Mutation {
  createPayment(orderId: String!, amount: BigDecimal!): Payment!
}
```  

And we could define a resolver for the Order fields as the following

```java 
@Component
public class OrderMutation implements GraphQLMutationResolver {

  private OrderService orderService;

  public OrderMutation(OrderService orderService) {
    this.orderService = orderService;
  }

  public Order createOrder(BigDecimal total) {
    return this.orderService.newOrder(total);
  }

}
```

And for the Payment fields

```java 
@Component
public class PaymentMutation implements GraphQLMutationResolver {

  private final PaymentService paymentService;

  public PaymentMutation(PaymentService paymentService) {
    this.paymentService = paymentService;
  }

  public Payment createPayment(String orderId, BigDecimal amount) {
    return this.paymentService.create(orderId, amount);
  }

}
```

Now we can run our application and test our new mutations. Mutations are like queries but with a prepended `mutation`
keyword. We can still specify fields if the mutation returns a value. For example, to create a new Order we might write

``` 
mutation {
  createOrder(total: 100) {
    id,
    total,
    createdAt
  }
}
```

And expect something like the following as a response.

```json
{
  "data": {
    "createOrder": {
      "id": "5e4dc181a219637ac1f42948",
      "total": 100,
      "createdAt": "2020-02-19 16:15:13"
    }
  }
}
```

## Error Handling

One of the disadvantages of GraphQL is that we abstract away the status codes for our requests. Every request will result in 
a `200 OK` even if something catastrophic happens on the server. When an error occurs the data field on the GraphQL response
will be null and the errors field may contain information regarding the error.

For our new Payment mutation for example we probably want to inform the user if the orderId they passed doesn't correspond 
to an existing order. With traditional REST you would likely return a 400 along with a message to this effect. In GraphQL 
we no longer have status codes but we can throw an error give our users a bit more information about what went wrong.

GraphQL will not put regular exceptions thrown into the response, but we can define our own custom exceptions that
implement `GraphQLError`. For example, our Order exception could look like the following.

```java 
public class OrderNotFoundException extends RuntimeException implements GraphQLError {

  public OrderNotFoundException(String orderId) {
    super("Order id: " + orderId + " not found");
  }

  public List<SourceLocation> getLocations() {
    return null;
  }

  public ErrorType getErrorType() {
    return ErrorType.DataFetchingException;
  }

}
```

If this exception is thrown by our service as seen in the following method signature

```java 
Payment create(String orderId, BigDecimal amount) throws OrderNotFoundException;
```

Then GraphQL will include it in the errors field of the response so the API users know what is going on.

## Endpoint Versions 

GraphQL allows us to skip traditional cumbersome REST versioning by providing some strong tools for continuous Schema
evolution. In traditional REST, any potentially breaking change would require a version increment. Since GraphQL only 
returns data that is explicitly requested new functionality can be added without requiring new versions. 

Fields can also be marked as deprecated by applying a deprecated directive to the schema as follows.

``` 
type Order {
  ...
  status: OrderStatus @deprecated(reason: "Poor implementation")
}
```

Directives are denoted by the `@` symbol and can be applied to many aspects of the Schema. You can declare custom
directives and implement your own behaviour for each. I'm not going to get into details here but it can be an alternative
to the custom scalars seen earlier. 
 
## Wrap Up

So we've seen how to set up a basic API using GraphQL. We've seen some of the possibilities of GraphQL and some of the
pros and cons when compared to traditional REST. GraphQL isn't for everyone, there are some important usage and 
performance considerations to keep in mind before switching, but hopefully now you know a bit more about how it works.

If your API already works with REST, I think its hard to justify the switch. But if you're starting something new, at least
you have an intro to a new tool you may want to use.

Have fun, and happy coding!

