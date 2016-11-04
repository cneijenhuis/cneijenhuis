# The rat tail of webhooks

Webhooks seem to be such a simple solution for callbacks: Just send an HTTP request. Just receive an HTTP request and return 200.

However, once we leave the happy path, we see that it's not so simple - neither for the sender, nor for the receiver. One can surely ignore many of these issues if the cost of failure is low, e.g. if your CI build isn't triggered, or a message was not posted into your Slack channel. However, when looking at commerce use cases for callbacks, many of these have a high cost of failure, e.g. if your lunch is not delivered within the promised time, or if a refund is not triggered after a return.

Let's have a look at what can go wrong:

## The delivery fails

Oops, we can't deliver the webhook succesfully because the destination itself or the network isn't available. What should the sender do?

Turns out: It depends. If the callback is supposed to trigger a refund on your credit card, it should be retried ad nauseam. However, if the callback is supposed to trigger a push notification with "your food delivery is on the way", it should be dropped after a couple of minutes as the customer has received his meal already.

## The receiver can't process a specific webhook

Oops, there is a bug in the receiver that is triggered by a specific payload. How can the developers of the receiver retrieve this payload? A nice sender would offer a dead letter queue that can be accessed via a UI and/or an API.

## The receiver can only process so much messages at once

How many HTTP requests can be send in parallel before overloading the receiver? 5? 50? 500?

Workloads in commerce are often spikey. Your frontend needs to scale, but for many of the background processes, a viable strategy is to be asynchronous and buffer tasks. Webhooks are not a good buffer, because the sender pushes to the receiver. If the receiver can pull the work, he can consume it at his own pace.

As a workaround, the receiver can refuse to accept webhooks under high load, and trust the sender to retry the delivery. However, the receiver would refuse incoming requests randomly. It may accept one request that is delivered for the first time, but refuse another one that has already been retried multiple times. Instead of a constant delay ("all tasks are worked on with a delay of 15 min"), you get random task processing ("some tasks are stuck since hours").

## Integration with monitoring systems

Even if the sender has adressed all of the above problems, it is still vital that the system can be monitored and that automated alerts can be send out when deliveries fail, a message is added to the dead letter queue or the buffer size grows. This should obviously nicely integrate with the system the (Dev)Ops are using already for monitoring the rest of the infrastructure.

# Message Queues are built to solve these issues

Fortunately, there is already a lot of software out there that is built to solve all of these problems - Message Queues! They allow you to define retry policies, dead letter queues, and are often integrated with your favorite monitoring software. They can be consumed via a pull-API, and many can be integrated with auto-scaling workers/lambdas.

With a Message Queue, the delivery is clearly separated from the processing of the message. It allows the sender to hand it off with a much higher change of success, because neither bugs nor performance issues in the receiver will have an impact on the delivery. On the other hand, the receiver has control over the messages. Not only can the policies be defined, it's also possible to manually manage the state of the queue. Often this is possible via a UI, which may also allow you to peek at messages.

## Using a Message Queue from a Webhook

If you are integrating with a system that only offers webhooks, fear not: It is quite easy to just accept the message and put it into the message queue of your choice. Better yet, a few message queues have an interface to act as a webhook-receiver themselves (for example IronMQ)!

# Callbacks in the commercetools platform

When designing our API and platform, we're giving our best to not only make it easy to learn and develop for, but also easy to maintain once a shop is in production. When we started designing our callback API, we first looked deeply at webhooks, because everybody knows them, and they are, hands-down, the easiest thing to implement. But when we started designing for maintainability in a production environment, we found ourselves re-inventing the wheel and building yet another message queue.

We decided to take a step back. While it surely would have been fun to build our own message queue, our mission is first and foremost to design a great commerce platform! Therefore, we decided to let our callback API stand on the shoulder of giants: Messages are put into a Message Queue of your choice. We currently support SQS on AWS, PubSub on Google Cloud and IronMQ of iron.io. Just like with programming languages, you can choose the one that fits your needs best.
