---
layout: post
title: "Handling RabbitMQ consumer failures with maxretry handler"
date: 2015-05-16 12:23
comments: true
categories:  [rabbitmq, bunny]
description: RabbitMQ consumer failures, sneakers failure control, maxretry handler, sneakers job control
keywords: rabbitmq consumer, sneakers, sneakers job control, sneakers failures, rabbitmq failures, rabbitmq consumer
sharing: true
---

In previous posts I talked about [Handling RabbitMQ publisher failures](http://warolv.net/blog/2015/05/01/handling-rabbitmq-publisher-failures/) and [Splitting your app into smaller apps using RabbitMQ](http://warolv.net/blog/2015/04/27/splitting-your-app-into-smaller-apps-using-rabbitmq/). Now it’s time to talk about consumer failures handler.

For consumer I use a great gem called ['sneakers'](https://github.com/jondot/sneakers), which uses different [job handling strategies](https://github.com/jondot/sneakers/wiki/Failures), with the default one called Oneshot handler, because it only tries once. For our app the maxretry handler was chosen; from its description: “Maxretry uses dead letter policies on Rabbitmq to requeue and retry messages after failure (rejections, errors and timeouts). When the maximum number of retries is reached it will put the message on an error queue.”

<!-- more -->


##### Sounds good, so let’s do it—but what does “dead policies” means?
###### It means messages from a queue can be republished to another exchange when any of the following events occur:

* The message is rejected (basic.reject or basic.nack) with requeue=false
* The TTL for the message expires
* The queue length limit is exceeded.



For our configuration, I used a retry number equal to 5, so, in case of failure, we resend the message up to 5 times, and only then move our message to error queue, delaying for 60 seconds before we resend the message.

Ok, let’s write some code, starting with the definition of a worker, which uses a custom handler (maxretry) and not the default (Oneshot).

##### Sneakers Initializer
``` ruby 
  # config/initializers/sneakers.rb
  require 'sneakers'
  require 'sneakers/handlers/maxretry'  # you must add handler to this folder

  config_file = File.read(::Rails.root.to_s + '/config/sneakers.yml') 
  config = YAML.load(config_file)[::Rails.env]
    .symbolize_keys
    .merge(handler: Sneakers::Handlers::Maxretry, workers: 3)
  Sneakers.configure(config)

  WORKER_OPTIONS = {
    retry_timeout:      60 * 1000,  #60 sec
    ack:                true,
    threads:            10,
    prefetch:           10,
    ....
  }
``` 

##### Configuration for sneakers
``` ruby 
  # config/sneakers.yml
  development:
    amqp: "amqp://guest:guest@localhost:5672"
    vhost: "/"

  integration:
   ....

  production:
    ...
``` 

##### Creating a worker which defines dead letter policies to resend failing messages
``` ruby 
  class MessagesWorker
    include Sneakers::Worker

    from_queue  'mailer.messages',
        WORKER_OPTIONS.merge(
          {:arguments => {:'x-dead-letter-exchange' => 'mailer.messages-retry'}}
        )

    def work(message)
        MyMailer.send_mail(msg).deliver

        ack!
    end
  end
``` 

After running a worker: ‘WORKERS=MessagesWorker rake sneakers:run’, and going to admin:http://localhost:15672, be sure that you see 3 queues created:

<img src="{{ root_url }}/images/maxretry-admin.png" />

* mailer.messages - your original queue
* mailer.messages-error - we’re moving the message here after 5 failed retries
* mailer.messages-retry - maxretry using this queue to resend message in case of failure, TTL ‘flag means’ we’re using a delay between retries: 60 sec in our case

‘DLX’ flags on mailer.messages and mailer-retry means we are using dead-letter-policy (x-dead-letter-exchange) on those queues

##### So how does it work?

* The message comes to the ‘mailer.messages’ queue and consumers try to send it. If it fails because of the ‘DLX’ flag defined with the ‘mailer.messages-retry’ exchange, we send it to the ‘mailer.messages-retry’ queue through a defined exchange.
* Because the ‘TTL’ flag is defined, we wait for 60 seconds and then we send it again to ‘mailer.messages’ queue, in the meantime increasing our retry counter.
* We repeat this procedure 5 times in case of failure and then move the message to the ‘mailer.messages-error’ queue.

For more details, you can look at the [maxretry handler](https://github.com/jondot/sneakers/blob/master/lib/sneakers/handlers/maxretry.rb).

##### How to extract message from error queue to resend it or for debugging purposes?

* Go to the error queue, click on the “get messages” button and copy this message.
* Run the rails console and use Base64.decode64 (‘paste your copied message here’).
