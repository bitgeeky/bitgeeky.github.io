---
layout: post
title: Redis For HA And Load Balancing Of A Service - Part-1
date: 2016-11-12 08:37:59 UTC
updated: 2016-11-12 08:37:59 UTC
comments: true
categories: redis browserstack ha load balancing homepage
---

Its been long since I have written a blog post, mostly because of my busy schedule and new life after University.
A lot has happened since then, about which I plan to write seperate blog posts. 

I have been working with the DevOps team at [BrowserStack](http://browserstack.com/) for a few months now. In this post I'll describe an interesting problem I have been working on recently.

Problem
----
Writing a highly available/ multi AZ service deployed independently in each region which does aggregation of data in fixed time intervals and pushes to a database cluster.

Lets first look and try to understand the given architecture:
<img src="/public/images/event_data_service.png" alt="Event Data Service"/>

In the above architecture we have:

1. Multiple Services and Products all in same region. For example: us-east
2. All services communicate with database servers via DNS which provides round robin load balancing.
3. Machine-1 and Machine-2 both have exact same configuration and services - UDP broadcast relayer and InfluxDB in two availability zones. For example: us-east-1a and us-east-1b
4. If a request from any of the productA, productB etc hits our DNS it can be redirected to either Machine-1 or Machine-2.
5. UDP relayer make sure that both databases contain the same data, i.e if a request comes on Machine-1 it is automatically sent to Machine-2 and vice versa.
6. Each product sends the database cluster an event message whenever a user event happens on it.
7. The message contains 4 parameters (event-type, product, username, timestamp).

Event message examples:
{% highlight bash %}
event_type=rails-5xx,product=live value=testusername1 1478507534415525888
event_type=stuck_in_queue,product=live value=testusername2 1478507534415525888
event_type=browser_error,product=automate value=testusername3 1478507534415525888
event_type=browser_error,product=automate value=testusername4 1478507534415525888
event_type=rails-5xx,product=live value=testusername1 1478507534415525890
{% endhighlight %}

Note: The time stamp for the above events is in nanoseconds.

Consider the following use case:

1. For instrumentation purpose, we need to count the total number of particular event types(eg: rails-5xx, browser_error, etc) which happened for a particular product(eg: live, automate) per minute and also get the same count for unique users.

According to the given use case the above 5 example messages will yield an output:
{% highlight bash %}
unique_user_event,event_type=rails-5xx,product=live value=1
cumulative_user_event,event_type=rails-5xx,product=live value=2
unique_user_event,event_type=stuck_in_queue,product=live value=1
cumulative_user_event,event_type=stuck_in_queue,product=live value=1
unique_user_event,event_type=browser_error,product=automate value=2
cumulative_user_event,event_type=browser_error,product=automate value=2
{% endhighlight %}

Understanding the above output
----
```cumulative_user_event``` tag describes events which happened within one minute for a particular product, i.e if an event happened for testuser1 two times within a minute it will be counted two times. On the other hand

```unique_user_event``` tag describes events which happened within one minute for a particular product, considering unique users only, i.e if an event happened for testuser1 two times within a minute it will be counted once only.

1. The first output message says: Within a duration of 1 minute the product `live` gave `rails-5xx` to 1 user only.
2. The second output message says: Within a duration of 1 minute the product `live` gave `rails-5xx` 2 times.
3. The third output message says: Within a duration of 1 minute the product `live` gave `stuck_in_queue` to 1 user only.
4. The fourth output message says: Within a duration of 1 minute the product `live` gave `stuck_in_queue` 1 time only.
5. The fifth output message says: Within a duration of 1 minute the product `automate` gave `browser_error` to 2 users.
6. The sixth output message says: Within a duration of 1 minute the product `automate` gave `browser_error` 2 times.

Service Requirements
----
Let's call the above example messages as input, the output yield as output and the service to be implemented as Event Data Service.
It should be pretty clear by now that we need to write a service which takes in the `input` from multiple products and sends `output` to the database cluster.

Easy and unpromising solution
----
We use a simple hash map with the following keys and values:

1. `cumulative_user_event,<product_type>,<event_type> ` : ` (int)(count_of_events)`
2. `unique_user_event,<product_type>,<event_type>` : `(set)(usernames)`

Whenever a new message comes we increase the counter for cumulative_user_event, and add a username to set corresposnding to unique_user_event.
Parallely, we run a loop every minute and send the aggregated ouput:
`cumulative_user_event with the counter and unique_user_event with the cardinality of set`
to database cluster and clear the hash maps.

<img src="/public/images/event_data_service_easy_solution.png" alt="Event Data Service Easy Solution"/>

Problems with above solution
----
The above solution works perfectly fine when deployed on a single machine as a service, but deploying the service on a single node makes it a single point of failure for our complete system, i.e even though we have HA for our database but whenever the node containing Event Data Service crashes, all our database servers stop receiving data.

If you observe carefully, in this solution we can simply deploy the service on Machine-1 or Machine-2 and just point our DNS to one of them with deployed service. It will disable our HA and when the server pointed by DNS crashes whole of our system will crash.

Deploying Event Data Service on multiple machines
----
This is the actual solution for our problem and the main reason of this blog post, i.e we want to deploy our Event Data Service both on Machine-1 and Machine-2.

Challenges and Brainstorming
----
I'll describe solution for how to achieve this in my next post i.e Part-2 of this series but I'll leave some interesting hints for reader to brainstorm on:

When Event Data Service gets deployed to both Machine-1 and Machine-2, DNS points to both the machines and does regular round robin load balancing:

1. Which machine gets the event messages from products ?
2. When does that machine gets the message ?
3. Which machine processes and pushes data and at what intervals ?
4. Does the one minute loop finish at same time on both machines ?
5. What is the title of this post ? :)

Until then keep thinking and stay tuned. Feel free to discuss the possible solution with me in the meantime or ask questions if you couldn't get the problem statement clearly.
