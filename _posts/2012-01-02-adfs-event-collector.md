---
layout: post
tags : [adfs, identity, reactive, event-log]
---
{% include JB/setup %}

The event collector was one of the features that represented a challenge for the team to implement. We need an event collector to gather all `Audit` event that every ADFS in the farm writes so that we can build a web application and have a nice UI to see **who has logged in? when? to what application? and what claims he/she/it has?**. We usually  write to the Windows event log but we rarely consume events from it. As a developer of this feature you would ask yourself:

* How to read from the event log? Push or pull model?
* There are many events written. How do we extract a pattern out of the stream of events?

### How to read from the event log? Push or pull model?

First thing was to understand how to read from the Event Log. Naturally as a developer you would think there is an API like `eventLog.GetLogs(source, eventId)`. There is something like that, but it's even more powerfull: you can use XPath. 

    EventLogQuery eventsQuery = new EventLogQuery("Application", PathType.LogName, "//System[EventID==500]");
    EventLogReader logReader = new EventLogReader(eventsQuery);

[MSDN doc](http://msdn.microsoft.com/en-us/library/bb671200.aspx)

This is the PULL model. Asking the Event Log things. The other model would be an event-based model (event log PUSHes event to the client).

    EventLogQuery subscriptionQuery = new EventLogQuery(
                "Application", PathType.LogName, "*[System/Level=2]");

    watcher = new EventLogWatcher(subscriptionQuery);
    watcher.EventRecordWritten +=
          new EventHandler<EventRecordWrittenEventArgs>(EventLogEventRead);


[MSDN doc](http://msdn.microsoft.com/en-us/library/bb671202.aspx)

#### What are the things to consider in each model

**Polling**

* It works on most of the Windows versions.
* It can query historical events
* If there is a stream of events that is big it might consume resources (memory mostly) and the query might take more time to execute
* Depending how fast the events are being queried and procesed there (poll time) there will be a gap between the event happenning and the user getting it
* Easier to administrate as it is usually a central component/service.

**Push (Events)**

* The events are transferred when they happen if the criteria is satisfied
* No big batches of data send over network
* No network traffic while there are no events that matches
* Less resources consumption, especially when there is a big amount of events
* It is only supported from Vista / Windows 2008 and above

We've chosen the push model based on events. Essentially because it's more scalable and clean. The API even has a way to specify a bookmark to start reading from so you could get events from the past.

### How to extract a pattern out of the stream of events from the event log?

The other thing we had to solve is how to extract the events that we are interested in. ADFS generates different events: 299, 500, 501, 502. For each succesful loin it will generate:

* Two 299 containing a correlation id and the target (identity provider or relying party)
* Two 500 containing the claims of the caller identity for the identity provider or relying party 
* Two 501 containig the claims of the identity that runs the service
* Other that don't really matter.

You could get a stream of events like this:

    Event ID   Correlation ID
    299        1AD...                             X1
    500        1AD...                             X2
    500        1AD...
    500        1AD...
    299        43B...                             Y1
    299        EDB...                             Z1
    500        43B...                             Y2
    500        EDB...                             Z2
    500        43B...
    500        43B...

In general events might come in order (299 and then 500) but in a high concurrency situation I foresee two 299 events coming together and then the 500 events so I don't want to rely on the order the events come. The correlation id is the key to correlate them (which is the first property of the event `eventRecord.Properties[0]`)

Something like this

    { 1AD.., <X1, X2> }
    { 43B.., <Y1, Y2> }
    { EDB.., <Z1, Z2> }

### Using Reactive Extensions to extract a pattern out of an event stream

After doing [the right question in StackOverflow](http://stackoverflow.com/questions/10106622/processing-events-from-event-log-and-react-on-a-certain-pattern-rx) we had a starting point.

Reactive Extensions represents a stram of evens as an `IObservable` that can be "queried" using Linq to RX. Here is the query that solves the pattern:

     var pairs = events
            .Where(e299 => e299.EventArgs.EventRecord.Id == 299)
            .SelectMany(e299 => events.Where(e500 => e500.EventArgs.EventRecord.Id == 500 &&
                                                    e299.EventArgs.EventRecord.Properties[0].Value.ToString() ==
                                                    e500.EventArgs.EventRecord.Properties[0].Value.ToString())
                                    .Take(1), 
                        (e299, e500) => new { First = e299, Second = e500 });

Here is the output of a small program that uses the `EventLogWatcher` and Reactive Extensions to extract what we want from the event log. This is running on an ADFS server while poeple is logging in to applications

<img src="http://swmarkdownr.blob.core.windows.net/images/1877864182.png" width="650">

The source code for a Console App with the spike is here: <https://gist.github.com/2370950>

