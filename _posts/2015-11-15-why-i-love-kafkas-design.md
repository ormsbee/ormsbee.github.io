---
layout: post
title:  Why I Love Kafka's Design
date:   2015-11-15 00:00:00
---

Many developers revere the interface. As long as you build the right
abstractions, they believe, how you actually store the data is an implementation
detail that can be fixed up later. Just make a clean API that has a clear
division of responsibilities, and you will be fine. I've heard this time and
time again, particularly in reference to initial implementations of APIs that
worked fine for N=100, but that we intended to "scale up later" to handle a
thousand times that (it wasn't wishful startup thinking — we already had those
use cases).

The problem is that for non-trivial applications, you often can't know what the
"right abstractions" are until you understand your data at the desired scale.
One of my favorite examples of this is [Kafka](http://kafka.apache.org/), an
open source messaging system originally developed at LinkedIn.


# Poor First Impressions

When I first encountered Kafka four years ago, the server protocol made no sense
to me. Things were poorly documented back then (those were early days), and I
was trying to fix a bug in a Python client that was written in a hurry by
someone that didn't fully grok it either.

The interface was odd, you see. For instance, when you queried for messages,
it was something like this: "Give me 500K of messages from topic 'events',
partition #4, starting at offset 49,031,920,381." The server would then give you
back exactly 500K worth of message data. If 500K cuts off halfway through a
message or header, that was the client's problem.

This immediately offended my design sensibilities. As developers, we get
abstraction and separation of concerns drilled into us all the time. Why was it
forcing me to know all this? As a client, I shouldn't care about low level
details like the partition, and certainly not byte offsets. I should never have
to worry about partial messages. I expected to be able to say, "Give me the next
500 messages from topic 'events'." This interface was clearly wrong.

Except that it wasn't. Kafka is actually a brilliant example of what can be done
when you look at the requirements and think hard about your data before rushing
off to build a pretty abstraction.

# Understanding the Data

Kafka's design goals included durability and massive throughput. Durability
meant that data had to be written to disk. Disk I/O is often seen as a
performance killer, but the folks that designed Kafka recognized that it's
really the random seeks that hurt you. Sequential reads and writes are actually
pretty fast, and operating systems will use gobs of RAM to cache that data for
you.

So they basically opted for writing out messages as big log files. When a
consumer makes a request for a topic and partition, that maps to a directory on
the filesystem. The requested offset will let Kafka know what file to read, and
what byte offset within that file to start from. You can get partial messages
back because (and this is the awesome part), Kafka doesn't ever need to actually
read the content of the data it sends you. It uses the
[`sendfile`](http://man7.org/linux/man-pages/man2/sendfile.2.html) system call
to send that data directly from the [page cache](https://en.wikipedia.org/wiki/Page_cache)
to the network buffer without ever copying anything to user space.

This [clever design](https://kafka.apache.org/08/design.html) allows them to
[scale up](https://engineering.linkedin.com/kafka/benchmarking-apache-kafka-2-million-writes-second-three-cheap-machines)
to millions of messages per second across a few cheap machines. It lets
clients replay events, and it removes a lot of complexity from the core server
code.

There are tradeoffs of course. The simple logfile mechanism means that there is
no advanced message routing functionality as found in other messaging systems.
Consumers have to do a lot more bookkeeping, and there is a Zookeeper
orchestration layer that advanced clients have to know to coordinate with each
other. It's also just not very pretty. But it scales insanely well, and that's
a perfectly valid tradeoff to make.

# The Lesson I Took Away

> Reality is not a hack you're forced to deal with to solve your abstract,
> theoretical problem.
>
> Reality is the actual problem.
>
> — Mike Acton, [Data-Oriented Design and C++](https://youtu.be/rX0ItVEVjHc?t=1120)

Kafka's interface is what it is because that's the way disks and operating
systems work in the real world. It's not the most elegant or featureful of APIs,
but they could not have arrived at a solution that met their goals nearly so
well if they had started with an ideal interface and figured out the data store
later.

Our job as software developers is to design and implement systems that meet
various business requirements while being reliable, maintainable, secure, and
performant. Abstraction and elegance are highly desirable things that often
further those goals, but they can be traded off against, just like anything
else.

Web developers may not be working as close to the metal as Kafka, but we still
have to deal with real data models in actual databases. Yet a lot of API design
discussions just ignore this part as a low level concern and focus on abstract
notions of elegance, or whatever API conventions are in vogue that year. It's
like we're buying a house, all our debates focus on the paint and the lawn, and
the question of whether we can make the mortgage payments is a detail that we
can worry about later because it takes too much effort to think about.

Usually, the rationale for this is that if you get the interface "right", you
can change the data model under it without disrupting everyone. So the API is
the thing you have to commit to. But again, that ignores the fact that we won't
know if the API we've dreamed up is actually workable at larger scale. It also
ignores one other inconvenient detail about data.

> "We managed to migrate over our data earlier than we thought we would."
>
> — No one, ever, in the history of software development.

If your system has any success at all, you will be stuck with your basic data
model for far longer than anyone wants to admit to themselves. Data migration is
a pain, and there's always Something More Important to work on.

The real takeaway for me was that the data model is not a second-class citizen.
Data gets to push back on API design. The more scale matters, the bigger say it
has.
