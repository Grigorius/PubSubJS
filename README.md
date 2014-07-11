# PubSubJS
Port of PubSub to Angular - so you can inject it as a factory:

## Installation

Add the module to your dependencies

```javascript
angular.module('myApp', ['gr.PubSub', ...])
```

And inject the service into anoter service or controller or directive ...
The important bit is using 
```javascript $scope.$on('$destroy', function () { PubSub.unsubscribe(token); }); ```

````html
<body ng-app="YOUR_APP" ng-controller="MainCtrl">
</body>
<script src="http://ajax.googleapis.com/ajax/libs/angularjs/1.2.10/angular.js"></script>
<script src=".../pubsub.js"></script>
<script>
  angular.module('YOUR_APP', [
    'gr.PubSub',
  ]);
  angular.module('controllers', [])
    .controller('MainCtrl', ['$scope', 'PubSub', function($scope, PubSub) {

        var someMessageHandler =  function (event, queue) {
            // ...
        };

        var someMessageToken = PubSub.subscribe('someMessage', messageHandler);

        $scope.$on('$destroy', function () {
            PubSub.unsubscribe(someMessageToken);
        });

    }])
    .factory('someFactory', ['PubSub', function (PubSub) {
        return {
            notifyEvent: function (data) {
                PubSub.publish('someMessage', data);
            },
            processEvent: function (someEvent) {
                //process event
                this.notifyEvent(processedData);
            }
        }
    }]);
</script>

````

PubSubJS is a [topic-based](http://en.wikipedia.org/wiki/Publish–subscribe_pattern#Message_filtering) [publish/subscribe](http://en.wikipedia.org/wiki/Publish/subscribe) library written in JavaScript.

PubSubJS has synchronisation decoupling, so topics are published asynchronously. This helps keep your program predictable as the originator of topics will not be blocked while consumers process them.

For the adventurous, PubSubJS also supports synchronous topic publication. This can give a speedup in some environments (browsers, not all), but can also lead to some very difficult to reason about programs, where one topic triggers publication of another topic in the same execution chain.

For benchmarks, see [A Comparison of JS Publish/Subscribe Approaches](http://jsperf.com/pubsubjs-vs-jquery-custom-events/51)

#### Single process

PubSubJS is designed to be used within a **single process**, and is not a good candidate for multi-process applications (like [Node.js – Cluster](http://nodejs.org/api/cluster.html) with many sub-processes). If your Node.js app is a single process app, you're good. If it is (or is going to be) a multi-process app, you're probably better off using [redis Pub/Sub](http://redis.io/topics/pubsub) or similar

## Key features

* Dependency free
* Synchronization decoupling
* ES3 compatible. PubSubJS should be able to run everywhere that can execute JavaScript. Browsers, servers, ebook readers, old phones, game consoles.
* AMD / CommonJS module support
* No modification of subscribers (jQuery custom events modify subscribers)
* Easy to understand and use (thanks to synchronization decoupling)
* Small(ish), less than 1kb minified and gzipped


## Examples

### Basic example

```javascript
// create a function to subscribe to topics
var mySubscriber = function( msg, data ){
    console.log( msg, data );
};

// add the function to the list of subscribers for a particular topic
// we're keeping the returned token, in order to be able to unsubscribe
// from the topic later on
var token = PubSub.subscribe( 'MY TOPIC', mySubscriber );

// publish a topic asyncronously
PubSub.publish( 'MY TOPIC', 'hello world!' );

// publish a topic syncronously, which is faster in some environments,
// but will get confusing when one topic triggers new topics in the
// same execution chain
// USE WITH CAUTION, HERE BE DRAGONS!!!
PubSub.publishSync( 'MY TOPIC', 'hello world!' );
```

### Cancel specific subscripiton

```javascript
// create a function to receive the topic
var mySubscriber = function( msg, data ){
    console.log( msg, data );
};

// add the function to the list of subscribers to a particular topic
// we're keeping the returned token, in order to be able to unsubscribe
// from the topic later on
var token = PubSub.subscribe( 'MY TOPIC', mySubscriber );

// unsubscribe this subscriber from this topic
PubSub.unsubscribe( token );
```

### Cancel all subscriptions for a function

```javascript
// create a function to receive the topic
var mySubscriber = function( msg, data ){
    console.log( msg, data );
};

// add the function to the list of subscribers to a particular topic
// we're keeping the returned token, in order to be able to unsubscribe
// from the topic later on
var token = PubSub.subscribe( 'MY TOPIC', mySubscriber );

// unsubscribe mySubscriber from ALL topics
PubSub.unsubscribe( mySubscriber );
```

### Hierarchical addressing

```javascript
// create a subscriber to receive all topics from a hierarchy of topics
var myToplevelSubscriber = function( msg, data ){
    console.log( 'top level: ', msg, data );
}

// subscribe to all topics in the 'car' hierarchy
PubSub.subscribe( 'car', myToplevelSubscriber );

// create a subscriber to receive only leaf topic from hierarchy op topics
var mySpecificSubscriber = function( msg, data ){
    console.log('specific: ', msg, data );
}

// subscribe only to 'car.drive' topics
PubSub.subscribe( 'car.drive', mySpecificSubscriber );

// Publish some topics
PubSub.publish( 'car.purchase', { name : 'my new car' } );
PubSub.publish( 'car.drive', { speed : '14' } );
PubSub.publish( 'car.sell', { newOwner : 'someone else' } );

// In this scenario, myToplevelSubscriber will be called for all
// topics, three times in total
// But, mySpecificSubscriber will only be called once, as it only
// subscribes to the 'car.drive' topic
```

## Tips

Use "constants" for topics and not string literals. PubSubJS uses strings as topics, and will happily try to deliver your topics with ANY topic. So, save yourself from frustrating debugging by letting the JavaScript engine complain
when you make typos.

### Example of use of "constants"

```javascript
// BAD
PubSub.subscribe("hello", function( msg, data ){
	console.log( data )
});

PubSub.publish("helo", "world");

// BETTER
var MY_TOPIC = "hello";
PubSub.subscribe(MY_TOPIC, function( msg, data ){
	console.log( data )
});

PubSub.publish(MY_TOPIC, "world");
```

### Immediate Exceptions for stack traces in developer tools

As of versions 1.3.2, you can force immediate exceptions (instead of delayed execeptions), which has the benefit of maintaining the stack trace when viewed in dev tools.

This should be considered a development only option, as PubSubJS was designed to try to deliver your topics to all subscribers, even when some fail.

Setting immediate exceptions in development is easy, just tell PubSubJS about it after it's been loaded.

```javascript
PubSub.immediateExceptions = true;
```