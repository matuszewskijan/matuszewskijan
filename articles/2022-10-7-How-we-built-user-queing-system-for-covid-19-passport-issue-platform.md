# How we built user queueing system for platform issuing COVID-19 passports - Rails, Sidekiq, Redis

> Building a platform issuing COVID-19 passports is a very diffcult task especially in terms of security and performance. I am describing the problems we faced working on the application and the solutions we found to successfully release the application in 3 weeks of development time.

* * *
#### *Jump straight to [the implementation section](#implementation) to see code examples, however, showing code isn't main point of this post so I encourage you to read the whole story.*
* * *

## Background

My colleague and I worked together on a very interesting application: **COVID-19 passport issuing platform**. The requirements were very simple:
1. End-user input their details into the web form
2. We are validating the input whether it's a legit person
3. If the validation is successful we're generating PDF with the passport

Sounds straightforward, right? But we had two main concerns beforehand:
1. **Security** - patient data can't be stolen, we have to make sure no one will obtain data from our database with any kind of attack.
2. **Performance** - it was the peak of the pandemy so we were aware that hundreds of thousands of people will go into application in a very short period of time.
3. **Time** - we had only **3 to 4 weeks** deadline when starting from completely scratch. Our team consisted of only 2 developers: I was the main developer working together with the tech lead who was also involved in the sibling project.

## Architectural solutions

We assumed that we have to put our users in some kind of queue as the process of passport generation was taking some time relying on 3rd party API.

After many hours of discussion we've decided that the best solution will be a 3(and a half) layered architecture:
1. **Static front-end server** - Next.js - blazing fast server because all of its files were static
2. **Public Rails API** - all of the front-end queries were landing there, it's only responsibilities were sanitizing the inputs and scheduling background workers, without access to the database.
    - **Redis** - the only touch point between the public API and separated backend. Being the main point of the client queue we relied on.
3. **Rails Backend** - unaccessible from the internet, stored in a DMZ. Having access to all of the sensitive data.

![Vaccination Portal Architecture](/images/vaccination-portal-architecture.png)


We were sure that the passport generation must be processed as a background job. `Sidekiq` were our first choice as the number we already knew it, in opposition to `Kafka` or `RabbitMQ`:
When we were reading about creating queues those two were the most often mentioned. However, none of us worked with those technologies and with such close deadline we decided that I'd be too risky.

### Implementation

All of the queuing systems available for Rubyists: **Sidekiq**, **ActiveJob**, **DelayedJob** and etc. aren't designed to work as a "users queue" so the user joins the queue and sees the waiting screen until the specific logic is processed. But after some digging we were able to align Sidekiq to work in the way we needed it.

Sidekiq by default works as FIFO([First-In-First-Out](https://www.geeksforgeeks.org/fifo-first-in-first-out-approach-in-programming/?ref=lbp){:target="_blank"}) queue. But we wanted to optimize it more - what if a user enters the queue but doesn't wait on the page until his request is processed?

It will be always processed in the default setup unless you use a trick that helped us tremendously:

### Meet [Redis TTL (time-to-live)](https://redis.io/commands/TTL){:target="_blank"}
```sh
redis> SET mykey "Hello"
"OK"
redis> EXPIRE mykey 10
(integer) 1
redis> TTL mykey # after 4 seconds
(integer) 6
redis> GET mykey # after more than 10 seconds
(nil)
```

Each background job is scheduled under a separate key in `Redis`. Given the knowledge of this key, you could utilize `TTL` function to wipe the job from the queue after X number of seconds.

When you schedule Sidekiq worker with `#perform_async` the Redis key is the result of this method call(it's named JID in [Sidekiq](https://www.rubydoc.info/gems/sidekiq/Sidekiq%2FWorker:jid){:target="_blank"}):
```ruby
001:0> job_id = ExampleSidekiqWorker.perform_async
=> "fc3f44f792492883d843fac4"
002:0> job_id = ExampleJob.perform_later.job_id # ActiveJob alternative
```

Next step will be to set the expiration time for our JID:
```ruby
# Rails applications likely do have Redis instance defined
redis = Redis.new(...)
redis.setex(job_id, 2)
```

Now if the scheduled job won't be picked by Sidekiq worker in the next 2 seconds the job will be completely removed from the queue. To prevent this behaviour the front-end have to ping the back-end as long as the user is on page.

Simple Javascript `setInterval` could do the job, or if you want something more sophisitcated then you can use [ActionCable](https://guides.rubyonrails.org/action_cable_overview.html#example-1-user-appearances){:target="_blank"}.

When our back-end receives the ping it should recall `redis.setex(job_id, 2)` to reset the time counter.

* * *
#### Did you know?
It's possible to schedule a Sidekiq worker from outside of your application:
```ruby
Sidekiq::Client.push(queue: 'external', class: 'ExternalWorker', args: [])
```
all you need is access to the Redis instance of the source app.
* * *

### Inform the user about queue length

```ruby
queue = Sidekiq::Queue.new('default')
aprox_wait_time = queue.length / queue.latency
```
<a href="https://www.rubydoc.info/github/mperham/sidekiq/Sidekiq/Queue#initialize-instance_method" class="text-grey font-small" target="_blank">Sidekiq Docs for reference</a>

Having the queue length and the latency(time between the last processed job) we could try to calculate how long the user will wait.
Let's assume that client number in queue is 600 and the queue latency is 0.5s => `(600 * 0.5 = 300s => 30min)`.

This calculation will give you only the estimation as latency could vary. ATM I don't know the method to find out the position of the key in Redis list. If we'd be able to find out it then we could recalculate the position for every front-end ping, currently, we could only calculate it at the moment of scheduling the worker.

### Retrieving results

Once the private back-end server process the job successfuly we were changing the value of the `job_id` Redis key to the URL of passport PDF. We were also setting the key expiration(`setex`) in Redis to 10 minutes to make sure the passport is publicly available only for a short period of time.

Afterall next front-end poll will read the passport PDF url from Redis and redirect the user to a success page showing the PDF.

On top of that there was a cookie signature ensuring that no-one with only the public URL could access the passport.


### Result

Using this strategy we were able to scale our application by spawning additional Sidekiq workers what is fairly easy. In our case the main limitation were rate-limit of the external API we needed to call.

I won't be humble - the release of this platform were a **huge success.** Our platform worked for hundreds of thousands of people with almost zero bugs. The 3 week deadline were a huge stress for everyone, but successful almost bug-free release gave everyone relief. I recon it as one of the most exciting programming challenges I've ever worked on.
