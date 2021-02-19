---
title: Scheduling
layout: article
---

<div id="toc"></div>

A [Schedule](/#Schedule) describes when a user will be asked by the Bridge server to perform activities. A scheduled activity describes an activity (task or survey) and a start and end time when the user should be prompted to perform that activity. A client application may use this information to inform the user about their ongoing participation in the study (e.g. to show that a task is coming up in 2 hours, or that it is currently in progress).

The schedule API returns all activities within a given date range with the exception of deleted activities (to delete an activity, update it with a finishedOn timestamp, but no startedOn timestamp).

The API allows you to retrieve activities for up to two weeks given an arbitrary start and end timestamp (see the [scheduled activities API](/swagger-ui/index.html#/Activities/getScheduledActivitiesByDateRange)). You can use the date ranges to retrieve activities, one page at a time, for a longer time period.

Schedules are generated by a schedule plan (see below). A schedule plan describes a *strategy* for assigning schedules to participants. In the simplest case, every participant gets the same schedule (and thus the same scheduled activities); in more complex cases, researchers may assign users to different groups for A/B testing, or pursue other assignment strategies. A single schedule can be associated with more than one activity, if these activities should be grouped together. For example, you may have multiple activities in the A and B portion of an A/B test in order to co-vary the activities being assigned to a participant.

## Types of schedules

### Once

The simplest schedule generates an activity to be done one time, after an event occurs (usually enrollment). Once done, it will never be asked for again.

### Cron Schedules

A cron expression is a configuration string that describes times on a calendar, and allows for schedules such as "Mondays and Fridays at 8am" or "The first Thursday of every month at noon." The format of the cron expression is the seven field format as described in the [Java Quartz Scheduler](http://www.quartz-scheduler.org/documentation/quartz-2.3.0/tutorials/tutorial-lesson-06.html) documentation (note that there are other formats that take up to eleven fields, this [online cron expression generator](http://www.cronmaker.com/) creates expressions in the right format). Although cron-based schedules are not super-useful for scheduling study tasks, if you need this kind of schedule, cron expressions are a tested and successful way of describing them.

```json
{
   "scheduleType": "recurring",
   "cronTrigger": "0 0 6 ? * MON,WED,FRI *",
   "expires": "PT6H"
}
```
_Schedule this activity on Monday, Wednesday and Friday at 6am, and expire the activity 6 hours later._

### Time intervals from an event

Alternatively, activities can be scheduled periodically from a specific event (such as the completion of a prior activity, or the date and time that a user enrolled in a study). In this scheme, activities are implicitly scheduled against the participant's study enrollment date if no other event is specified. This approach allows you to schedule tasks such as "wait 2 weeks after enrollment and then schedule a task every 3 days thereafter" or "assign a task one hour after taking a medication."

Intervals are described using the [ISO 8601 Durations](https://en.wikipedia.org/wiki/ISO_8601#Durations) notation. Times of day are expressed in 24 hour time.

```json
{
    "scheduleType": "recurring",
    "interval": "P3D",
    "times": ["07:00", "15:00"],
    "expires": "PT6H"
}
```
_Schedule this activity every 3 days at 7am and 3pm in the time zone of the user. Expire each task after 6 hours._

### Sequenced activities

Recurring activities can also be restricted to a sequence (a limited period of time) by adding the `sequencePeriod` property to a schedule, with an ISO 8601 period value. This can be used to model crossover studies, using `delay` values in different schedules to generate "cool down" periods between sequences of activities. 

Like other scheduled activities, the sequence is scheduled from an initial event, like enrollment in the study. Sequences are inclusive of the start and exclusive of the end of the period. In the example given below, activies are scheduled on day 0, day 3, and day 6, but not day 9. This makes the math easier to calculate—you divide the `sequencePeriod` by the `interval` to get the number of days the activity will be scheduled.


```json
{
    "scheduleType": "recurring",
    "interval": "P3D",
    "times": ["07:00", "15:00"],
    "expires": "PT6H",
    "sequencePeriod": "P9D"
}
```
_Schedule this activity every 3 days at 7am and 3pm in the zone zone of the user, expiring each activity after six hours, and ending after 9 days, for a total of 6 activities, two on each of 3 days in the sequence._

### Persistent activities

This form of activity is always in the user's list of scheduled activities. When the client updates the activity to indicate it is finished, a new one is immediately added to the list of scheduled activities. This useful for tasks such as "Send Us Feedback" if you wish to control this from the server.

```json
{
    "scheduleType": "persistent"
}
```
_This activity will always appear in the participant's list of activities._

## Available Task Events

The following events are currently available for scheduling (set them as the `eventId` of the schedule):

|Event|Event ID structure|Description|
|---|---|---|
|Enrollment|enrollment|The time that the user signs the consent to participate in researcher. This event is the default for a schedule if no other eventId is provided or found.|
|Activity finished|activity:&lt;GUID&gt;:finished|The time a scheduled activity is finished. This can be used to create recurring schedules where a task is issued N days after the same task was last performed.|
|2 weeks before enrollment|two\_weeks\_before_enrollment|Start schedule two weeks before enrollment. |
|2 months before enrollment|two\_months\_before_enrollment|Start schedule two months before enrollment. |

## Task scheduling logic

When scheduling a task, the scheduler does the following:

* The scheduler selects a starting date and time. This would be the enrollment date by default;
* If a delay has been specified, it is added to that timestamp;
* If a cron trigger is defined, it then finds the next valid time using the cron expression, from that timestamp;
* Otherwise, it creates a task on that day for each of the times specified in the schedule. The scheduler then applies the interval to the last of these timestamps, and continues assigning on the next day.
* Once the start time for the task is determined, if the schedule has an expires duration, that is added to the start time to derive the end time of the task;
* If a startsOn or endsOn timestamp have been defined and the activity's window for performance falls outside of this window, then no tasks is created. Currently this is the only way to stop scheduling activities as of a specific date (in UTC time).
* The scheduler continues this way until it has calculated the activities that fall within the time window specified in the API call. 

_You must set an expiration duration on repeating tasks._ Otherwise unfinished activities would stack up on each other during periods of inactivity, and the user would be presented with all of these activities at once.

### More Examples

```json
{ 
   "scheduleType":"once",
   "delay":"P1W",
   "times":["08:00"],
   "expires":"PT24H",
   "activities":[...],
   "type":"Schedule"
}
```
_Schedule activity one week after enrollment at 8am, expire it after 24 hours._


```json
{   
    "scheduleType":"recurring",
    "delay":"P1M",
    "interval":"P1M",
    "times":["08:00"],
    "expires": "P2W",
    "activities":[...],
    "type":"Schedule"
}
```
_Schedule activity one month after enrollment, and every month thereafter, at 8am, expiring each activity after 2 weeks._

```json
{ 
    "scheduleType":"recurring",
    "interval":"P1W",
    "times":["06:00"],
    "expires": "P1W",
    "activities":[...],
    "type":"Schedule"
}
```
_Schedule activity at 6am on the day of enrollment, and every week thereafter, expirint each activity after 1 week._


```json
{ 
    "scheduleType":"recurring",
    "cronTrigger":"0 0 8 ? * MON,WED *",
    "expires":"PT4H",
    "activities":[...],
    "type":"Schedule"
}
```
_Schedule activity at 8am on Monday and Friday, expiring the activities after 4 hours._

```json
{ 
    "scheduleType":"once",
    "delay":"PT1H",
    "eventId":"medication:f26ffeea-8361-4863-91d1-4353dd6d6d23:finished",
    "expires":"PT1H",
    "activities":[...],
    "type":"Schedule"
}
```
_Schedule activity an hour after taking medication. Activity does not expire._

```json
{ 
    "scheduleType":"recurring",
    "delay":"P3D",
    "interval":"P7D",
    "times":["14:00"],
    "expires":"P1D",
    "activities":[...],
    "type":"Schedule"
}
```
_Schedule activity 3 days after enrollment at 2pm, and then every 7 days at 2pm thereafter, expiring activity after 1 day._

```json
{ 
    "scheduleType":"recurring",
    "delay":"P2D",
    "interval":"P2D",
    "times":["07:00","16:00"],
    "expires":"PT4H",
    "activities":[...],
    "type":"Schedule"
}
```
_Scheduled activities 2 days after enrollment at 7am and 4pm, expiring them after 4 hours, and then schedule them every other day._

```json
{ 
    "scheduleType":"recurring",
    "interval":"P1D",
    "times":["09:00"],
    "expires":"P1D",
    "endsOn":"2015-03-26T17:00:00.000Z",
    "activities":[...],
    "type":"Schedule"
}
```
_Schedule activity every day from enrollment, to March 26, 2015 at 5pm UTC, expiring each task after one day._

```json
{   
    "scheduleType":"once",
    "delay":"P7D",
    "eventId":"survey:248cff84-adbe-47a0-ae20-da86e8f6ef5c:finished,enrollment",
    "times":["09:00"],
    "activities":[...],
    "type":"Schedule"
} 
```
_Schedule activity at enrollment, and then each time the activity is finished, wait 7 days and schedule it again (assuming this schedule's activity is the survey referenced)._

## Schedule plans

When you create a schedule plan for users, you take one or more schedules and combine them with strategies for how to assign activities to participants using the schedules.

For example, you can divide study participants into a control and test group, and use different schedules to give them tasks with different frequencies.

Bridge has three strategies at the current time for creating schedule plans:

|Strategy|Description|
|---|---|
|SimpleScheduleStrategy|All users are scheduled with the same schedule, and get the same activities.|
|CriteriaScheduleStrategy|The most flexible strategy, where each schedule is evaluated sequentially against a set of [Criteria](/model-browser.html#Criteria) and the first criteria that match a participant's request, that schedule is used to schedule activities for the participant. To assure a default schedule for participants who do not match any other criteria, you can include a last [ScheduleCriteria](/model-browser.html#ScheduleCriteria) object at the end of the list with no defined criteria.|
|ABTestScheduleStrategy|Divide your participants by percentage into random groups, and use a different schedule for each group to assign activities. The combined set of groups should total 100%. After the initial assignment, new users joining the study will be randomly assigned to one of the groups in proportion to their percentage representation in the study.|

Further documentation can be found under the [schedules endpoint API documentation](/swagger-ui/index.html#/Schedules). 