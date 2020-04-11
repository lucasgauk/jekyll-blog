---
layout: post
title: Time Slots and Set Theory
---

Time slots are defined periods of time with a start date and an end date. 
There are many situations for which we will need to define a section of time in code and, surprisingly, 
not many tools to do it. Today we will be looking at my implementation of a TimeSlot in Java and doing a 
teeny tiny bit of set theory in the process. At the end we'll look at how we might use this implementation to solve
some common time slot related interview questions.

A repository with all of the code discussed in this article can be found 
[here](https://github.com/lucasgauk/time-slot).

## A Basic Definition
A time slot is a period of time that starts at a given moment and ends at a given moment. My implementation will 
use LocalDateTime to mark the start and the end of the time slot.

```java 
public final class TimeSlot {

  /**
   * The start of the time slot.
   */
  private final LocalDateTime start;

  /**
   * The end of the time slot.
   */
  private final LocalDateTime end;
```

It is assumed that all time between the start of the time slot and the end of the time slot is *occupied* by the time slot.

A time slot can *contain* a specific moment in time. A time slot contains a moment in time if that moment occurs after
the time slot has started and before it has ended. In the case of time slots, Something that occurs at *exactly* the start
or the end of the time slot is not actually *contained* by the time slot. More on this later.

```java 
  public boolean contains(LocalDateTime dateTime) {
    return dateTime.isAfter(this.start) && dateTime.isBefore(this.end);
  }
```

In mathematics, the set of real numbers is a set that contains all rational and irrational numbers. Values of this set
can be represented as points on a infinitely long connected line. You can define a subset of rational numbers that
contains a specific set of rational numbers on this line. A is a *subset* of B if all members of A are also members of B.

It is useful to describe time as another infinitely long connected line, containing any given moment in time as points 
along it. Any point in time can be drawn as a point on this line.

``` 
------------------------------------------------------------ time ---> 
          |                                     | 
 1902-01-14T00:14:22                   2014-07-22T14:56:44
```

We can then say that a time slot is effectively a *subset* of this set. We can draw these subsets on the 
infinite line by marking the start and the end of the time slot on the line and considering that every value on the line
between the start and the end of the time slot is part of the time slot's set. 

For example we might draw a time slot as the following.

``` 
------------------------------------------------------------ time ---> 
                     |------------------------| 
            2020-01-02T15:00:00      2020-01-02T17:30:00
```

Any point in time between the start date and the end date of the time slot can be drawn as a point on both lines,
and any point in time before or after does not belong to the *set* defined by the time slot.

Finally, in set theory a *disconnected* set is a set that is made up of the union of multiple disjoint sets.
You'll notice that our time slots are *connected* sets. 

## Overlapping
Time slots can overlap each other. Continuing with our set mumbo jumbo we can say that a time slot A overlaps
time slot B if there are moments in time that are contained by both A and B. More formally, we can say that
if the *intersection* of A and B is not an empty set, our time slots have overlapped each other.

Let's look at all the cases where A might intersect B. Assume in each example that as before, these subsets are drawn
against the continuous (connected) set of time.

1. Set A equals set B.
2. Set A contains set B.
``` 
A   |-------------------------------------|
B        [------------------]
```
3. Set B contains set A.
4. Set A starts before set B, and ends within it.
``` 
A   |-----------------------|
B            |--------------------------------|
```
5. Set A starts within set B, and ends after it. 
``` 
A                     |-----------------------|
B   |--------------------------------|
```

Let's define these possibilities in code.

```java 
  private enum OverlapPossibility {
    EQUALS, // Starts and ends at the same time as other
    CONTAINS, // Other is completely within this
    IS_CONTAINED, // This is completely within other
    STARTS_BEFORE_ENDS_WITHIN, // This starts before other and ends within it
    STARTS_WITHIN_ENDS_AFTER, // This starts within other and ends after it
    NO_OVERLAP // This does not overlap other
  }
```

We can then define a method that returns the OverlapPossibility that occurs when comparing two time slots.

```java 
  private OverlapPossibility checkOverlap(TimeSlot other) {
    if (this.equals(other)) {
      return OverlapPossibility.EQUALS;
    }
    if (this.contains(other)) {
      return OverlapPossibility.CONTAINS;
    }
    if (other.contains(this)) {
      return OverlapPossibility.IS_CONTAINED;
    }
    if (this.startsWithinEndsAfter(other)) {
      return OverlapPossibility.STARTS_WITHIN_ENDS_AFTER;
    }
    if (this.startsBeforeEndsWithin(other)) {
      return OverlapPossibility.STARTS_BEFORE_ENDS_WITHIN;
    }
    return OverlapPossibility.NO_OVERLAP;
  }
```

Then we can define the following.

```java 
  public boolean overlaps(TimeSlot other) {
    return !this.checkOverlap(other).equals(OverlapPossibility.NO_OVERLAP);
  }
```

## Addition
The addition of two time slots A and B is the *union* of A and B. Or more exactly, the set of all possible
members in either A *or* B. The time slots might be *disjoint* meaning they share no common time. In which case, the 
resulting set would be non connected, but still include all members of A and B. 

For example:

A + B
``` 
A          |------------------|
B                |------------------|
Union      |------------------------|
``` 

A + B
``` 
A       |--------------|
B                          |----------------|
Union   |--------------|   |----------------|
```

A + B + C
``` 
A       |---------| 
B                     |---------|
C                          |-------------|
Union   |---------|   |------------------|
```

As you can see, this gets more complicated with more sets. It isn't perfect, but we might implement that as the following.

```java 
  private List<TimeSlot> overlapsWith(List<TimeSlot> others) {
    return others.stream().filter(other -> other.overlaps(this)).collect(Collectors.toList());
  }

  private TimeSlot combineWith(TimeSlot other) {
    switch (this.checkOverlap(other)) {
      case IS_CONTAINED:
        return other;
      case STARTS_BEFORE_ENDS_WITHIN:
        return TimeSlot.of(this.start, other.end);
      case STARTS_WITHIN_ENDS_AFTER:
        return TimeSlot.of(other.start, this.end);
      default:
        return this;
    }
  }

  public List<TimeSlot> add(List<TimeSlot> others) {
    List<TimeSlot> results = new ArrayList<>();
    results.add(this);
    for (TimeSlot other : others) {
      for (TimeSlot overlap : other.overlapsWith(results)) {
        other = other.combineWith(overlap);
        results.remove(overlap);
      }
      results.add(other);
    }
    return results;
  }
```

When we are adding time slots together there are a few possibilities. If the time slot doesn't overlap
any of the other time slots we are adding together then we can leave that time slot alone and add it to the resulting set.
If it does overlap other time slots then we have to combine those time slots together into one connected time slot before
adding it to the set.

## Subtraction
The *complement* of a set A is the set of all values that aren't contained by A. Subtracting time slot A from time slot 
B will result in all members of A that were *not* members of B. In formal terms, A - B is the intersection between
A and B's complement.

For example:

A - B
``` 
A              |------------------|
B                     |------------------|
Subtraction    |------|
``` 

A - B
``` 
A             |--------------|
B                                |----------------|
Subtraction   |--------------|
```

A - B - C
``` 
A            |----------------------------------------| 
B                     |---------|
C                                             |----|
Subtraction   |-------|         |-------------|    |--|
```

Again, more complicated with more sets. There's room for improvement, but an implementation of this might look like the following.

```java 
  public List<TimeSlot> subtract(List<TimeSlot> others) {
    List<TimeSlot> results = new ArrayList<>();
    List<TimeSlot> overlaps = this.overlapsWith(others);
    if (overlaps.isEmpty()) {
      results.add(this);
    } else {
      TimeSlot firstOverlap = overlaps.remove(0);
      switch (this.checkOverlap(firstOverlap)) {
        case CONTAINS:
          results.addAll(TimeSlot.of(this.start, firstOverlap.start).subtract(overlaps));
          results.addAll(TimeSlot.of(this.end, firstOverlap.end).subtract(overlaps));
          break;
        case STARTS_BEFORE_ENDS_WITHIN:
          results.addAll(TimeSlot.of(this.start, firstOverlap.start).subtract(overlaps));
          break;
        case STARTS_WITHIN_ENDS_AFTER:
          results.addAll(TimeSlot.of(firstOverlap.end, this.end).subtract(overlaps));
          break;
      }
    }
    return results;
  }
```

First we check which of the others overlap this. We take the first and subtract it from this. Then from the resulting 
piece (or pieces) we subtract each of the remaining others recursively. 

## Intersection
As we discussed earlier, the intersection of time slots A and B is the time slot C that contains all shared elements of A
and B. In my implementation, if C contains nothing then C will be null, it will no longer exist as a time slot.

At this point I don't think we need any more examples. Let's see how we might implement a method that returns the 
intersection between a number of sets.

```java 
  public TimeSlot intersect(List<TimeSlot> others) {
    if (others.size() == 0) {
      return null;
    }
    return intersectHelper(new ArrayList<>(others));
  }

  private TimeSlot intersectHelper(List<TimeSlot> others) {
    if (others.size() == 0) {
      return this;
    }
    TimeSlot firstOverlap = others.remove(0);
    switch (this.checkOverlap(firstOverlap)) {
      case STARTS_BEFORE_ENDS_WITHIN:
        return TimeSlot.of(firstOverlap.start, this.end).intersectHelper(others);
      case STARTS_WITHIN_ENDS_AFTER:
        return TimeSlot.of(this.start, firstOverlap.end).intersectHelper(others);
      case CONTAINS:
        return firstOverlap.intersectHelper(others);
      case NO_OVERLAP:
        return null;
      default:
        return this.intersectHelper(others);
    }
  }
```

This case is easier than the others. We check the first intersection, and then if there is still time left after this
we recursively check to see the intersection between this remaining time and the other time slots.

## Disjoint Sets
You may have noticed one limitation of our time slot implementation so far. A time slot can only represent a 
*connected* set of time. If you look back at our implementations of subtract and add you'll notice we're often left
with a *list* of time slots. Each time slot in the resulting list is part of a greater *set* of time that represents
the result of the operation. The sets that make up this result form a *non connected* set of time. 

This concept is immensely useful when it comes to representing normal human events. You would always consider someone's
availability to be a set of time even though there may be gaps within the schedule that the person cannot work.

You will often want to perform similar set operations on these sets of time. For example, to find when two people 
are available at the same time you will likely want to check the intersection of two time sets. That intersection 
will also likely be a non continuous set.

To solve this problem, I've implemented the concept of a TimeSet. A time set contains a list of time slots.

```java 
public class TimeSet {

  /**
   * List of continuous sets of time that make up a time set.
   */
  private List<TimeSlot> timeSlots;
```

The time slots in the time set may be disjoint, but may not be overlapping. Overlapping time slots will be combined
together in the set.

```java 
  private TimeSet(List<TimeSlot> timeSlots) {
    this.timeSlots = new ArrayList<>();
    timeSlots.forEach(this::add);
  }

  ... 

  public void add(TimeSlot timeSlot) {
    this.timeSlots = timeSlot.add(this.timeSlots);
  }
```

Time sets support many of the same operations that can be performed on a single time slot, and at this point 
should hopefully be fairly self explanatory.

For example, adding time sets together will result in the *union* of all time sets, or rather a non continuous
set of time that contains all the time in every set.

```java 
  public void add(List<TimeSet> others) {
    List<TimeSlot> otherTimeSlots = new ArrayList<>();
    others.forEach(other -> otherTimeSlots.addAll(other.timeSlots));
    otherTimeSlots.forEach(this::add);
  }
```

Subtracting sets from a set will result in a set containing all time in the original set that was not contained
in any of the other sets. Or rather, the union of that set with the complement of the union of the sets you are subtracting.

```java 
  public void subtract(TimeSlot other) {
    List<TimeSlot> subtractionResults = new ArrayList<>();
    this.timeSlots.forEach(timeSlot -> subtractionResults.addAll(timeSlot.subtract(other)));
    this.timeSlots = subtractionResults;
  }

  public void subtract(List<TimeSet> others) {
    List<TimeSlot> otherTimeSlots = new ArrayList<>();
    others.forEach(other -> otherTimeSlots.addAll(other.timeSlots));
    otherTimeSlots.forEach(this::subtract);
  }
```

Finally, the most interesting and complex case is the intersection between sets. The intersection will return
a time set that contains all time that is shared between all sets. This is a little more complex to implement but 
my attempt looks a little like the following.

```java 
  public void intersect(List<TimeSet> others) {
    if (others.size() == 0) {
      this.timeSlots = new ArrayList<>();
    }
    this.intersectHelper(new ArrayList<>(others));
  }

  private void intersectHelper(List<TimeSet> others) {
    if (others.size() == 0) {
      return;
    }
    TimeSet firstSet = others.remove(0);
    List<TimeSlot> intersections = new ArrayList<>();
    this.timeSlots.forEach(timeSlot -> {
      List<TimeSlot> overlaps = timeSlot.overlapsWith(firstSet.timeSlots);
      if (overlaps.size() == 0) {
        return;
      }
      overlaps.forEach(overlap -> intersections.add(timeSlot.intersect(overlap)));
    });
    this.timeSlots = intersections;
    this.intersectHelper(others);
  }
```

Lets walk through an example to try to explain the intersect method. Say we had three time sets that looked like the
following.

``` 
A   |--------|         |-----------------------------| |----| 
B        |------------------|          |------|        
C          |-------------------------------|
```

We're performing A.intersect(B, C). Following the algorithm, we are starting by comparing A and B. Rs is the result.

``` 
A   |--------|         |-----------------------------| |----| 
B        |------------------|          |------|
Rs 
```

We check the first time slot of set A. We're looking to see if that time slot overlaps anything in time set B.

``` 
A   |--------|
B        |------------------|          |------|
Rs
```

It overlaps time slot 1 of set B. Therefore we will add the intersect between the two time slots to our result.
We then check time slot 2 of A. It overlaps time slot 1 and 2 of B. We add the intersection to the result. We check
3 of A. It overlaps nothing, therefore there is no way it will be included in the intersection. We discard it. 
After comparing A and B our result looks as follows.

``` 
A   |--------|         |-----------------------------| |----| 
B        |------------------|          |------|
Rs       |---|         |----|          |------|
```

Now we compare the Rs time set to C in the same way. The result of that comparison is our intersection. If there
were more time sets, we would compare the result of Rs intersect C with the next, then onwards. 

The final result looks as follows.

``` 
A   |--------|         |-----------------------------| |----| 
B        |------------------|          |------|        
C          |-------------------------------|
Rs         |-|         |----|          |---| 
```

## Application
With these methods established you should be able to accomplish nearly any basic time slot use case. Let's have a look
at a few common user stories and see how we might tackle them with our new time slot and time set classes.

```
Three employees have a set of availabilities. When are all three of them available to meet for at least 30 minutes?
```

First its worthwhile to note that you should avoid storing time sets in the database. Time sets should be used as a 
utility class to wrap time slots together before performing operations.

So in this case we would have each employees available time slots saved in the database. Implementing this solution
might look something like the following.

```java
  TimeSet e1 = TimeSet.of(e1TimeSlots);
  TimeSet e2 = TimeSet.of(e2TimeSlots);
  TimeSet e3 = TimeSet.of(e3TimeSlots);

  return TimeSet.intersection(Arrays.asList(e1, e2, e3)).getTimeSlots().stream()
                .filter(timeSlot -> timeSlot.length(ChronoUnit.MINUTES) >= 30)
                .collect(Collectors.toList());
```

Here we're finding the intersect of three time sets made up of employee availabilities. Of those, we're finding the
ones that are at least 30 minutes long. Let's look at another.

``` 
An employee has a set of availabilities and a set of shifts. See if he still has 1 hour available to work at 
some point on a certain day (2020-01-01).
```

```java 
  TimeSet availabilities = TimeSet.of(availabilities);
  TimeSet shifts = TimeSet.of(shifts);

  availabilities.subtract(shifts);

  availabilities.intersect(TimeSlot.of(LocalDate.of(2020, 1, 1)));

  return availabilities.getTimeSlots().stream()
                       .anyMatch(availability -> availability.length(ChronoUnit.HOURS) >= 1);
```

Here we're subtracting the shift set from the availability set to see what time slots are still available. 
After we're finding the intersection of those availabilities and a single day, and seeing if there's any 
time slot more than an hour in length.

## Wrap Up

Today we've had a look a time slots and time sets. We've seen how we might facilitate communication by describing them as sets,
and we've seen some implementations of very handy operations that we might leverage to manage time slots in our 
applications.

This code is all fairly rough and unoptimized, but hopefully it's given you an idea about how you might manage time 
slots in your own applications.

Have fun, and happy coding!
