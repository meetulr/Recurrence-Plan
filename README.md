## Unserstanding The Creation and Management of Recurring Events

## Table of Contents:

1. [**What do we need?**](#what-do-we-need)

2. [**Solution**](#solution)
    - [Interfaces](#interfaces)
    - [Approach](#approach)
        - [Creating recurring events](#creating-recurring-events)
        - [Updating recurring events](#updating-recurring-events)
            - [Updating this instance only / updating an exception instance](#updating-this-instance-only--updating-an-exception-instance)
            - [Updating all instances / this and future instances](#updating-all-instances--this-and-future-instances)
        - [Deleting recurring events](#deleting-recurring-events)
            - [Deleting this instance only / deleting an exception instance](#deleting-this-instance-only--deleting-an-exception-instance)
            - [Deleting all instances / this and future instances](#deleting-all-instances--this-and-future-instances)
        - [Querying events](#querying-events)
        - [Handling exception instances](#handling-exception-instances)
        - [Historical Records](#historical-records)

3. [**References**](#references)
    - [rrule](#rrule)
    - [RecurrenceRule](#recurrencerule)
    - [BaseRecurringEvent](#baserecurringevent)
    - [Recurring Event Instance](#recurring-event-instance)
    - [Recurring Event Exception Instance](#recurring-event-exception-instance)


## What do we need?
1. The functionality to create recurring events.
2. The functionality to have custom recurrence patterns, like top calendar apps out there.
3. The functionality to update and delete:
    - a single instance.
    - this and following instances.
    - all instances.
      following a recurrence pattern.
5. A way to track the historical records of a recurring event.

## Solution:
### Interfaces
```javascript
  interface InterfaceEvent {
    // ...existing event fields
    isRecurringEventException: boolean;
    isBaseRecurringEvent: boolean;
    recurrenceRuleId: ObjectId;
    baseRecurringEventId: ObjectId;
  }

  interface InterfaceRecurrenceRule {
    // ...recurrence specific properties (frequency, count, etc.)
    recurrenceRuleString: string;
    baseRecurringEventId: ObjectId;
    startDate: Date;
    endDate: Date;
    latestInstanceDate: Date;
  }
```

The purpose and need for each of the fields and Interfaces will be explained in the approach as their necessity arises.

**I request going through this whole approach thoroughly first (to see if the edge cases you're thinking of are already covered in this solution or not), and then discuss if anything's been left out. Thank you**.

### Approach
1. We will use the rrule libary and follow the dynamic generation approach.

#### Creating recurring events
1. First, let's talk about the input:
    - The EventInput would have `recurring` field set to true, but about the `recurrence` field that we currently have, we should remove it and add an `rrule` field instead which would be a string.
    - Why? The recurrence field in the existing approach is an enum : `ONCE`, `WEEKLY`, `MONTHLY`, etc... But the rrule already has that in it's [`freq`](https://github.com/jkbrzt/rrule#rrule-constructor), and we want an exact pattern for event creation.
    - We want to have google calendar like functionality. So we should be implementing modals similar to that in the frontend to select recurrence patterns.
    - According to the values we select, and the boxes we check, we would create a recurrenceRuleObject that would be sent alongside the eventInput to the backend. There we would generate an rrule string based on that object.

2. After getting the input, we'd follow these steps:
    - Create a `RecurrenceRule` document that would contain the rrule string and the rrule fields for easy understanding and debugging, let's name this document's _id to be `recurranceRuleId`.
    - Create a `BaseRecurringEvent` that would just be like creating a normal event with `isBaseRecurringEvent: true`, let's name it's _id to be `baseRecurringEventId` (This is what we will use as the base event for generating instances.)
    - Both of these would contain the `startDate` and the `endDate` as provided in the input.
    - We would decide on an endDate, say `X` months or years ahead from the current date, that would decide the number of instances we create in the createEvent mutation, or during the query.
    - Get the dates for the instances to be generated using the `rrule`. Here:
        - If the event has no `endDate`, we'd generate events upto `X` date.
        - If the `endDate` is very far into the future, we'd generate events upto `X` date, and leave the rest for the query.
        - If there is an `endDate`, and the last date returned from the rrule is less than `X`, then we just create all the instances.
    - All the instances (Event documents) we created in the previous step will be based on the `BaseRecurringEvent` document we created above. All of them would have their *recurrenceRuleId* field set to `recurranceRuleId`, and the *BaseRecurringEventId* set to `BaseRecurringEventId` (from the first two steps).
    - Update the `latestInstanceDate` of the `RecurrenceRule` document created to be equal to the start date of the last instance we generated here.


#### Updating recurring events
1. #### *Updating this instance only / updating an `exception` instance:*
    - This would be straightforward, just make a regular update.

2. #### *Updating all instances / this and future instances:*
    For single events made recurring:
    - Get the data used to generate the instances, i.e. the current data of the event, and the latest data from the update input.
    - Follow the steps for creating a recurring event.
    - Delete the current event and its associations, as new ones would be generated while generating the instances.
    
    While updating a recurring event, we would base it on the `recurrenceRule`:
    - If the `recurrenceRule` has not changed:
        - then we can just make a regular update.
        - update the `BaseRecurringEvent` if required according the update inputs (which would be used as the new base event).
    - If the `recurrenceRule` has changed:
        - The `recurrenceRule` would be changed for the current and future events only, not the past events which have already occurred (changing their recurrence pattern wouldn't make sense, the first point would cover all the other event specific changes).
        - We would follow these steps:
            - Generate new instances based on the new `recurrenceRule`.
            - Delete every instance (current and the future) conforming to the old `recurrenceRule`. We can do this because we are generating events dynamically, i.e. we are only creating instances upto a certain date, so not many documents have to be deleted.
            - Find the latest instance that was following the old `recurrenceRule`, say `latestInstance`, and set the `latestInstanceDate` and the `endDate` of the old `recurrenceRule` to be the `latestInstance's` `startDate`.
            - Update the `BaseRecurringEvent` document if required to have values of the current update input (which would be used as the new base event).
            - Now, all the previous instances would have a different `RecurrenceRule` than the current and future ones. 

  What I'm suggesting here is that when the user changes the `recurrenceRule` and hits "save", this and the future instances will be affected.
  **Note here that we're not creating a new `BaseRecurringEvent` document, just updating the existing one.**

#### Deleting recurring events
1. #### *Deleting this instance only / deleting an exception instance:*
    - Make a regular deletion.

2. #### *Deleting all instances / this and future instances:*
    - For deleting all instances:
        - Delete all the recurring instances with the current `recurrenceRuleId` (like google calendar).
        - Update the `endDate` of the `baseRecurringEvent` accordingly.

    - For this and future instances:
        - Find the document that was created previously to the current document with the current `RecurrenceRule`, set the `latestInstanceDate` and the `endDate` of the `RecurrenceRule` to that instance's `startDate`. Update the `BaseRecurringEvent` accordingly (modifying the `endDate`).
        - Delete all the recurring instances with the same `recurrenceRuleId` as the current document, starting from the current date (The way google calendar does it).

#### Thing to keep in mind: Updates would only be done on the `BaseRecurringEvent` if bulk operations being are done on the instances with the latest `RecurringRule`, because we want to generate new instances based on the latest `recurrenceRule` that the already generated instances were following. How do we ensure that?
  - By adding a check, of `endDate`s. i.e. we would only modify the `BaseRecurringEvent` if its `endDate` matches that of the current `RecurrenceRule`.
  
#### Querying events

In the query, we would add a function for creating recurring event instances, and then query all the events and return them. Here's the two step process:
  - Generate recurring event instances. Here we'd follow these steps:
    - Fix a `queryUptoDate`.
    - Find all the `RecurrenceRule` documents with the `latestInstanceDate` less than `queryUptoDate`.
    - For every recurrenceRule document queried:
      - Find the `BaseRecurringEvent`.
      - Generate new recurring instance dates after `latestInstanceDate`.
      - Account for the number of existing instances following that recurrence pattern and how many more to generate based on the `RecurrenceRule`'s count (if specified).
      - Generate more instances based on the `BaseRecurringEvent`.
      - If more instances were created, update the `latestInstanceDate`.
  - Query events according to the inputs (`where` and `sort`) and return them.

#### Handling exception instances
  - With this approach, we don't have to worry about the single instances that have been updated/deleted, because the new instances are to be generated with `BaseRecurringEvent`.
  - However, if a bulk operation is made (changing rrule, or other event specific parameters), then every instance conforming to the current rrule is affected, even the ones that were edited seperately in single instance updates (their dates might have been changed, attendees list might be modified, etc.), because they still follow that rrule. i.e. the rrule wins in the end. Same with deletion, all the events conforming to an rrule are deleted on a bulk delete operation.
  - If we want to exclude a certain instance from such operations, we could add the `isRecurringEventException: true` for that instance. By doing that, we could make it completely independent (like a normal event), so that it won't be affected by the bulk operations. If we want it to conform to the rrule again, we could just set the `isRecurringEventException: false`.

#### Historical Records
  - `BaseRecurringEvent`, aside from being used as the base event to create new instances, also connects all the instances, even if their `rrule` are different. Which means we could also use it to track the historical records for a recurring event, accross all the instances, no matter what recurrence pattern it followed at any point.

## References
### rrule 

The library we're using that automatically generate the dates of recurrence for a `RecurrenceRule`. Official repo: [`rrule`](https://github.com/jkbrzt/rrule)

### RecurrenceRule

A schema containing the properties of that represents the recurrence rule followed by a recurring event. Currently it has three properties:
```javascript 
  interface InterfaceRecurrenceRule {
    recurrenceRuleString: string
    frequency: ["DAILY", "WEEKLY", "MONTHLY", "YEARLY"]
    weekdays: ["MONDAY", ... , "SUNDAY"]
    count: number
    baseRecurringEventId: ObjectId
  }
```
   - **recurrenceRuleString**: an `rrule` string that would be used to generate an `rrule` object, from which we would generate the recurrence dates.
   - **frequency**: Frequency of recurrence.
   - **weekDays**: The days of the week at which the instances would be scheduled.
   - **count**: The number of instances for that recurring event.
   - ...more recurrence specific properties could be added to this interface.
   - **baseRecurringEventId**: The `BaseRecurringEvent` for that recurring event.

### BaseRecurringEvent

  - A special type of event, that connects all the instances of a recurring event, even across different recurrence patterns, which is useful for tracking the historical records of a recurring event.
  - It is also used as the base event to generate new recurring event instances during queries. As we can't just use the latest instance, which could be an `exception` instance.
  There would be a flag in the event interface indicating whether it's a `BaseRecurringEvent`:
```javascript 
  interface InterfaceEvent {
    //...existing event fields
    isBaseRecurringEvent: true
  }
```

### Recurring Event Instance

Every instance of a recurring event would have these fields:
```javascript 
  interface InterfaceEvent {
    //...existing event fields
    isBaseRecurringEvent: false
    recurrenceRuleId: ObjectId
    baseRecurringEvent: ObjectId
  }
```
   - **isBaseRecurringEvent**: The instance itself would not be the base recurring event.
   - **recurrenceRuleId**: Representing the `RecurrenceRule` followed by the recurring event.
   - **baseRecurringEventId**: Representing the `BaseRecurringEvent` for that recurring event.

### Recurring Event Exception Instance

  - The bulk operations on a recurring event (`update`/`delete` multiple instances) would affect every instance following that `RecurrenceRule`.
  - If we want some instance to not be affected by those bulk operations, we could make it an `exception` to the `RecurrenceRule` rule it was following.
There would be a flag to mark an exception instance:
```javascript 
  interface InterfaceEvent {
    //...existing event fields
    isRecurringEventException: true
  }
```
   - With this flag, a recurring event instance could be like a single non-recurring event.
   - If we want it to conform to the recurrence rule again, we could update the `isRecurringEventException: false`.
