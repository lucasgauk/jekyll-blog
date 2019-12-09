---
layout: post
title: Timezones, Grouping, and Saturdays
---

Timezones are a presentation layer problem. It should always be up to the front end application to present UTC time in 
a user friendly timezone. But this isn't always possible. Today we're going to look at one Spring Boot / Angular use case
where this is no longer a possibility, and my thoughts about it.

## The Setup

Imagine we had an API that tracked shifts. Say we had a shift class that looked a bit like the following

```java 
@Document
@TypeAlias("Shift")
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Shift {

  @Id
  private ObjectId id;
  private String computerId;
  @Indexed private LocalDateTime closedAt;
  @Indexed private LocalDateTime createdAt;

}
```

Lets assume we're being responsible and saving all of our times as UTC.

Say that we had a class called `PaymentEntry` that tracked any payments applied against that shift. Say it looked a bit like the following

```java 
@Document
@TypeAlias("PaymentEntry")
@Data
@AllArgsConstructor
public class PaymentEntry {

  @Id
  private ObjectId id;
  private BigDecimal amount;
  private ObjectId shiftId;

}
```

Pretty straight forward. Now this isn't actually how you would do this, but for the purposes of this example say 
the front end wanted to know how much we had paid against each closed shift, as well as the time the shift was closed. 
So we decide to declare a class that will represent this data as the following 

```java 
@Data
@AllArgsConstructor
@NoArgsConstructor
public class ShiftPayment {
  
  private String shiftId;
  private LocalDateTime closedAt;
  private BigDecimal amountPaid;
  
}
```

Then we create a custom repository and `MongoTemplate` to implement our join and group by query as the following 

```java 
  public List<ShiftPayment> getShiftReportSummary() {
    Aggregation aggregation = Aggregation.newAggregation(match(Criteria.where("closedAt").exists(true)),
                                                         lookup("paymentEntry", "_id", "shiftId", "payments"),
                                                         unwind("payments"),
                                                         group("_id")
                                                             .first("_id").as("shiftId")
                                                             .first("closedAt").as("closedAt")
                                                             .sum("payments.amount").as("amountPaid"));
    return this.mongoTemplate.aggregate(aggregation, "shift", ShiftPayment.class).getMappedResults();
  }
```

Cool. Now the front end can grab this little summary and just use a date pipe to 
display the date. The date pipe defaults to the local system's default timezone.

Ok, so far so easy, but where's the problem?

## A Little More Complicated

Say the user only wanted to see shifts closed on a single given day. This is an interesting one, because days are different 
in each timezone. 

Again, no need to panic, a quick solution to this one is to take two `LocalDateTime` objects from the front end in UTC
and let the front end decide exactly what constitutes a day for them. Your method might look like this then

```java 
  public List<ShiftPayment> getShiftReportSummary(LocalDateTime startTime, LocalDateTime endTime) {
    Aggregation aggregation = Aggregation.newAggregation(match(Criteria.where("closedAt").exists(true)),
                                                         match(Criteria.where("closedAt").gte(startTime.toInstant(ZoneOffset.UTC))
                                                                       .lte(endTime.toInstant(ZoneOffset.UTC))),
                                                         lookup("paymentEntry", "_id", "shiftId", "payments"),
                                                         unwind("payments"),
                                                         group("_id")
                                                             .first("_id").as("shiftId")
                                                             .first("closedAt").as("closedAt")
                                                             .sum("payments.amount").as("amountPaid"));
    return this.mongoTemplate.aggregate(aggregation, "shift", ShiftPayment.class).getMappedResults();
  }
```

Cool, so we're still safe and the backend still doesn't care about timezones.

## A Problem

Say the user now wanted to see payments against any shift grouped by close day. Uh oh. Now we've got ourselves a problem.

Say a user in Edmonton closed a shift at 22:00 on December 1st GMT. We responsibly save this date in our backend as 05:00 December 2nd UTC. 
Now we decide to group by the date portion of the date in the back end. Their shift close gets lumped in with the sum for December 2nd. Your user is now very scared
and confused, and calls you on a Saturday to express their concerns about your skills as a developer.

So what can we do in this situation? One suggestion I heard was to save a second date in local time along with the UTC date. I can say from experience that this
gets unmanageable extremely quickly. And if your API serves multiple timezones you're going to be in for some very interesting queries.

So what can we do? Well I've come up with a solution that I think is pretty clean. We do need to break one barrier though.
The API needs to know what timezone the user wants their report. This hurts but it's a necessity. You can decide how you want
to handle this. The timezone can be sent along with the request or it can be put in the env variables if your API is only 
serving a single timezone at a time.

The solution is to project our dates into the requested timezone before we do our group. This took me a while to figure out 
with MongoTemplate, but you can do this by writing the following

```java 
  public ProjectionOperation projectDateAsFormat(ProjectionOperation operation, String format,
                                                 String field, String as, String timezone) {
    return operation.and(DateOperators.dateOf(field)
                                      .withTimezone(DateOperators.Timezone.valueOf(timezone))
                                      .toString(format)).as(as);
  }
```

I also wrote a quick and filthy addFields alternative for MongoTemplate, because it was a pain that it wasn't supported

```java 
  public static ProjectionOperation projectClass(Class clazz) {
    ProjectionOperation p = project();
    for (Field field : clazz.getDeclaredFields()) {
      p = p.and(field.getName()).as(field.getName());
    }
    return p;
  }
```

The object we are returning is also going to change a little, to look more like the following

```java 
@Data
@AllArgsConstructor
@NoArgsConstructor
public class ShiftPayment {
  
  private String date;
  private BigDecimal amountPaid;
  
}
```

And now we're ready to write our query

```java 
  public List<ShiftPayment> getShiftReportSummary() {
    Aggregation aggregation = Aggregation.newAggregation(match(Criteria.where("closedAt").exists(true)),
                                                         lookup("paymentEntry", "_id", "shiftId", "payments"),
                                                         projectDateAsFormat(
                                                             projectClass(Shift.class).and("payments").as("payments"),
                                                             "%Y-%m-%d",
                                                             "closedAt",
                                                             "date",
                                                             "America/Edmonton"),
                                                         unwind("payments"),
                                                         group("date")
                                                             .first("date").as("date")
                                                             .sum("payments.amount").as("amountPaid"));
    return this.mongoTemplate.aggregate(aggregation, "shift", ShiftPayment.class).getMappedResults();
  }
```

And bam! No more calls on Saturday, and no bothering with persisting pesky timezones. 

## Conclusion 

Today we looked at a few timezone applications and my approaches to them. Note that all of the code in this post 
is untested, but should give you some ideas about how you might handle timezones in your own applications.

Have fun, and happy coding!









