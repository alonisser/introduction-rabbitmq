class: center, middle

# Message queues and Rabbitmq

##An introduction to using message queues, design patterns and RabbitMQ

[live presentation](https://alonisser.github.io/introduction-rabbitmq) <br/>
[twitter](alonisser@twitter.com), [medium](https://medium.com/@alonisser/)

###Shameless promotion: you can also read my political blog: **[degeladom](degeladom@wordpress.com)**
---

class: center, middle

# Why this talk?

---

# message queues: The Why?

* Decoupling
* Order
* Persistent (optional) - especially when compared to http
* For the age of microservices: modular, evolving, eventually consistent, reactive applications.


---
class: center, middle
# Using message queues: design patterns

## Message driven vs Event Driven

(What I would be glad if you remembered from this talk)

---

# Message driven

* remote rpc, sort of.
* We specifically tell a remote service **WHAT TO DO** and (optionally) provide the information needed to do this

---

# Message driven - Real world example: "Belery"

* Use case: In our view we need to call a long running task, out of the request-response cycle.  

--

```python
def long_running_task(name):
    
    sleep(10)
    logger.info(name)
    
class SomeView(object):
    Belery.apply_async(long_running_task,'alon')
    return Response()
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

# Belery has multiple workers:
## processes consuming tasks from rabbitmq 

--

"worker" process pseudo code:

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

--

* THIS IS A PSEUDO CODE, don't expect this to work "as is", you can use Belery older brother: Celery. That would handle most of the things this code elegantly avoids

--

* Actually I lied, Distributed Task Queue isn't a great example for the message driven pattern, but most of us are familiar with. So let's look at a better example.

---

# Message driven: real world example: "Mailer"

Lets say we have a "mailer" service, a simple app that integrates with mandrill api.
It receives the email data via http, compiles templates and sends via mandrill.

--

```python
class SomeView(object):
    user_id = self.kwargs.get('pk')
    mailer_proxy.send_email(user_id, 'The new password you requested has been sent')
    return Response()
    
class MailerProxy(HttpProxy):
    def send_email(user_id, text):
        httpProxy.post('themailer.example.com/mail/password_email/', data={'user_id':user_id, 'text':text}) # lets assume we have requests.post underneath
         
mailer_proxy = MailerProxy()
    
```

---

# Message driven: real world example: "Mailer"

##Questions:
--

* What would happen if the mailer service just blew up? mandrill api was down? network hickup? We just lost our email.

--

* We could setup a retry scheme, error handling, etc. But could we handle all cases from the sending app?

--

* What if the requests just timed out but the email was actually sent. a retry scheme would cause it to be resend or risk not sending at all.

--

* We need a better solution.

---

# Message driven: real world example: "Mailer"

Lets replace the MailerProxy to use a message queue instead

--

```python
class MailerProxy(BusPublisher):
    def send_email(user_id, text):
        message = SendEmailCommandMessage(user_id=user_id, text=text)
        self.publish(message)
```

--

Now, if the mailer message queue binded to SendEmailCommandMessage is persistent (and it should be) then the event would just wait in the queue until
The mailer app is ready to consume it..
 
Maybe after a tired programmer woke up and deployed some fixing code to the UnhandledException..

All the expected emails, in the expected order, would still be sent. and the sending app does not need to know anything about the state of the mailer app.

---

# Event driven
## Lets move to an "Event driven" example

* An event notifies other services that "SOMETHING HAPPENED", It's up to them what to do with this.

* We don't tell the remote service "WHAT TO DO", we don't even know who would the downstream consumers be.

* The number of consumers can change overtime, as needs change and develop, etc.

---


# Event driven

### An event can be as "thin" as a name with semantic meaning:

--

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

### An  event can be as "fat" as the needs of the downstream consumers:

--

```python
class FatUserLoggedIn(object):  # Please don't be offended, fat is about the event data..

    def __init__(self, user_id, user_data, login_site, login_type, total_number_of_users):
        user_id = user_id
        name = self.__class__.__name__a
        timestamp = calendar.timegm(datetime.now().timetuple())
        event_id = uuid.uuid4()
        user_data = user_data
        login_site = login_site
        login_type = login_type
        # I can include dubious metadata, if downstream someone needs this
        total_number_of_users = total_number_of_users  
        
```

---

# Event driven: real world example

Lets say we are building a financial application and Our main financial service handles a deposit
So following the processing of the deposit we would emit this event.

```python

class UserDepositedBusEvent(BusEvent):
    #  BusEvent base class adds timestamp, name, unique event id, handles serialization etc

    def __init__(user_id, deposit_amount):
        super(UserDepositedBusEvent, self).__init__()
        self.user_id = user_id
        self.deposit_amount = deposit_amount
        
```
--

* Notice the event name: We report something that **happened** and **not** something that someone else should do.

---

# Event driven: real world example

## Our first consumer - notifications center app/service

```python
class UserDepositedBusEventHandler(BusEventHandler):
    def handle(event):
        assert isinstance(event, UserDepositedBusEvent)  # Helping the code intel :)
        Notification.objects.create(user_id=event.user_id,
         text='Congrats for your {0} deposit'.format(event.deposit_amount)
```

---
# Event driven: real world example

## Now the Business demands another, unspecified before, usage:

**"Attention: From now on, deposits should also be recorded as a transaction."**

Since we have a transaction app that listens to the message queue, we add a handler for the same event in this app:

```python
class UserDepositedBusEventHandler(BusEventHandler):
    def handle(event):
        assert isinstance(event, UserDepositedBusEvent) #Helping the code intel of the idea
        Transaction.objects.create(user_id=event.user_id, transaction_type=TransactionTypesEnum.Deposit, amount=event.deposit_amount)
```        

---
# Event driven: real world example
* Notice that the notifications center app, the original consumer, does not have to know anything about the new consumer. Both listen to the same event and do different things with it.

* Notice that we can add more and more services that use the same event while keeping them completely decoupled.

* We can use different programming languages, different platforms, etc.

---

# Event driven: real world example

## Lessons learned:

* Use JSON, not Pickle, even if the producer is a python app. so can consuming becomes language agnostic.

 I can consume message with a java service, different python services (both 3.4 and 2.7), Node.js or whatevent hipster language you use.
 
* Figure a way to save all published events, someday a new micro service would join the crowd and would need to initialize its state with all existing data.
Writing a data migration using the app logic (the event handlers) would be easier if you can query all past events (or specific events)

* Figure a way to make event handling idempotent: account revisions, db storing of handled event ids, Whatever works for your app.

---

# Event driven: real world example

## Lessons learned:

* Be prepared for eventually consistent state: A notification can be created before the transaction is created of vice verse,
 depends on what other events are stuck in the queue of each app and how fast they are processed
 
* Make publishing an event part of the standard workflow. Even if there is no consumer currently, someday there would be.

* Separate busEvents classes to libraries, so code can be reused between different apps of the same language (we use gemfury as a private cheeseshop)

* Be prepared to handle race conditions (see next)

---

## Lessons learned: inherent race conditions

consider this code:

```python
class UserDepositedBusEventHandler(BusEventHandler):
    @transaction.atomic
    def handle(event):
        assert isinstance(event, UserDepositedBusEvent) #Helping the code intel of the idea
        transaction = Transaction.objects.create(user_id=event.user_id, transaction_type=TransactionTypesEnum.Deposit, amount=event.deposit_amount)
        User = get_user_model()
        user = User.objects.get(user_id=event.user_id)
        user.deposit(event.deposit_amount) # Triggers a user.save() inside the method
        serialized_transaction = self.get_serializer(instance=transaction).data
        busPublisher.publish(TransactionCreatedBusEvent(user_id=event.user_id, transaction=serialized_transaction))
```   

* Since I wrapped the handler with transaction.atomic to prevent the user.deposit() to be saved if the transaction creation has failed, 
then the busPublisher.publish() would publish an event on the message queue before the actual save took place! 

* If a downstream service receives the event and immediately triggers an http request to the upstream server depending on the transaction to exist 
then it might fail! (and yes it **did** happen, quite often, mostly in automatic e2e testing, but even with real users)


---


## Lessons learned: inherent race conditions

* My (partial) solution is wrapping the publishing of events to downstream with *on_commit* hook of some kind. 
So the actual publishing takes place only after the data has been commited to db.

---

# Message driven Vs Event driven: Summary

*"The difference being that messages are directed, events are not.
a message has a clear addressable recipient while an event just happen for others (0-N) to observe it"*
[Jonas boner update to the reactive manifesto](https://www.typesafe.com/blog/reactive-manifesto-20)

--

* Messages are "WHAT TO DO", events are "WHAT HAPPENED"
* Messages have a specific recipient. Events don't - so might trigger multiple actions in different services
* We can use both..
 
---

# AMQP

## Advanced Message Queueing Protocol

Trivia:
* Most of the contributes to the current spec work in either JPMorgan Chase, Red hat or Cisco.. Open source, Big tech and Banks

Actually Two different parts:
* "defined set of messaging capabilities" the AMQ Model
* "network wire-level protocol" AMQP 

---

# RabbitMQ

* Written in Erlang
* Robust solution with widespread adoption and quite good tooling.
* Quite easy to install and configure, as stand alone and in cluster mode. (Yap HA is here)
* Available also with multiple cloud services.
--

* Other popular message queue offerings I bumped into (NOT necessarily RabbitMQ based): IronMQ, Aws SQS.  

---

# Terminology

* Virtual host - administrative. not very relevant if you are not setting up a RabbitMQ cloud business
* Exchange - the post office
* Connection (Can have many channels)
* Queue  - a mailbox, "stores messages until they can safely processed"
* bindings - Relationship between Queue and Exchange - the mail address perhaps
* Broker
* Transports (Actually Kombu specific)
* Message
* Consumers

---

# Broker

* Message middleware servers: Such as RabbitMQ

---

# Connection and channels

* Connection is the tcp connection. you need at least one
* There can be many channels on this one connection (But at least one)
--

* "Best practice for mutlithreaded apps consuming from a queue is using a channel per thread." (I read this somewhere..)

---

# Exchanges

* Exchanges take a message and route it into zero or more queues
* Can be durable or transient (durability by persistence to disk)
* exchange types: direct, fanout, topic, headers. (We'll get back to that later)
* In Rabbitmq only (not in the spec) exchanges can also bind to other exchanges (allowing complex routing topology)

---

# Queues

* What you might expect.. 
* Durable or Transient. 
* Can have multiple consumers! (clients) (But can be also exclusive)

---

# The message

* Has a routing key (In topic and direct exchanges at least)

* Has headers (all kinds of metadata)

* Can be acked, nacked (only in rabbit) rejected or requeued (we'll come back to that later)

--

* In RabbitMQ: does not support priority level yet.
* delivery mode (transient or durable)

---

# Consumer

* Consumes messages from a queue (or queues), Can be analogues to an email client.
--

* A specific message is consumed in a specific queue by ONE consumer

* There can be multiple consumers connected to one queue.
---

# Multiple consumers

* If multiple consumers are connected to the same queue, then a message would be delivered only to one consumer 
 (as long as it was acked) - using some sort of round robin logic.
* One point to consider: prefetching (see celery issues with that)

---

# Exchanges - The commonly used: fanout exchange

* No need for routing keys
* Publishers publish to exchange
* All consumers get all the messages
* Use case: "Missile attack alert" to all troops. "Market is closed" to all brokers 

---

# Exchanges - the commonly used: direct exchange

* the queue is binded with a specific key (common practice: queue name). and gets everything the exchanges sends with that key
* Example: a queue for each account Id. that handles all the account messages
* Multiple queues can use the same key (Actually behaving the same as a fanout exchange for this key) - all of the queues would get the same messages

---


# Exchanges - The commonly used: Topic exchange

* pattern based routing keys (with limited wildcard)
* Publishers publish with different routing keys, while different queues can each bind to multiple (and different keys)
* Example: publisher publishes messages with routing keys: "food.pizza", "food.icecream", and another publisher "deserts.icecream",
One queue can consume (bind - see later) "food.*" and get everything food related. and the other consumer ask for "*.icecream" to 
get everything icecream related or just "deserts.icecream" to get only messages with this routing key


---

# Summary until now

* Each message received by the exchange would be delivered to *all* relevant queues (by bindings). 
* A message is consumed by *one* consumer (client) in the queue (unless rejected)

---
# Acking

* You should ack messages. You can ack multiple message in the same command
* Unacked messages would be delivered to other consumers
 for the same queue and might be redelivered (if not acked by another consumer) after the consumer reconnects
* You can Reject messages - if (requeue=False) then It would be discarded. if (requeue=True) it would just be placed in the queue again
* The AMQP spec has an [incompatibility about reject](http://www.rabbitmq.com/blog/2010/08/03/well-ill-let-you-go-basicreject-in-rabbitmq/). 
So the **current** RabbitMQ implementation allows a consumer to get the same rejected messages again (if requeued) 
* Nack - allowing multiple reject
* Consider: Dead lettering (an exchange getting the reject messages, for further treatment perhaps)
---

# Persistence

* I lied again, RabbitMQ isn't persistent by default, exchanges are not, queues are not, messages are not.
--

* But we can make them persistent: :) 
* Queues and Exchanges should be declared as persistent at **CREATION**, so restarting the rabbitmq-server process won't kill them
--

* But.. the messages in queue would still be lost after a service restart........
* Message durability is defined when sending the message: use delivery_mode=2 to ensure persistent. 
--

* But be warned:

*The persistence guarantees aren't strong*

* For stronger message persistence guarantees you can use [publisher confirms](https://www.rabbitmq.com/confirms.html)
--

* Persistence, of course, comes with a price.

---

# In python

##Rabbitmq binding: 

* librabbitmq - c based,fast, no support for python 3
* amqp - pure python + 3, some problems

##Client libraries

* [Kombu](https://kombu.readthedocs.org/en/latest/) From the celery project, quite good exprience with it.
* [Pika](https://pika.readthedocs.org/en/0.10.0/) looks a little lower level, I didn't write production code with it
* [Puka](http://majek.github.io/puka/puka.html) Just bumped into it, looks nice, especially the asynchronous api.
 I'll try it in a future project

---

# Kombu

* Nice high(er) level api 

>>The aim of Kombu is to make messaging in Python as easy as possible by providing an
>>idiomatic high-level interface for the AMQ protocol, and also provide proven and tested
>>solutions to common messaging problems

--

* Supports multiple transports: both amqp and others: sqs, zeromq, redis, etc..
* Transports: a Kombu abstraction over message brokers

* Would default to High performance c based Librabbitmq if installed (only python 2.7!)

* consumer and producers pools, mixins, other goodies

---

# Connecting and consuming in python (Kombu)

```python
def connect(self):
        self.connection = Connection(self.connection_url, heartbeat=30)

        with self.connection as connection:
            channel = self._connect_channel(connection)
            self.exchange = Exchange(name=self.exchange_name, type=self.exchange_type, channel=channel, durable=True)

            self.queue = Queue('%s_updater' % self.app_name, channel=channel, durable=True)
            self.queue.declare()
            self._register_topics()
            self.logger.info('Connected to broker %s' % self.connection_url)

            with connection.Consumer(self.queue, callbacks=[self.process_message]) as consumer:
                '''
                callbacks are a list of functions accepting decoded message body and
                message instance (for headers for example)

                alternative is using on_message - would be a function receiving a raw and undecoded message
                if you want to override the Consumer internal receive method and disable callbacks
                '''
                try:
                    self._drain_events_forever(connection)
                except socket.timeout:
                    self.logger.warning('socket timeout')

                    self.reconnect()

```

---

# Connecting and consuming in python (kombu)

```python
def process_message(self, body, message):

        key = str(message.delivery_info['routing_key']).split('.')
        topic = key[0]
        event_name = key[1]

        marshaller = self.bus_event_mapper.get_marshaller(topic, event_name)
        if marshaller and event_name in self.event_handlers.get(topic, []):
            try:
                marshaled_event = marshaller(body)
                handler = self.get_handler(topic, event_name)
                handler.handle(marshaled_event)
        
```

---

# Connecting and consuming in python (kombu)

```python
def _drain_events_forever(self, connection):
        '''

        :param connection:
        :return:
        kombu best practice generator
        Used to drain events from queue with some error handling
        '''

        for _ in eventloop(connection, timeout=240, ignore_timeouts=False):
            pass

def _register_topics(self):
    '''

    :return: None
    bind self.queue to routing keys created dynamically from topic and event names
    '''

    for topic in self.event_handlers:
        for event_name in self.event_handlers[topic]:
            self.queue.bind_to(exchange=self.exchange, routing_key=self.create_routing_key(topic, event_name))
        
```

---


# Publishing in python (kombu)

```python
def publish(self, message, routing_key):

        try:
            message = message.serialize()
        except AttributeError:
            pass

        with producers[self.connection].acquire(block=True, timeout=60) as producer:
            
            try:
                logger.info('{0} publishing to bus {1}'.format(self.app_name, message))
                if not RecoveryMode.is_in_recovery:
                    producer.publish(message,
                                     delivery_mode=2,
                                     exchange=self.exchange,
                                     routing_key=routing_key,
                                     serializer="json")
            except:
                logger.exception('{0} unable to publish bus event: {1}'.format(self.app_name, message))
                raise
```
---
class: center, middle

#the management plugin: rabbitmqadmin

the management plugin.
Provides an http api, web based frontend. and also a command line tool

---

# Todo (Things you SHOULD handle):

* Monitoring on queues, connections.
* Keep alive
* Graceful exit.

---


#Plugins

Rabbitmq also has plugins..

[The plugins](https://www.rabbitmq.com/plugins.html)
```bash

rabbitmq-plugins enable plugin-name
rabbitmq-plugins disable plugin-name
rabbitmq-plugins list

```
Some notable plugins (Many more there..):  Consistent Hash Exchange Type, shovel (Shoveling message from a queue to other brokers/exchanges)
, Tracing, stomp, mqtt, federation

Of course: the must have management plugin

---
# Installation (For future reference)

```bash
wget http://www.rabbitmq.com/rabbitmq-signing-key-public.asc
sudo apt-key add rabbitmq-signing-key-public.asc
echo 'deb http://www.rabbitmq.com/debian/ testing main' | sudo tee /etc/apt/sources.list.d/rabbitmq.list
sudo apt-get update
sudo apt-get install -y rabbitmq-server
sudo rabbitmqctl set_vm_memory_high_watermark 0.9 #If on dedicated machine
sudo rabbitmq-plugins enable rabbitmq_management

```
Or use my Ansible playbook..

---

class: center, middle

#Thanks for listening!

---
