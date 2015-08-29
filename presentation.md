class: center, middle

# Rabbitmq and message queue

###A gentle introduction and Some real live examples

[live presentation](http://alonisser.github.io/introduction-rabbitmq) <br/>
[twitter](alonisser@twitter.com), [medium](https://medium.com/@alonisser/)

###Shameless promotion: you can also read my political blog: degeladom@wordpress.com
---

# Installation

```bash
wget http://www.rabbitmq.com/rabbitmq-signing-key-public.asc
sudo apt-key add rabbitmq-signing-key-public.asc
echo 'deb http://www.rabbitmq.com/debian/ testing main' | sudo tee /etc/apt/sources.list.d/rabbitmq.list
sudo apt-get update
sudo apt-get install -y rabbitmq-server
sudo rabbitmqctl set_vm_memory_high_watermark 0.9
sudo rabbitmq-plugins enable rabbitmq_management

```
Or use my Ansible playbook..

---

# message queues: The Why:

* Decoupling
* Order
* Persistent (optional) - when compared to http

---
# Using message queues: design patterns

## Message driven vs Event Driven

(What I would like you to remember from this talk)

---

# Message driven

* remote rpc, sorts of.
* We specifically tell a remote service **WHAT TO DO** and (optionally) provide the information needed to do this

---

# Message driven - Real world example: "Belery"

* Use case: In our view we need to call a long running task, out of the request-response cycle.  

```python
def long_running_task(name):
    
    sleep(10)
    logger.info(name)
    
class SomeView(object):
    Belery.apply_async(long_running_task,'alon') 
````
---

# Belery pseudo code to enqueue the task:

```python
# Belery uses kombu to connect to rabbitmq as a message broker
from kombu import Connection, Exchange, Queue, BrokerConnection, producers

BeleryTask = namedtuple('BeleryTask', ['task', 'arg_list', 'kwargs_dict'])

        

class Belery(object):

    def __init__(self):
        self.connection = BrokerConnection(settings.BROKER_URL)
        self.exchange = 'Belery'
                
    def apply_async(a_task, *args, **kwargs):
        task = BeleryTask(a_task, args, kwargs)
        
        self.publish_task(task)
                       
    def publish_task(message):
        with producers[self.connection].acquire(block=True) as producer:
           producer.publish(message,
            delivery_mode=2,
            exchange=self.exchange,
            routing_key='tasks',
            serializer='pickle')
    
```

---

# Belery "worker" process pseudo code
  
```python
class BeleryWorker(object):
    def __init__(self):
        with self.consumer(callback=self.consume_message) as consumer:
            consumer.drain_forever() # Does something that fetches message from the queue and feed to the callback handler
        
    def consume_message(message):
        task_class = pickle.loads(message.body) #Notice that we don't really do that with kombu but for brevity in this example
        try:
            
            task_class.task(*task_class.arg_list, **task_class.kwargs_dict)
            message.ack()  #Notice that the message has an ack method - to allow us moving to the next message
        except Exception as e:
            message.requeue()
            raise e
            
for i in settings.['BELERY_WORKER_COUNT']:
    BeleryWorker()
```
---

# Belery - conclusion

* Scales well, as long as we provide a connection to a "message broker" that we can consume messages from
* THIS IS A PSEUDO CODE, don't expect this to work "as is", you can use Belery older brother: Celery. That would handle most of the things this code elegantly avoids 
* Actually I lied, Distributed Task Queue isn't a great example for the message driven pattern, but most of you are familiar with. So let's look at a better one.

---

# Message driven: real world example: "Mailer"


--- 

# Event driven

* Event notifies other services that "SOMETHING HAPPENED", It's up to them what to do with this.
* We don't tell the remote service "WHAT TO DO", we don't even know who would the downstream consumers be.
* The number of consumers can change overtime, as needs change, develop, etc.
* An event can be as "thin" as a name with semantic meaning:

```python
class UserLoggedIn(object):
    def __init__(self, user_id):
        user_id = user_id
        name = self.__class__.__name__
        timestamp = calendar.timegm(datetime.now().timetuple())
        event_id = uuid.uuid4()
        
```
---

# Event driven

* An  event can be as "fat" as the needs of the downstream consumers:

```python
class FatUserLoggedIn(object):
    def __init__(self, user_id, user_data, login_site, login_type, total_number_of_users):
        user_id = user_id
        name = self.__class__.__name__a
        timestamp = calendar.timegm(datetime.now().timetuple())
        event_id = uuid.uuid4()
        user_data = user_data
        login_site = login_site
        login_type = login_type
        total_number_of_users = total_number_of_users
        
```

---

# Event driven: real world example


---

# Message driven Vs Event driven: Summary

* Messages are "WHAT TODO", events are "WHAT HAPPENED"
* Messages have a specific recipient. Events don't - so might trigger multiple actions in different services
*  We can use both..
* "The difference being that messages are directed, events are not.
a message has a clear addressable recipient while an event just happen for others (0-N) to observe it"
[Jonas boner update to the reactive manifesto](https://www.typesafe.com/blog/reactive-manifesto-20)

---

# AMQP

## Advanced Message Queueing Protocol

* defined set of messaging capabilities
* network wire-level protocol

---

# Rabbitmq

* Erlang

---

# Terminology

---

* Virtual host - administrative. not very relevent if you are not setting up a rabbitmq cloud business
* Exchange - the post office
* Connection (Can have many channels)
* Queue  - a mailbox
* bindings - Relationship between Queue and Exchange
* Broker
* Message
* Consumers

---

# Exchanges

* Exchanges take a message and route it into zero or more queues
* Can be durable or transient (durability by persistence to disk)
* exchange types: direct, fanout, topic, headers.

---

# Queues

* What you might expect.. 
* Durable or Transient. 
* Can have multiple consumers! (clients) (But can be also exclusive)

---

# The message

* a message is sent with a routing key, headers.
* Has a routing key
* has headers (all kinds of metadata)
* messages can be acked, nacked (only in rabbit) rejected or requeued (we'll come back to that later)
* In rabbitmq: does not support yet priority level

---

# Multiple consumers

* One point to consider: prefetching (see celery issues with that)

---

# Exchanges - The commonly used: Topic exchange

* pattern based routing keys (with limited wildcard)
* Publishers publish with different routing keys, while different queues can each bind to multiple (and different keys)
* Think: publisher publishes messages with routing keys: "food.pizza", "food.icecream", and another publisher "deserts.icecream",
One queue can consume (bind - see later) "food.*" and get everything food related. and the other consumer ask for "*.icecream" to 
get everything icecream related or just "deserts.icecream" to get only messages with this routing key

---


# Exchanges - The commonly used: fanout exchange

* No need for routing keys
* Publishers publish to exchange
* All consumers get all the messages

---

# Summarize until now
* Each message received by the exchange would be delivered to *all* relevant queues (by bindings). 
* A message is consumed by *one* consumer (client) in the queue (unless rejected)

---
# In python

##Rabbitmq binding: 

* librabbitmq - c, no support for python 3
* amqp - pure python, some problems

##Client libraries

* Kombu
* Pica

---

class: center, middle

#Show us some code


---


class: center, middle

#rabbitmqadmin


---


class: center, middle

#Plugins


---

class: center, middle

#Thanks for listening!

---
