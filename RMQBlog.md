# Setting up a Highly Available RabbitMQ cluster for asynchronous tasks


## What are task queues and why are they required:

Task queue or message brokers accept and forward messages. They act like post office. Task queues manage background work that can be executed outside the usual HTTP request-response cycle. So a task queue acts like a mail box. A program that sends message to a mail box is a producer and the program that pulls tasks form the queue and performs them is called a consumer.

For example a website or an android app that pulls a person's facebook profile to maintain their database entries, would do so once at the time when the person signs up and later whenever the person logs in again can push the task to fetch the user's profile from facebook onto a task queue and deliver him the content he wishes to see without waiting for his updated details to be fetched.

Another example could be a sports website or app that lists the latest sports or cricket news, can use a task queue that will handle call to ESPN or Cricinfo's API every 15 minutes to refresh their page.

## RabbitMQ

RabbitMQ is a message broker [Task Queue], which can be easily configured at backend of one's web/android application and can be used to store the tasks that need to be performed but can be done asynchronously, i.e. does not require to be performed immediately after a user sends in some HTTP request.

## Danger

Although task queues can help increase your application's response time but there is always this threat what if the task queue fails or the server on which you were hosting your task queue goes down. This will have tremendous effects on your application. As you will lose important tasks that were to be performed and would be left with nothing to recover from.
So to fight such failure instances we come up with the solution of Highly Available RabbitMQ cluster, so that even if one queue goes down you don't lose all your data.

## RabbitMQ in more detail

RabbitMQ as previously said can be considered to be a post office which can keep one or more task queues. So we call a single such RabbitMQ post office as RMQ node or RMQ server.

## Highly available RabbitMQ cluster

A system is said to be highly available if it is fault resistant, i.e. if we have multiple RMQ nodes R1, R2, R3 and each has multiple queues. Then if queues on any node fail then all the messages present in the node's queues must get migrated to some other node so that there is no loss.

Now it is not possible to migrate all the messages from queues on 1 server to another server in case of abrupt power failure or fire kind situation. So we use queue mirrors. Queue mirrors are copy of the main queue maintained on other nodes.

So lets say we have a cluster of 2 nodes R1 and R2 as RMQ servers R1 has queue q1a and q1b. R2 has just 1 queue called q2. So to ensure that even in case of failure at either of the nodes, no queue is lost we can enable ha-policy (mirroring) that will ensure that each queue has its mirror on every other node in the cluster. So, in our case we will have three additional queues, q2_m at R1, and  q1a_m and q1b_m at R2.

![RMQ cluster before queue mirroring](/rmq-images/rmq_before.png)

## Working of a Highly available RabbitMQ cluster

In the above example the queues q1a,q1b and q2 are called the master queues, while the mirrors q1a_m, q1b_m and q2_m are called slave queues. 

- Whenever any task is enqueued onto master queue it will also get pushed into its slave(s). Similarly whenever any task is collected from the master queue by a consumer it will be dropped from the slave queue(s) as well. 
- If there are n nodes in a RMQ cluster then any queue made on any node will have a mirror on every other node in the cluster. So there will be n-1 mirror or slave queues per queue in a cluster.

- In case of multiple slave queues, the oldest slave is promoted to become the new master.

- A message is never directly sent to a queue. It is first sent a RMQ server and then if that node holds the master queue [that message is intended to go to], it will be enqueued on that queue, else it will be sent to the RMQ node that holds the master queue for that message and gets enqueued over there.

For example, if there is a task T supposed to be enqueued on q1a and it reaches node R1. It will be enqueued on q1a and subsequently at q1a_m.
And if a task T is supposed to be enqueued on q2 and reaches R1, it will first be routed to R2, then pushed to q2 and subsequently at q2_m.

- Similarly, a message is always consumed from the master queue, and subsequently dropped from the mirror queues. If a consumer tries to make a connection with some other node it will be routed to the correct node internally.

- Communication between RMQ cluster nodes is only possible when each RMQ node has the same Erlang key. This erlang cookie  on Linux system is usually present in /var/lib/rabbitmq/  directory. 

## Steps to make a Highly available RabbitMQ cluster using AWS

First create an AWS account if you do not already have one and open the EC2 management console page.

- Create a new security key which shall be same for all the AWS instances. Let this be named "rabbitmqkey". Any description can be added. Also change permission of the key file eg : chmod 400 rabbitmqkey.pem

