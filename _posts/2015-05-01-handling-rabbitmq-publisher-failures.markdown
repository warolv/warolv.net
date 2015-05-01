---
layout: post
title: "Handling RabbitMQ publisher failures"
date: 2015-05-01 14:21
comments: true
categories:  [rabbitmq, bunny]
description: RabbitMQ publisher failures, bunny publisher failure control, make bunny asynchronous using jobs, resque for bunny gem
keywords: rabbitmq publisher, publish with bunny, bunny job control, bunny failures, rabbitmq failures, rabbitmq publisher, resque bunny
sharing: true
---

In a previous post I wrote about [Splitting your app into smaller apps using RabbitMQ](http://warolv.net/blog/2015/04/27/splitting-your-app-into-smaller-apps-using-rabbitmq/), talked about the basics, gave an example of code to create basic publisher and consumer with [bunny](https://github.com/ruby-amqp/bunny) and [sneakers](https://github.com/jondot/sneakers) and how to connect all pieces together.

Today I want to talk about handling publisher failures for dashboard app, an example, when we experience a connection problem because broker for any reason is down, you need to handle this case because the bunny is synchronous and your app may get stuck.
To solve this issue, I added background job: [Resque](https://github.com/resque/resque) in my case we are using Redis/Resque for background jobs, with a great plugins which called [Resque retry](https://github.com/lantins/resque-retry) and provides retry, delay and exponential backoff support for resque jobs and [Resque scheduler](https://github.com/resque/resque-scheduler) for support of queueing items in the future.

<!-- more -->

Now let's write some code - adding a wrapper called message_sender which runs publisher and if there some problem exist and we got an exception, job executes with a delay of 1 minute (configure this value for your needs).

#### Message Sender
``` ruby 
  class MessageSender
    def self.deliver(message)
      begin
        Publishers::MessagePublisher.publish(message) # it's publisher's code from previous post
      rescue => e
        Resque.enqueue_in(1.minutes, Jobs::DeliverMessageJob, message.id)
      end
    end
  end
``` 

#### Deliver Message Job
``` ruby 
  require 'resque-retry'

  class DeliverMessageJob
    extend Resque::Plugins::Retry
    @queue = :messages_delivery

    @retry_limit = 3
    @retry_delay = 60

    def self.perform(message_id)
      message = Message.find_by_id(message_id)

      if message
        MessageSender.deliver(message)
      end
    end
  end
``` 
In DeliverMessageJob we used 'resque-retry' plugin to retry job 3 times with a delay of 1 minute in case of failure (configure this value for your needs), during this time we might fix this problem.

In coming post I will talk about 'Job Handling Strategies' chosen for consumer.