Thought I'd share my experience as this is so commonly missed.

**What's diagnostics exactly?**

When something goes wrong in production, it's in everyone's best interest to find out what exactly is broken as quickly as possible.

**What should programmer do to help with that?**

**1 Lower complexity**

Consider this. You've got a physical machine, it's got Linux and you run a C application that opens a listening TCP socket and echoes back whatever input it's receiving. Something goes wrong, what is the list of things you can check for errors, however small the possibility?

- Your small piece of code. Whatever you see is exactly how it's working. No extra dependencies.
- System-level code - the user and kernel space C sources.
- Your OS configuration.
- Hardware.

Now let's say it's actually a virtual machine. With Docker inside. That's running JVM. That's running an OSGi container. That, in turn, is running Spring Framework. That runs Netty. And finally Netty runs your echoing logic. In that scenario, there's just such a huge amount of code that could go wrong anywhere and you'd potentially have to debug and go through everything.

The thing is, even though most of that code is probably well-tested and isn't under suspicion, it still occasionally gets buggy like everything else so it's always under suspicion. It may not be the hypervisor's network emulation code but *you don't know that*.

No only does increased complexity makes diagnostics much slower and takes much more energy and resources, but it also makes it hard to predict your system's stability. The more complex a system is, the more fragile it is, the harder it is to forecast a chance of something going wrong.

Now, that doesn't mean everything should be written from scratch. But when considering architecture, complexity should be kept in mind. Those additional layers of software should not be picked just because it's cool and fashion and people give talks about it in conferences with pretty interiors and video recording and everything.

The simpler, the better.

**2 Simple configuration**

In the complex example above also consider that each layer has a configuration. How do you check through that as quickly as possible and get accurate data?

The host OS has its kernel configuration, the hypervisor has its own, the guest OS has its own, the Docker, the JVM, the Karaf (OSGi container), the Spring's. That's a lot of places to check. 

Now I don't know about config files outside your app, but what you can do for your app is this. Avoid clever, parametrised configuration systems that are resolved in runtime. In production, keep your configuration resolved and immutable.

Here's an example, I worked at a company where many applications queried a shared database for config key-values. As input, they would give a host name, an environment type, an environment name. Like, someapp.opsnetwork.int/production/real, or someapp.opsnetwork.int/qa/mercury. If there was a full match, that value would be returned. Otherwise, it dropped the environment name and try again. No luck, drop the environment type. If something along the way matched, that value would be returned. Now with that system, if you wanted to quickly see the full list of configuration values, what would you do? Spend a hell of a lot of time digging through the whole table trying to resolve those rules in your mind. What's worse, the env type/name could be given directly on each call, or taken from the process environment variables. If the variables weren't there, it'd just silently default to something like "development". And it's a multi billion dollar company.

Even if you have a hierarchical system like that, you could still just resolve everything and dump it into a file on startup and then use the file. That way when trouble comes, you just look into it and immediately get the full configuration picture of your app. And if the database or the remote configuration service goes down, you won't be affected by it. But that's more about fault tolerance than diagnostics.

**3 Timings**

Place checkpoints across your code and take timestamps at them. Take the time it's taken for an operation to complete and if the threshold is exceeded, log the checkpoints along with their times.

Realistically, an operation is anything with an entry point:

- An HTTP request
- An MQ message
- A scheduled job
- A filesystem event

Anything that initiates an action.

I've seen such library implemented like that:

In your HTTP request handler, create a `Timings` object and give it a name with the request's URL. Pass that object down the whole chain of calls.

At blocking places in your code, add checkpoints. I.e, before a database call, before a logging operation (writing to disk is expensive), before an external network call, etc. In Java, you can use AOP to automatically bind that operation to every `@Controller` or `@Transaction` or `@Cache`.

When your code is finishing processing the request, i.e. probably back in the HTTP request handler, call `finish()` on the `Timings` object. The finish will look at time the operation taken and if the configured threshold is exceeded, it will log something like that:

    Threshold exceeded for /myurl; taken=1412ms, threshold=100ms. Details:
      15ms     fetching config value
      91ms     loading user info
      1388ms   finished

So you immediately know that it was between "loading user info" and "finished" that something blocked for a long time.

It's important to log this only when the threshold is exceeded, to avoid slowing the code down by constant logging and spamming the logs with useless info.

You could also parse that log and feed its data to a monitoring system to build charts showing how checkpoint time difference is spread across the uptime.

**4 Detailed error logging**

Let's just say many popular open source frameworks really suck at logging so this is really important, as it's wrong most of the time since people just take popular stuff and use it.

In Java, when you catch an exception, don't just log it. Carefully describe the context. If that was a network call, what was the IP address and the port? If that was a timeout, what was the timeout threshold and actual time taken? If that was a filesystem access, what was the absolute path?

Bad:

    java.net.ConnectException: Connection timed out
    Caused by: java.net.ConnectException: Connection timed out

Good:

    Connection to 74.125.131.138:80 (google.com) timed out; timeout=5000ms, time taken=51234ms

Don't just log a hostname as it might get resolved to different IP address on different attempts. Print the IP address. But also print the host name to make it easier for Ops to understand what was the intention.

**5 Passthrough operation id**

With every log message, add the operation id to it. Definition of operation is the same as in timings, but for HTTP requests it's usually generated by nginx sitting in front of your backend services and passed on in HTTP headers. When you make a network call, pass that operation id along. That way, when one of the services break, you could filter logs from all services by the operation id that we assume the customer gave you from their 500 Internal Error webpage and get a clear picture of what was logged in relation to that specific user transaction.

**6 Careful error handling**

Don't swallow errors/exceptions. Ever. If some API says there could be an error, however low the probability, handle that case and *log* it. You wouldn't believe the amount of headbanging during a post-mortem analysis that goes on just because someone thought "hey, c'mon, I know this exception will never be thrown and look, my code is so much cleaner without all the error handling!".

Apache's IOUtils in Java have a method called `closeQuietly()`. It does a try/catch around trying to close an `InputStream` I think, if memory serves. In the catch section, it just does nothing. Now, Google's Guava has a similar method, but they actually log a WARN in case an error actually happens, so maybe programmers at Google know something, eh? It's just an example that even respected libraries screw up such fundamental things.

It would take a lot of time to explain why you should catch and log and handle everything, so I'll just give a small example about closing a database connection. So you've done your query, got a result and now want to close the connection, what does it matter if it fails? You don't need it anymore! Thing is, an error during closing might indicate an existing unstable situation with either your application or the database. It might not all break now, but under different conditions - like a higher load - it might escalate quickly. It's the same reason you should treat all warnings as errors when building source code. Actually much more important than compiler warnings in general, but you get the idea.

**7 Metrics**

Collect everything. But especially:

- Incoming network requests: total, how many failed with a 500, how many active now, process times by percentiles 50, 75, 85, 90, 95, 99, 99.9, etc.
- Outgoing network requests: same as above.
- Scheduled operations: same as above.
- Thread pools: free, busy, max.
- Connection pools: free, busy, max.
- Any other kind of pool: same thing.
- An embedded cache: how close to capacity, etc.

**The end**

So that's it. There could be some errors in the text, feel free to correct me,
and certainly there could be some edge cases where what I described doesn't
really fit, but these are general things to keep in mind, not a dogma. I might
have forgotten something, in which case I'll update the post. Hope this helps somebody.
