## Table of Contents:

1. [**Issue**](#issue)
    - [What do we have?](#what-do-we-have)
    - [What do we need?](#what-do-we-need)

2. [**Solution**](#solution)
    - [Interfaces](#interfaces)
    - [Approach](#approach)
        - [Creating recurring events](#creating-recurring-events)
        - [Updating recurring events](#updating-recurring-events)
            - [Updating this instance only](#updating-this-instance-only)
            - [Updating all instances / this and future instances](#updating-all-instances--this-and-future-instances)
        - [Deleting recurring events](#deleting-recurring-events)
            - [Deleting this instance only](#deleting-this-instance-only)
            - [Deleting all instances / this and future instances](#deleting-all-instances--this-and-future-instances)
        - [Thing to keep in mind](#thing-to-keep-in-mind)
        - [Querying events](#querying-events)
        - [Handling exception instances](#handling-exception-instances)
        - [Historical References](#historical-references)


## Issue:
### *What do we have?*
1. We have the functionalities to deal with single non-recurring events.

### *What do we need?*
1. We need the ability to create recurring events.
2. We need the functionality to implement custom recurring patterns, like top calendar apps out there.
3. We need the ability to update and delete:
    - a single instance of the recurring pattern
    - this and the future instances
    - all instances.
5. We need a way to track the historical record for an event.

## Solution:
### *Interfaces*
![RecurringEvents](https://github.com/PalisadoesFoundation/talawa-api/assets/55585268/f752edd7-363c-412a-a50e-5069b8c602d6)


The purpose and need for each of the fields and schemas will be explained in the approach as their necessity arises.

**I request going through this whole approach thoroughly first (to see if the edge cases you're thinking of are already covered in this solution or not), and then discuss if anything's been left out. Thank you**.

### *Approach*
1. We will use the rrule libary and follow the dynamic generation approach.

#### *Creating recurring events*
1. First, let's talk about the input:
    - The EventInput would have `recurring` field set to true, but about the `recurrence` field that we currently have, we should remove it and add an `rrule` field instead which would be a string.
    - Why? The recurrence field in the existing approach is an enum : `ONCE`, `WEEKLY`, `MONTHLY`, etc... But the rrule already has that in it's [`freq`](https://github.com/jkbrzt/rrule#rrule-constructor), and we want an exact pattern for event creation.
    -  We want to have google calendar like functionality. So we should be implementing modals similar to that in the frontend to select recurring patterns.
    - We should use `rrule` in the frontend too, which, according to the values we select, and the boxes we check, would easily create an rrule string for us in the frontend and we could send it to the backend in the input.

2. After getting the input, we'd follow these steps:
    - Create a `RecurrenceRule` document that would contain the rrule sent from the frontend, let's name this document's _id to be `recurranceRuleId`.
    - Create a `TrueRecurringEvent` that would just be like creating a normal event, let's name it's _id to be `trueRecurringEventId` (This is what we will use as the base event for generating instances.)
    - Both of these would contain the `startDate` and the `endDate` as provided in the input.
    - We would decide on an endDate, say `X` months or years ahead from the current date, that would decide the number of instances we create in the createEvent mutation, or during the query.
    - Get the dates for the instances to be generated using the `rrule`. Here:
        - If the event has no `endDate`, we'd generate events upto `X` date.
        - If the `endDate` is very far into the future, we'd generate events upto `X` date, and leave the rest for the query.
        - If there is an `endDate`, and the last date returned from the rrule is less than `X`, then we just create all the instances.
    - All the instances (Event documents) we created in the previous step will be based on the `TrueRecurringEvent` document we created above. All of them would have their *recurrenceRuleId* field set to `recurranceRuleId`, and the *trueRecurringEventId* set to `trueRecurringEventId` (from the first two steps).
    - Update the `latestInstanceDate` of the `RecurrenceRule` document created to be equal to the start date of the last instance we generated here.


#### *Updating recurring events*
1. ##### Updating this instance only:
    - This would be straightforward, just make a regular update.

2. ##### Updating all instances / this and future instances:

    We would base it on the rrule:
    - If the rrule has not changed:
        - then we can just make a regular update.
        - update the `TrueRecurringEvent` according the update inputs (which would be used as the new base event).
    - If the rrule has changed:
        - Now correct me if i'm wrong here, but the user would change the rrule for current and future events only, not the past events that has already occured (changing their recurrence pattern wouldn't make sense, any changes other than rrule that are made on them would be covered by the first point).
        - If so, we would follow these steps:
            - Find the last instance that was generated before the current instance, say `latestInstance`, and set the `latestInstanceDate` and the `endDate` of the `recurringRule` for the current pattern to be the `latestInstance's` `startDate`.
            - Delete every instance (current and the future) conforming to the current rrule. We can do this because we are generating events dynamically, i.e. we are only creating instances upto a certain date, so not many documents have to be deleted.
            - Update the `TrueRecurringEvent` document to have values of the current update input (which would be used as the new base event).
            - Create a new `RecurrenceRule` document with the new `rrule` from the input. Set it's `startDate` as the current date, and `endDate` as the `TrueRecurringEvent's` `endDate`. Now, follow the same steps that were used in creating events in createEvent mutation to generate instances.
            - Now, all the previous instances would have a different `RecurrenceRule` than the current and future ones. 

  What I'm suggesting here is that when the user changes the rrule and hits "save", this and the future instances will be affected.
  *Note here that we're not creating a new TrueRecurringEvent document, just updating the existing one.*

#### *Deleting recurring events*
1. ##### Deleting this instance only:
    - Make a regular deletion.

2. ##### Deleting all instances / this and future instances:
    - For deleting all instances:
        - Delete all the recurring instances with the current `trueRecurringEventId`. (So you see, `TrueRecurringEvent`, aside from being used as the base event to create new instances, also connects all the instances, even if their `rrule` are different. Which means we could also use it to track the historical record of a recurring event, accross all the instances, no matter what recurrence pattern it followed at any point).

    - For this and future instances:
        - Find the document that was created previously to the current document with the current `RecurrenceRule`, set the `latestInstanceDate` and the `endDate` of the `RecurrenceRule` to that instance's `startDate`. Update the `TrueRecurringEvent` accordingly (modifying the `endDate`).
        - Delete all the recurring instances with the same `recurringRuleId` as the current document, starting from the current date (The way google calendar does it).

##### Thing to keep in mind: Updates would only be done on the `TrueRecurringEvent` if bulk operations being are done on the instances with the latest `RecurringRule`, because we want to generate new instances based on the latest `rrule` that the already generated instances were following. How do we ensure that?
  - By adding a check, of `endDate`s. i.e. we would only modify the `TrueRecurringEvent` if its `endDate` matches that of the current `RecurrenceRule`.
  
#### *Querying events*

Currently we're just querying all the events belonging to an organization. We would need significant modifications in the query or make a seperate query for this.
These are the steps we'd follow:
  - Find all the events upto the date `X`(what we'd decided on earlier) ahead of the current date.
  - Find all the `RecurrenceRule` documents with the `latestInstanceDate` less than `X` && `latestInstanceDate` not equaling the `endDate`.
  - Find the `TrueRecurringEvent`s for those.
  - Let's say the `endDate` for the current `RecurrenceRule` be `rRuleEndDate`.
  - Generate the recurring instances starting from `latestInstanceDate` uptill `min(X, rRuleEndDate)` based on the `TrueRecurringEvent`.
  - Update the `latestInstanceDate` of the `RecurrenceRule` documents.
  - Return all the documents we fetched/generated here.

#### *Handling exception instances*
1. With this approach, we don't have to worry about the single instances that have been updated/deleted, because the new instances are to be generated with `TrueRecurringEvent`.
2. However, if a bulk operation is made (changing rrule, or other event specific parameters), then every instance conforming to the current rrule is affected, even the ones that were edited seperately in single instance updates (their dates might have been changed, attendees list might be modified, etc.), because they still follow that rrule. i.e. the rrule wins in the end. Same with deletion, all the events conforming to an rrule are deleted on a bulk delete operation.
3. If we want to exclude a certain instance from such operations, we could add the `isException: true` for that instance. By doing that, we could make it completely independent (like a normal event), so that it won't be affected by the bulk operations. If we want it to conform to the rrule again, we could just set the `isException: false`.

#### *Historical References*
As mentioned earlier, `TrueRecurringEvent` will take care of that.