-Create a new security group on AWS, with following under inbound ports: This shall be used by our EC2 RMQ instances.
```
SSH                TCP             22         0.0.0.0/0
Custom TCP Rule    TCP         0-65535        sg-x.x.x.x (your security group)
```
In above step adding your own security group id might not happen in first time. So create SG with SSH rule only and later add the 2nd rule.
- Next create a new AWS EC2 instance [This will be the first node of the RMQ cluster]
1. I used Ubuntu 16.04 server with t2.micro instance type. 
2. Use the same security group and security key for the instance when prompted.
3. Upon creation of your instance ssh into your  EC2 machine and run the following commands to install and set up a RMQ node running on this instance.
```
			sudo apt-get install rabbitmq-server
			sudo rabbitmq-plugins enable rabbitmq_management
			sudo rabbitmq-server -detached
```
- Management plugin is provided by rabbitMQ to be able to check status of tasks and queues present on the particular node and in the cluster. To check details of RabbitMQ management plugin check [1].

- Now to ensure that each node of the rabbitMQ cluster has the same erlang cookie and we don't need to install RMQ server again and again, we will create an AMI (Amazon machine image) of the above EC2 instance.

- After the image is constructed, create a new instance using that image. Make sure this instance also belongs to the same security group.

- SSH into the new EC2 instance created  and type in the  following commands:
```
			sudo rabbitmqctl stop_app
			sudo rabbitmqctl join_cluster rabbit@<private ip of 1st EC2 instance>
			sudo rabbitmqctl start_app
			sudo rabbitmqctl cluster_status
```

'join_cluster' will make the 2nd RMQ node join with the cluster of the 1st node. You can check this for more details about joining rabbitmq nodes in a cluster [2]

- Next again SSH into the main RMQ node and enable "High Availability", the mirroring policy. Mirroring policy needs to be activated just once for the entire cluster and should ideally be done at the first node of the cluster. The command for the same goes as:
```
			sudo rabbitmqctl set_policy ha-all "" '{"ha-mode":"all","ha-sync-mode":"automatic"}' 
```
To check more options for specifying the ha-policy of RMQ cluster check [3]

## Load  Balancing the RabbitMQ cluster:

- A rabbitMQ  cluster is effective if we have different queues on different nodes. Otherwise, all consumers will attach to the same server. Secondly if all queues are on the sane server then you are not using services of the 2nd server that is ready for use.

- So we can load balance our rabbitmq cluster using ELB [Elastic load balancer] provided by AWS.

- ELB is load balancing service provided by AWS. ELB directs any request to send/fetch task in Round Robin fashion. But we need not worry even if master queue for the message lies on another node and it is send to some other node. RabbitMQ internally sends the message to its right node to be enqueued there. This internal routing is very efficient and would not cause much overhead. 

- Launch an elastic load balancer (ELB) with the same security group as was used to create the EC2 instances. For listening to producers and consumers set the LB protocol to ```TCP/5672``` and that of instances to ```TCP/5672```.

- For listening to  management plugin: Set LB Protocol to TCP/15672 And Instance protocol to ```TCP/15672```.

- Thus your highly available RabbitMQ cluster is ready for use. You can use celery with Django to send and consume tasks from the RMQ queues. For details about configuring Celery and RabbitMQ with your project please check [4] [5]. Once you have set up the same you can fill in the ```broker_url``` [in your projects settings.py] used by celery to point towards the ELB instance. Use A-record address shown on ELB's description page for the same.

## Beware

ELB resets the connection with the application server. Thus, it is possible that your application tries to send some task onto the RMQ, but RMQ never receives the same. In such situation, it is better that in you settings.py [for Django users] you set confirm publishes to true like.
```
			BROKER_TRANSPORT_OPTIONS = {'confirm_publish': True}
```
This is something more Celery specific but it means that the application server would wait for the RMQ to send an ack before sending any further tasks to the same queue. Please refer [6] for more details.

## Remote Setup

You can use python and Parmaiko for setting up the RabbitMQ cluster on EC2 instances locally. If required can refer [7] https://github.com/aniket03/SR_proj for setting up Django-Celery-RMQ project and setting up the RMQ servers on EC2 remotely.

## References:

[1] https://www.rabbitmq.com/management.html

[2] https://www.rabbitmq.com/clustering.html

[3] https://www.rabbitmq.com/ha.html

[4] https://tests4geeks.com/python-celery-rabbitmq-tutorial/

[5] http://docs.celeryproject.org/en/latest/django/first-steps-with-django.html

[6] https://tech.labs.oliverwyman.com/blog/2015/04/30/making-celery-play-nice-with-rabbitmq-and-bigwig/

[7] https://github.com/aniket03/SR_proj 
