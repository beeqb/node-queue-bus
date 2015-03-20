# node-resquebus

[![Nodei stats](https://nodei.co/npm/node-resquebus.png?downloads=true)](https://npmjs.org/package/node-resquebus)

[![Build Status](https://travis-ci.org/taskrabbit/node-resquebus.png?branch=master)](https://travis-ci.org/taskrabbit/node-resquebus)

## Acknowledgments
- [Resque-Bus in Ruby](https://github.com/taskrabbit/resque-bus)
- [Original Blog Post](http://tech.taskrabbit.com/blog/2013/09/28/resque-bus/)

## What?

ResqueBus is a plugin for [Resque](https://github.com/resque/resque) (we use [node-resque](https://github.com/taskrabbit/node-resque) for this project) which transforms Resque into a distributed message bus.  This allows more than one application to share workloads on the bus, and for events/jobs to fan out to more than one application.  Perhaps you want the `user_created` event to be consumed by the `analytics` and `email` applications... then ResqueBus is for you.

This application is a [port of the main ruby project](https://github.com/taskrabbit/resque-bus) to node.js.  This project is likley to be behind the Ruby project.  However, this package aims to be 100% compatible with the Ruby version, and can fill a driver, worker, or scheduler role within that same ecosystm (more information below).

## Ecosystem

![img](https://raw.github.com/taskrabbit/node-resquebus/master/doc/data_flow.jpg)

There are a few roles which are required to use the resquebus:

- Publisher
- Driver
- Scheduler
- Worker

The publishing application sends an event (`bus.publish()`).  This event is entered into an `incomming` resque queue.  The Driver then inspects `subscriptions` and sends the event over to queues for application which have registered interest in the event.  Then, finally, workers for those events consume the jobs in a normal resque way.  You can also delay a publication `bus.publishAt()`, and in that case, a Scheduler is required to coordinate.  To learn more about the worker, [visit the node-resque](https://github.com/taskrabbit/node-resque) project.

## Subscribing and Publishing.

We use the 'bus' object to both subscribe and publish events.  It should be passed your `connectionDetails` and optionally `jobs`

```javascript
var BusPrototype = require("node-resquebus").bus;
var bus = new BusPrototype({connection: connectionDetails}, jobs, function(){
  var appKey   = 'exampleApp';
  var priority = 'default';
  var job      = 'remoteEventAdd'
  var matcher  = { bus_event_type : /^.*add.*/ }
  bus.subscribe(appKey, priority, job, matcher, function(error, bus_queue){
    // ...
  });
});
```

Once your `bus` is connected, you can publish events.  You must always define 'bus_event_type' and then an optional hash of data:

```javascript
bus.publish('add', {
    a: 5,
    b: 10,
  });

bus.publishIn(1000, 'subtract', {
  a: 10,
  b: 5,
});

bus.publishAt((new Date.getTime() + 1000), 'multiply', {
  a: 10,
  b: 5,
});
```
### Methods:

- `bus.subscriptions(callback(error, subsciptions, count))`
- `bus.subscribe(callback(appKey, priority, job, matcher, callback(error, queue_name)))`
- `bus.unsubscribe(callback(appKey, priority, job, callback))`
- `bus.unsubscribeAdd(callback(appKey, callback))`
- `bus.publish(bus_event_type, args, callback(error, toRun))`
- `bus.publishAt(timestamp, bus_event_type, args, callback(error, toRun))`
- `bus.publishIn(delay, bus_event_type, args, callback(error, toRun))`

Keep in mind that if you use `publishAt` or `publishIn`, you will need to have a `Scheduler` running in your ecosystem.

### Matching and Special Keys

You can use a single string `{ bus_event_type: 'add' }` or regex `{ bus_event_type : /^.*add.*/ }` to match events in your subscription.  There are also special keys you can use:

- `bus_special_value_key`     (key is not null)
- `bus_special_value_blank`   (key exists but is blank (''))
- `bus_special_value_empty`   (key is null)
- `bus_special_value_nil`     (key is undefined)
- `bus_special_value_value`   (key is not null)
- `bus_special_value_present` (key is not null and has a length > 0)

For example, a matcher like `{ first_name: 'bus_special_value_present' }` would match all events where the field 'first_name' is present.

### Subsciption Notes

- Be sure to subscribe before you start working.
- Subscribing returns `error` and `worker_queue` to the `callback`.  You can then use `worker_queue` to tell your worker what queues to work.
  - When subscribing, `appKey` will always be lowercased, and all spaces will be replaced with `"_"`
  - You can subscribe any number of times with no adverse effects.  Feel free to do it from every server at boot.
- All times should be in JS micoseconds (unix timestamp * 1000).  This project will convert them to unix timesamps to match Ruby when needed.  

## Driver

Running a driver will take items from the incomming queue and fan them out to registered applications.  The driver inherits from a `node-resque` worker, and opperates the same way:

```javascript
var driver = new DriverPrototype({connection: connectionDetails}, jobs, function(){
  driver.workerCleanup(); // optional: cleanup any previous improperly shutdown workers
  driver.start();
});
```
You can optionally configure some additional options:

```javascript
options = {
  looping: true,
  timeout: 5000,
  name:    os.hostname() + ":" + process.pid
}
```

### Driver Notes

- Unlike the ruby driver, the node driver will cache the subscriptions for a few seconds.  This if configurable.
- When using a regexp matcher, we will attempt to [convert JS's RegExp to a ruby regular expression](https://github.com/taskrabbit/node-resquebus/blob/master/lib/sections/utils.js#L28-L84).  This conversion is less than perfect, and there are liley to be problems with more complex matchers.  Please let us know if you find something wrong and open a GitHub issue.

## Data

When an event is passed though the entire bus, metadata is added by both the publisher and the driver. You can overwrite any of these fields in your `attributes` hash.  A simple event like `bus.publish('add', {a: 5, b: 10})` will yeild:

```javascript
{
   "class":"remoteEventSubtract",
   "queue":"exampleapp_default",
   "args":{
      "bus_driven_at":1392251619,
      "bus_rider_queue":"exampleapp_default",
      "bus_rider_app_key":"exampleapp",
      "bus_rider_sub_key":"exampleapp_default_remoteEventSubtract",
      "bus_rider_class_name":"remoteEventSubtract",
      "bus_event_type":"subtract",
      "bus_created_at":1392251614,
      "bus_published_at":1392251614,
      "bus_id":"1392251614-6a3d3a00-9cde-4877-9e53-9c580a0caa07",
      "bus_app_hostname":"OpsimusPrime.local",
      "a":5,
      "b":10
   }
}
```

## Full Example

You can see a full example with workers, drivers, and schedulers in [/examples/example.js](https://github.com/taskrabbit/node-resquebus/blob/examples/example.js)