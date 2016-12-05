---
layout: post
title: Redis For HA And Load Balancing Of A Service - Part-2
date: 2016-12-04 10:08:18 UTC
updated: 2016-12-04 10:08:18 UTC
comments: true
categories: redis browserstack ha load balancing homepage
---

In [last post](http://pankajmalhotra.com/Redis-For-HA-And-Load-Balancing-Of-A-Service-Part-1) I described the deployment of event data service on a single node.

Quick Recap:
<img src="/public/images/event_data_service_easy_solution.png" alt="Event Data Service"/>


Easy and unpromising solution
----
We use a simple hash map with the following keys and values:

1. `cumulative_user_event,<product_type>,<event_type> ` : ` (int)(count_of_events)`
2. `unique_user_event,<product_type>,<event_type>` : `(set)(usernames)`

Whenever a new message comes we increase the counter for cumulative_user_event, and add a username to set corresposnding to unique_user_event.
Parallely, we run a loop every minute and send the aggregated ouput:
`cumulative_user_event with the counter` and `unique_user_event with the cardinality of set`
to database cluster and clear the hash maps.

We also discussed the problems with above solutions, main problem being when the machine goes down we start losing data and this can happen quite often in a cloud environment.
But if you observe carefully, deploying on a single machine is not even an option here !


Why single machine is not even an option ?
----
Machine-1 and Machine-2 both sit behind a DNS which does a round robin load balancing i.e some requests go to Machine-1 and others go to Machine-2. If we want to deploy our service on one of the machines say Machine-1 we will start losing data which comes on Machine-2.

The worst idea is to boot up Machine-3 in-between Machine-1 and Machine-2 on which we can deploy our service. After processing data, Machine-3 sends data to databases on both the machines.

I call it the worst solution because it means, booting up of a machine for just one service, usage of an elastic-ip and a new DNS entry and still the whole system can fail on failure of just one machine.


Extending the same solution to multiple machines
----
Now, we plan to make this service highly available, so that even if one of the machine for this service goes down our system is not impacted and everything continues to work the same way.

We deploy our Event Data Service on both Machine-1 and Machine-2 behind a UDP relayer:
<img src="/public/images/event_data_service_ha_solution.png" alt="Event Data Service"/>

1. Whenever request comes to either of Machine-1 or Machine-2 it gets replicated to both the machines by udp replayer. If you observere the picture carefully, there are two arrows from UDP Replayer, one that sends data directly to InfluxDB and the other which sends it to our Event Data Service. This is because the data we want to process is not the only data that our UDP relayer receives, it receives other data also which isn't of any interest 

2. Important point to note here is that the complexity of our UDP replayer has increased now, the dumb relayer which was made to only forward/ broadcast requests is now also processing data and distinguishig which request should directly be sent to database and which one to the Event Data Service.

3. Lets call Event Data Service on Macine-1 as EDS-1 and the one on Machine-2 as EDS-2.

4. Both EDS-1 and EDS-2 receive same data and start processing it.


Problem with this deployment
----
Important point to note here is that 1 minute for EDS-1 and EDS-2 can be different, i.e the processing time isn't synchronized.
Even though the system clocks on both machines are synchronized, i.e `date` command on both machines shows same value but:

{% highlight python %}
while True:
    sleep(60)
    print "One minute finished"
{% endhighlight %}

When you run this script on both machines independently there is no guarantee that both will always print "One minute finished" at same time. Hence, as shown in picture below the processing time of one minute can overlap or just get out of sync anytime.

<img src="/public/images/eds_clocks.png" alt="Event Data Service Easy Solution"/>

In the overlapping intervals shown in above image same data gets processed twice by both EDS-1 and EDS-2 and we get duplicate/ incosistent data on database which doesn't makes any sense.

It is clear that both services can't operate simltaneously, i.e a minute's data should be processed only once by one of them.


Solution to above problem:
----
This problem is actaully one of the most common problem of a Distributed System and hence has a number of solutions to it, I'll describe one of them here:

1. Master-Slave architecture: We use only one of {EDS-1, EDS-2} at a given point of time. Both nodes send each other periodic heart beats and one of them takes over as a master node for processing the data.
If EDS-1 goes down EDS-2 will take over and vice-versa.

This ensures HA of our service ! Yes, now our service is highly available, because same logic can be scaled horizontally to add more nodes, for 3 or more node cluster.


Steps towards a scalable design and Load Balancing
----
If you think carefully, in our solution only one of the service out of {EDS-1, EDS-2} is processing data, the other one is sitting idle and just sending heart beats.

We can modify our service and design it in such a way that the load of data processing can be distributed among the active nodes so that in future if the volume of data will increase our system can be scaled by simply adding more number of nodes and keeping architecture and implementation the same. Computing power of a single node in previous solution can become a bottle neck, the new solution should also remove any such bottle necks from our system.


Taking advantage of Redis and NTP time:
----
Now try to think of the problen yourself again, you'll realize that from very begining all we needed was just a memory which is shared by both machines and is always same for both.
That is the data once taken by EDS-1 shouldn't be taken again by EDS-2, and since we need an aggregate over one minute we can take advantage of system time, the one shown by `date` command since that will be same on both machines ! It gets synchronized via NTP client no matter what geographical area the nodes are in.


Final Solution
----
To keep track of clock time we define our 0 point as mid night and all the minutes after that have an abolute value, which can also be defined as `minutes_since_midnight`.
The value of `minutes_since_midnight` will be same on both machines since both are synchronized by NTP.

We keep 1 sorted set and 3 unsorted sets in Redis:

1. Sorted set "minutes" - stores minutes_since_midnight values.
2. Unsorted set "<minutes_since_midnight>_event_types" - contains types of events that happened for a particular product in a particular minutes_since_midnight.
3. Unsorted set "<minutes_since_midnight>_unique_user_<event_type_for_a_product>" - contains usernames for which the event happened.
4. Unsorted set "<minutes_since_midnight>_cumulative_user_<event_type_for_a_product>" - contains usernames + timestamp for which the event happened.

When the events for a product type comes:
Add minutes_since_midnight to a set "minutes" with a weight equal to the value.
{% highlight python %}
ZADD 'minutes' minutes_since_midnight minutes_since_midnight
{% endhighlight %}
Even if both machines push the same value multiples time, we are sure that there is only a single entry.

Keep track of all the events happened for a product in a particular minute:
{% highlight python %}
SADD minutes_since_midnight+'event_types', <event_type_val for a product>
{% endhighlight %}
Since, this is also a set, it ensures each type of event for a product gets pushed only once.

Preappend `minutes_since_midnight` to each event type to create a set so that we 
can later get to know the events which came in a particular minute for a particular product:
{% highlight python %}
SADD minutes_since_midnight+'unique_user,'+<event_type> <username>
{% endhighlight %}
We store the usernames in this set, the cardinality of this set will later give us the
number of users for which a particular event happened for a particular product.

Similar to above set we construct a set of cummulative events here, the difference is that
we also add a timestamp appended to the username in this set, this is done to differentiate
between two events which happened at two different times but within same minute.
{% highlight python %}
SADD minutes_since_midnight+'cumulative_events,'+<event_type> <username>+<timestamp>
{% endhighlight %}
And since this is a set, even if EDS-1 and EDS-2 push the same event, it will get stored only once.

This will become more clear with the following example:
Consider the same input as in previous example.

{% highlight python %}
event_type=http-5xx,product=productA value=testusername1 1480876707352348928
event_type=os_error,product=productA value=testusername2 1480876707352348928
event_type=browser_error,product=productB value=testusername3 1480876707352348928
event_type=browser_error,product=productB value=testusername4 1480876707352348928
event_type=http-5xx,product=productA value=testusername1 1480876707352348930
{% endhighlight %}

The above events happen within same minute as indicated by timestamp in nanoseconds.
If we calculate the minutes since midnight for the above events it will be 1118.

Now, value of minutes_since_midnight = 1118
other datastructures in Redis will have the values:

{% highlight python %}
minutes = {1118,}
1118_event_types = {"event_type=http-5xx,product=productA",
    "event_type=os_error,product=productA",
    "event_type=browser_error,product=productB"}

1118_unique_user_event_type=http-5xx,product=productA  = {testusername1}
1118_unique_user_event_type=os_error,product=productA  = {testusername2}
1118_unique_user_event_type=browser_error,product=productB  = \
					   {testusername3,testusername4}

1118_cumulative_user_event_type=http-5xx,product=productA  = \
  {testusername1_1480876707352348928, testusername1_1480876707352348930}
1118_cumulative_user_event_type=os_error,product=productA  =
				     {testusername2_1480876707352348928}
1118_cumulative_user_event_type=browser_error,product=productB  = 
				     {testusername3_1480876707352348928}
{% endhighlight %}


Service operation on Redis data
----
Both EDS-1 and EDS-2 start poping out an element form the "minute" set, this make sure if EDS-1 gets minute value of 1118
it gets deleted from the set and EDS-2 can never get that value.

Assuming that EDS-1 gets the minute 1118 to be processed it will iterate over the event types of ```1118_event_types```
and get cardinality of each set corresponding to the event.

{% highlight python %}
min_since_midnight = pop_from_redis_set("minutes")

# value of minutes_since_midnight = 1118

for event_tpe in 1118_event_types:
    get_cadinallity(1118_unique_user_event_type)
    get_cadinallity(1118_cumulative_user_event_type)
    delete_from_redis(1118_unique_user_event_type)
    delete_from_redis(1118_cumulative_user_event_type)
delete_from_redis(1118_event_tpes)
{% endhighlight %}

This will give us the desired output:

{% highlight python %}
unique_user_event,event_type=http-5xx,product=productA value=1
cumulative_user_event,event_type=http-5xx,product=productA value=2
unique_user_event,event_type=os_error,product=productA value=1
cumulative_user_event,event_type=os_error,product=productA value=1
unique_user_event,event_type=browser_error,product=productB value=2
cumulative_user_event,event_type=browser_error,product=productB value=2
{% endhighlight %}

Final Architecture:

<img src="/public/images/final_event_data_service.png" alt="Event Data Service"/>


That's it ! Now we have a highly available load balanced service which can be deployed on a cluster of more than two nodes.

Feel, free to get in touch of you have a better design or want to discuss more on this.
