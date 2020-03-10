---
layout: post
title: But When do I Work?
---

It's often useful to represent and store defined periods of time. Today we're going to look at what happens when
those periods of time overlap. It may seem trivial but read to the end and you'll see that it's more interesting than it appears!

A repository with the full working API and all of the code discussed in this article can be found 
[here](https://github.com/lucasgauk/time-slots)


## The Collection

We're going to be setting up a basic Shift / Availability database using MongoDB. Shifts and availabilities are going 
to look identical, they have a start date, an end date, and can be assigned to an employee.

```java 
@Data
@AllArgsConstructor @NoArgsConstructor
public class TimeSlot {

  private LocalDateTime startDateTime;
  private LocalDateTime endDateTime;

}
```

```java 
@Document
@TypeAlias("Shift")
@Data @AllArgsConstructor @NoArgsConstructor
@EqualsAndHashCode(callSuper = true)
public class Shift extends TimeSlot {

  @Id private ObjectId id;
  private ObjectId employeeId;

}
```

```java 
@Document
@TypeAlias("Availability")
@Data @AllArgsConstructor @NoArgsConstructor
@EqualsAndHashCode(callSuper = true)
public class Availability extends TimeSlot {

  @Id private ObjectId id;
  @NotNull private ObjectId employeeId;

}
```

I've also added generic MongoDB repositories for both of these classes.

## When do I work?

So far so good, we can assign shifts and availabilities to our employees. But we will quickly realize that the 
interactions between these two defined time periods are important. We dont want to be able to assign a shift to an employee  
that has no availabilities, for example. 

We are going to have to define a way of finding and dealing with overlaps. To show you my solution, I'll implement a slightly 
contrived but more advanced use case: the number of employees available for a minimum amount of time on a specific day.

We will define a new method as the following:

```java 
Set<ObjectId> availableForDurationOnDay(Integer duration, LocalDate date);
```

Which will take a duration and a date and return a Set of employees that are available for at least the consecutive duration 
(in minutes) on that day (Again, a bit contrived but it allows us to explore the concept a bit more).

**In basic terms, we are going to implement a solution to the following user story**: 
`I need someone to work for an hour today. I don't care when the hour is. Who can I call?`

So let's get started!

## What is an overlap?

So what is an overlap? An overlap is when one time frame intersects with another time frame at any point. Let's define that a 
bit more formally. There are a few ways a Shift can overlap with an Availability.

```
1. Shift ends within availability
    shift           |----------------------------|
    availability                      |-------------------------|
```

```
2. Shift starts within availability 
    shift               |--------------------------------|
    availability    |--------------------|
```

```
3. Availability is within shift 
    shift               |--------------------------------|
    availability              |--------------------|
```

```
4. Shift is within availability
    shift               |-----------------------|
    availability    |---------------------------------------|
```

These are all of the ways that a shift can overlap an availability. For many use cases, checking whether any of these
cases is true will be enough to detect an overlap. But ours is a little bit more complex. 

Consider each case individually. What is the remaining duration of our availability in each case?

1. We have from the shift end to the availability end available.
2. We have from the start of the availability to the start of the shift available.
3. We have no time left.

Now the fourth case is the most interesting. In the fourth case we have both the start of the availability (before the shift)
and the end of the availability (after the shift) available.

## So whats the problem here?

Well so far we've only dealt with single availabilities and single shifts. What happens when an employee has multiple shifts?
We start to get overlaps that look like the following.

```
    shift               |------------|      |------|      |---------------|
    availability    |---------------------------------------|        |--------------|
```

How long is our employee available today? 

Well there may be multiple ways of solving this but my approach was to define any intersections between shifts and availabilities 
as "unavailable". If we look at the previous example, that would mean that  

This
```
    shift               |------------|      |------|      |---------------|
    availability    |---------------------------------------|        |--------------|
```

Would turn into this
```
    shift               |------------|      |------|      |---------------|
    availability    |---|            |------|      |------|               |-------|
```

You'll notice that with this definition, a single availability can split into multiple different smaller availabilities. 
If we can create a method to handle that transformation then we can handle a lot of different use cases, including our 
contrived `availableForDurationOnDay` problem.

Let's write a recursive function that uses everything we've discussed so far to accomplish this goal

```java 
  private List<Availability> splitIntoTimeAvailable(Availability availability, List<Shift> shifts) {
    List<Availability> splitAvailability = new ArrayList<>();
    boolean isOverlapped = false;

    LocalDateTime aStart = availability.getStartDateTime();
    LocalDateTime aEnd = availability.getEndDateTime();

    for (Shift shift : shifts) {
      LocalDateTime sStart = shift.getStartDateTime();
      LocalDateTime sEnd = shift.getEndDateTime();

      // Shift ends within availability, starts earlier
      if ((sStart.isBefore(aStart) || sStart.isEqual(aStart)) && sEnd.isAfter(aStart) && sEnd.isBefore(aEnd)) {
        splitAvailability.addAll(splitIntoTimeAvailable(availabilityCopy(availability, sEnd, aEnd), shifts));
        isOverlapped = true;
      }

      // Shift within availability
      if (sStart.isAfter(aStart) && sEnd.isBefore(aEnd)) {
        splitAvailability.addAll(splitIntoTimeAvailable(availabilityCopy(availability, sEnd, aEnd), shifts));
        splitAvailability.addAll(splitIntoTimeAvailable(availabilityCopy(availability, aStart, sStart), shifts));
        isOverlapped = true;
      }

      // Shift starts within shift, ends later
      if ((sEnd.isAfter(aEnd) || sEnd.isEqual(aEnd)) && sStart.isAfter(aStart) && sStart.isBefore(aEnd)) {
        splitAvailability.addAll(splitIntoTimeAvailable(availabilityCopy(availability, aStart, sStart), shifts));
        isOverlapped = true;
      }

      // Availability is within shift
      if ((sStart.isBefore(aStart) || sStart.isEqual(aStart)) && (sEnd.isAfter(aEnd) || sEnd.isEqual(aEnd))) {
        isOverlapped = true;
      }
    }

    if (!isOverlapped) {
      splitAvailability.add(availability);
    }

    return splitAvailability;
  }
```

The base case here is that there are no shifts that overlap our availability, in which case our single availability
is split into a List of one availability: itself. If not, we look at all of the shifts that were passed to the method 
(we're going to handle fetching related shifts elsewhere). If we have an overlap, we're going to take the time remaining 
(as discussed earlier for each overlap possibility) and see if we need to split it down any further.

When all is said and done we have a method that takes an availability and any shifts and splits it into smaller availabilities with no 
overlaps.

With our new method, we can define the following function

```java 
  private long consecutiveTimeAvailable(Availability availability) {
    LocalDateTime shiftStartDate = availability.getStartDateTime().toLocalDate().atStartOfDay();
    LocalDateTime shiftEndDate = availability.getEndDateTime().toLocalDate().plusDays(1).atStartOfDay();

    List<Shift> shifts = this.shiftRepository.findAllByEmployeeIdAndStartDateTimeBetween(availability.getEmployeeId(),
                                                                                         shiftStartDate,
                                                                                         shiftEndDate);

    List<Availability> splitAvailabilities = this.splitIntoTimeAvailable(availability, shifts);

    long maxTimeAvailable = 0;
    for (Availability splitAvailability : splitAvailabilities) {
      long timeAvailable = this.differenceInMinutes(splitAvailability.getStartDateTime(),
                                                    splitAvailability.getEndDateTime());
      if (timeAvailable > maxTimeAvailable) {
        maxTimeAvailable = timeAvailable;
      }
    }
    return maxTimeAvailable;
  }
```

Our new method uses the `splitIntoTimeAvailable` method we just created to figure out the longest consecutive span of 
free time in an availability (in minutes). This method also deals with fetching related shifts.

With that, we can finally implement our `availableForDurationOnDay` method as the following

```java 
  public Set<ObjectId> availableForDurationOnDay(Integer duration, LocalDate date) {
    List<AggregationOperation> operations = new ArrayList<>();

    Criteria dateCriteria =
        new Criteria().andOperator(Criteria.where("endDateTime")
                                           .gte(date.atStartOfDay().toInstant(ZoneOffset.UTC)),
                                   Criteria.where("startDateTime")
                                           .lte(date.atTime(23, 59, 59).toInstant(ZoneOffset.UTC)));

    operations.add(match(dateCriteria));

    Aggregation aggregation = newAggregation(operations);

    List<Availability> availabilities =
        mongoTemplate.aggregate(aggregation, Availability.class, Availability.class)
                     .getMappedResults()
                     .stream()
                     .filter(availability -> consecutiveTimeAvailable(availability) >= duration)
                     .collect(Collectors.toList());

    return availabilities.stream().map(Availability::getEmployeeId).collect(Collectors.toSet());
  }
```

This has some room for improvement, but all we're doing here is finding all availabilities on a given day, then 
returning the ones that have at least the `duration` available consecutively, then returning the lucky owners of those
availabilities.

## Tests

So let's see if we've answered our original user story. It's really important to write tests for any methods that use `mongoTemplate`
because we often use string accessors for our attributes, which means that if someone decides to rename something 
our entire method would break. If we have tests then we can let Jenkins shame them for failing a build 
before that ever happens (rain clouds of deep, deep shame).

Let's look one of our tests.

```java 
  @Test
  public void availableForDuration_threeShifts() {
    ObjectId employee1 = new ObjectId();

    Availability availabilityEmployee1 = this.createAvailability(employee1,
                                                                 LocalDate.of(2020, 1, 1),
                                                                 LocalTime.of(8, 0),
                                                                 LocalTime.of(18, 0));
    this.availabilityRepository.save(availabilityEmployee1);

    Shift shift1 = this.createShift(employee1,
                                    LocalDate.of(2020, 1, 1),
                                    LocalTime.of(8, 0),
                                    LocalTime.of(15, 15));
    this.shiftRepository.save(shift1);

    Shift shift2 = this.createShift(employee1,
                                    LocalDate.of(2020, 1, 1),
                                    LocalTime.of(15, 30),
                                    LocalTime.of(16, 0));
    this.shiftRepository.save(shift2);

    Shift shift3 = this.createShift(employee1,
                                    LocalDate.of(2020, 1, 1),
                                    LocalTime.of(16, 0),
                                    LocalTime.of(17, 30));
    this.shiftRepository.save(shift3);

    Set<ObjectId> employeeIds = this.availabilityTemplate.availableForDurationOnDay(30, LocalDate.of(2020, 1, 1));
    assertThat(employeeIds).contains(employee1);

    employeeIds = this.availabilityTemplate.availableForDurationOnDay(40, LocalDate.of(2020, 1, 1));
    assertThat(employeeIds.size()).isEqualTo(0);
  }
```

Here we define one availability for our charismatic `employee1` from 8 to 6. He then gets three shifts, from 8 - 3:15,
from 3:30 to 4, and from 4 to 5:30. 

Our employee has two slots of free time, from 3:15 to 3:30 and from 5:30 to 6. 

Our user wants to know who can work 30 minutes at any point on the first of January. Lucky for him, we can now tell him 
that his faithful employee, `employee1`, is available!  
 
## Wrap Up

Today we've looked at how we might implement a more complex user story regarding time definitions. This code is 
a little rough, but hopefully it gives you some hints as to how you might deal with time in your own applications.

Have fun and happy coding!

