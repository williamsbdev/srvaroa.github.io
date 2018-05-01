---
layout: post
title:  "GC forensics by example: allocation pressure and multi-second pauses"
date:   2018-04-28 11:31:22 +0200
categories: gc jvm java garbage-collection latency gc-pause
---

This post will use a real example to illustrate the concept of
allocation pressure as a cause of pathological behaviours on the JVM's
garbage collector. I assume some basic familiarity with the topic, you
know what a generational collector is, what is the basic lifecycle of an
object, that the G1 collector structures the heap in regions, etc. If
you want more info on these topics the [Oracle JDK
documentation](https://docs.oracle.com/javase/9/gctuning/) is a good
starting point.

The application ran mostly on default settings (I don't have them at
hand unfortunatly), with the G1 collector (`-XX:+UseG1GC`) and used
aprox. the following settings to produce GC logs:

    -Xloggc:$PATH
    -XX:+PrintGCDetails
    -XX:+PrintTenuringDistribution
    -XX:+PrintPromotionFailure
    -XX:+PrintGCApplicationStoppedTime

The logs are written at `$PATH`. In this case, they were filled with
logs similar to this:

	...
	2016-03-15T17:45:47.751+0100: 0.621: [GC pause (G1 Evacuation Pause) (young)
	Desired survivor size 4194304 bytes, new threshold 15 (max 15)
	, 0.0213976 secs]
	   [Parallel Time: 9.5 ms, GC Workers: 18]
	      [GC Worker Start (ms): Min: 621.3, Avg: 623.5, Max: 630.7, Diff: 9.4]
	      [Ext Root Scanning (ms): Min: 0.0, Avg: 0.2, Max: 1.4, Diff: 1.4, Sum: 3.1]
	...

Which is a detailed log of each GC event in the JVM. They'll be useful
to us later, but it's not the best format to get a sense of how GC is
behaving over time. A tool like [GC
Viewer](https://github.com/chewiebug/GCViewer) comes handy in these
situations.

<p align="center">
    <a href="/assets/gcviewer_sample_1.png">
	<img src="/assets/gcviewer_sample_1.png">
    </a>
</p>

The graph shows time on the horizontal axis. Both memory size in MBs and
pause time are shown in seconds on the vertical axis.

Two stacked areas represent the size of the Young and Old Generation
(yellow and pink respectively). We had about 9.5GB Old Gen plus 8.5GB of
Young Gen. These only show memory reserved by the JVM for the heap.
Usage is represented with thin lines inside each area. In the Old
Generation (pink area), a darker pink line shows Used Old Gen. In the
Young Gen, usage is represented with a light grey line.

According to the graph, the application allocated new objects in the
Young Gen until it eventually filled up. At those points, (when the grey
line hit the upwards limit of the yellow area), the JVM was forced to
make room in the Young Gen. This is called a **Minor Collection**. After
memory was released, the application continued creating objects and the
cycle repeated.

Minor collections recover memory in two ways. First, by releasing memory
consumed by objects that are no longer referenced. This is what happens
up to about 20:00:00. In that interval, we see a common see-saw pattern
showing that the JVM was being able to keep up with the application. It
was collecting all 8.5GB of Young Gen on each iteration.

But this pattern changes after 20:00, where the size of both the Old Gen
(pink area) and its utilisation (pink line) start growing. The reason is
that the JVM found live objects in the Young Gen that survived several
collections and decided to promote them to Old Gen. This is also called
**tenuring**. The "tenuring threshold" is the maximum number of Young
Collections (or "age") that an object may survive before being promoted
to Old Gen.

The JVM had to expand the Old Gen<sup>[1](#foot1)</sup> because it was
now seeing more capacity requirements. (I will keep speaking about
Young/Old Gen in the abstract, although the G1 collector does not keep a
single space for each generation and instead divides the heap in
regions. If you're not familiar with this check out [this overview of
the heap layout in
G1](https://docs.oracle.com/javase/9/gctuning/garbage-first-garbage-collector.htm#JSGCT-GUID-15921907-B297-43A4-8C48-DC88035BC7CF))

The diff between (1) and (2)
suggests that over half of the Young Gen was moved to Old Gen. It turns
out that the JVM had a fixed max heap of 18GB (-Xmx18g) so the JVM
wasn't able to extend it in order to make room for the bigger Old Gen.
So instead, it shrunk the Young Gen. This brought two interesting
effects.

    REMOVE: 1 == tip of the grey line right at the start of the resize
	    2 == tip of the pink line right at the end of the resize


* First, because the application kept allocating new objects at the same
  rate, the Young Gen filled up much faster, causing more frequent Young
  Collections forming a tighter (and narrower, cleaning up less memory)
  see-saw pattern.
* Second, shortly after growing the Old Gen, and after two cycles in the
  tigher see-saw pattern (3), the used Old Gen (pink line) suddenly
  drops to the same level as it was prior to the resize. In other words,
  the JVM just moved ~6GB to Old Gen, only to garbage-collect it shortly
  after.  (╯°□°）╯︵ ┻━┻

We're seeing here a case of **allocation pressure**: the application
generates too many short-lived objects, the JVM must trigger more
frequent Minor Collections, this makes live objects age faster and hit
the **tenuring threshold** earlier, which means getting them promoted to
the Old Gen prematurely (**premature promotion**.)

## Why is this bad?

Take a look at the grey rectangle at the bottom of the graph, precisely
in the interval where the JVM is growing the Old Gen. This is how
GcViewer represents a GC Pause. Looking at the timestamps, this is well
over a minute.

Why the pause? Well, none of this memory shuffling is cheap. To make
things worse, these minor collections are also Stop The World (STW)
events: this means that the application was completely stopped.

Let's look at the GC logs to understand the details. This is the
relevant log for the event.

	1 	2016-03-15T20:00:26.471+0100: 8079.342: [GC pause (G1 Evacuation Pause) (young)
	2 	Desired survivor size 593494016 bytes, new threshold 15 (max 15)
	3 	- age   1:      60136 bytes,      60136 total
	4 	- age   2:       4312 bytes,      64448 total
	5 	- age   3:       3864 bytes,      68312 total
	6 	- age   4:       3784 bytes,      72096 total
	7 	- age   5:       3784 bytes,      75880 total
	8 	- age   6:       7568 bytes,      83448 total
	9 	- age   7:       3784 bytes,      87232 total
	10	- age   8:       3784 bytes,      91016 total
	11	- age   9:       3784 bytes,      94800 total
	12	- age  10:       7568 bytes,     102368 total
	13	- age  11:       3784 bytes,     106152 total
	14	- age  12:       3784 bytes,     109936 total
	15	- age  13:       3784 bytes,     113720 total
	16	- age  14:       7568 bytes,     121288 total
	17	- age  15:       3784 bytes,     125072 total
	18	 (to-space exhausted), 80.1974703 secs]
	19	[Parallel Time: 77390.8 ms, GC Workers: 18]
	20	   [GC Worker Start (ms): Min: 8079342.1, Avg: 8079342.2, Max: 8079342.2, Diff: 0.1]
	21	   [Ext Root Scanning (ms): Min: 0.2, Avg: 0.3, Max: 0.4, Diff: 0.1, Sum: 5.2]
	22	   [Update RS (ms): Min: 155.4, Avg: 267.9, Max: 567.9, Diff: 412.4, Sum: 4821.3]
	23	      [Processed Buffers: Min: 2, Avg: 7.3, Max: 14, Diff: 12, Sum: 131]
	24	   [Scan RS (ms): Min: 0.1, Avg: 0.4, Max: 3.1, Diff: 3.0, Sum: 6.3]
	25	   [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.1, Diff: 0.1, Sum: 0.1]
	26	   [Object Copy (ms): Min: 76814.9, Avg: 77116.1, Max: 77231.5, Diff: 416.5, Sum: 1388089.1]
	27	   [Termination (ms): Min: 0.0, Avg: 5.9, Max: 7.2, Diff: 7.2, Sum: 106.6]
	28	   [GC Worker Other (ms): Min: 0.0, Avg: 0.1, Max: 0.2, Diff: 0.2, Sum: 2.1]
	29	   [GC Worker Total (ms): Min: 77390.4, Avg: 77390.6, Max: 77390.8, Diff: 0.3, Sum: 1393030.8]
	30	   [GC Worker End (ms): Min: 8156732.7, Avg: 8156732.8, Max: 8156732.9, Diff: 0.2]
	31	[Code Root Fixup: 0.0 ms]
	32	[Code Root Migration: 0.3 ms]
	33	[Code Root Purge: 0.0 ms]
	34	[Clear CT: 1.6 ms]
	35	[Other: 2804.7 ms]
	36	   [Evacuation Failure: 2778.0 ms]
	37	   [Choose CSet: 0.0 ms]
	38	   [Ref Proc: 5.6 ms]
	39	   [Ref Enq: 0.0 ms]
	40	   [Redirty Cards: 10.0 ms]
	41	   [Free CSet: 1.5 ms]
	42	[Eden: 9044.0M(9044.0M)->0.0B(320.0M) Survivors: 4096.0K->1132.0M Heap: 16.2G(18.0G)->14.8G(18.0G)]
	43    [Times: user=77.62 sys=2.54, real=80.20 secs]

There is a log of information there, but let's focus on the key relevant
points for our example.

Lines 1-18 give a summary of the event enclosed in brackets `[GC Pause
..`.  The first two lines explain that the event was triggered due to
"G1 Evacuation pause (young)", meaning that the Young Gen filled up and
needed to be cleaned up.  I won't go into the survivor spaces in this
post, but notice how lines 3-17 report the distribution of ages for
objects in the Young Gen.  Line 18 `(to-space exhausted), 80.1974703
secs]` indicates that the target space (Old Gen) is not big enough to
hold all the objects that need to be moved. The full event takes
80.1974703s.

Line 19 explains that this collection used 18 threads and took over 77s.

    19  [Parallel Time: 77390.8 ms, GC Workers: 18]

Lines 20-30 cover how long it took to execute the different phases of
the collection. Most times went to Object Copy:

	26     [Object Copy (ms): Min: 76814.9, Avg: 77116.1, Max: 77231.5, Diff: 416.5, Sum: 1388089.1]

Min/Max/Avg are for an individual thread. The Sum is the aggregate for
all worker threads (hence, Sum = N threads * Avg). All 18 threads are
spending 77s on average simply to copy objects around.

We also have a non-negligible times in other phases. Line 22 reveals
that almost 5s aggregated were spent in the Update RS phase.

	22	   [Update RS (ms): Min: 155.4, Avg: 267.9, Max: 567.9, Diff: 412.4, Sum: 4821.3]

RS stands for [Remembered
Set](https://docs.oracle.com/javase/9/gctuning/garbage-first-garbage-collector-tuning.htm#GUID-A0343B53-A690-4DDE-98F9-9877096DBF0F),
an internal datastructure kept by the JVM assocaited to each region.
Its purpose is to track inbound from other regions.  Whenever the
application updates references (e.g. a field in an object), the JVM
takes care of updating the RS. For efficiency reasons these updates are
buffered and processed concurrently with the application. However, when
the JVM starts a collection the JVM holds updates in this buffer while
GC workers perform an initial scan of the heap (Ext Root Scan at L21)
looking for live objects. Only when this is complete the RS updates in
the buffer are drained.  It looks like this step is taking significant
time. We'll try to explain possible causes later.

Finally, we spot one more phase with significant times:

	36     [Evacuation Failure: 2778.0 ms]

This line indicates that the JVM wasn't able to move any more objects to
Old Gen and instead kept objects in their current location, reassigning
the corresponding regions to the Old Gen.

I'm not clear on why that time is so high (>2s) as it's meant to be [a
relatively fast
operation](https://docs.oracle.com/javase/9/gctuning/garbage-first-garbage-collector.htm#JSGCT-GUID-BE157AF6-29E7-461A-82CF-50C1978785DA).
From [what I could
investigate](https://blogs.oracle.com/poonam/understanding-g1-gc-logs)
(search for EVAC-FAILURE), the **Evacuation Failure** event occurrs when
a single object in a region can't be moved out of it because there is no
space left on the destination. If the JVM is simply reclassifying the
source region as Old Gen the 2.7s still seem too high. My best
hypothesis is that the JVM is reacting to this in one of these ways:
* Compacting or resizing regions. The [region
  size](http://www.oracle.com/technetwork/articles/java/g1gc-1984535.html)
  is a power of 2 between 1MB and 32MB, but the JVM aims at having ~2048
  regions. The evacuation failures may be prommpting the JVM to compact
  / resize regions, causing more objects to move around.
* The JVM rollbacks the move of other objects that were already copied
  out of this region.

From one oft he articles linked above I learned that it would've helped
to use `-XX:+G1PrintHeapRegions` to see more detailed events on each
region. Good to know for the next one!

Moving on. Line 42:

	42	[Eden: 9044.0M(9044.0M)->0.0B(320.0M) Survivors: 4096.0K->1132.0M Heap: 16.2G(18.0G)->14.8G(18.0G)]

Eden was completely emptied (9044MB-> 0), and it was also shrunk from
9044MB to 320MB.  Some of the objects evacuated from Eden were not yet
over the tenuring threshold and remained in the Young Gen within the
survivor space, which now holds 1132MB. The total heap size remains at
18GB, and usage went from 16.2GB -> 14.8GB. Overall, of the 9GB
evacuated from Young Gen, 2 were released, 1 remained in the survivor
space, and 6 were moved over to Old Gen.

This would've been useful to spot a more verbose description of the
resizing decisions taken by the JVM.

    -XX:+PrintAdaptiveSizePolicy

Unfortunately it was disabled at the time that the log was collected.

Finally, line 43 summarizes the times.

	43    [Times: user=77.62 sys=2.54, real=80.20 secs]

"User" here is CPU time spent by the JVM. "sys" is time in OS calls
or waiting for system events. "Real" is clock time where the application
was stopped. So, to summarize:

1. 18 threads moved ~6GB worth of objects.
2. The full copy took about 77s for each thread.
3. Update RS took in aggregate 5s.
4. Evacuation Failures consumed over 2s.
5. Real pause time was 80s, with >2s for sys and 77 for user.

## The problem with premature promotions

The most obvious observation is that we're running into trouble because
we're promoting objects prematurely from the Young Gen to the Old.  This
causes useless copies to Old Gen, region resizes, etc.  Promoting
soon-to-be-dead objects is not desirable because collecting objects from
the Old Gen risks triggering a Full GC, which is [more
expensive](https://docs.oracle.com/javase/9/gctuning/garbage-first-garbage-collector-tuning.htm#JSGCT-GUID-0DD93225-0BCF-4605-B365-E9833F5BD2FC)
than in Young Gen.

The long term effect is even worse.

TODO: enter in the point that objects accumulate in old gen


If
those objects were left in Young Gen for a bit longer, they'd lose
all references.  We want to stay in the area with see-saw pattern.

1. Generage less garbage.
2. 

- Try to hit a soft spot in region sizes to remain in the left hand side
  pattern. This is however a fragile equilibrium, and hard to achieve in
  non trivial applications.
- Zero allocations.

## Why those huge pause times?

There are several awkward points here.

(1) + (2) imply that each thread's throughput was a tiny 4.3MB/s. Just
for the sake of comparison, the same application running in a similar
hardware had GC logs was making 407 MB/s. Regardless of the
theoretical JVM throughput, a 100x difference indicates that
something is impacting the performance of GC worker threads.  It's
not about the amount of garbage we're generating.

LinkedIn has a
[couple](https://engineering.linkedin.com/garbage-collection/garbage-collection-optimization-high-throughput-and-low-latency-java-applications)
of
[posts](https://engineering.linkedin.com/blog/2016/02/eliminating-large-jvm-gc-pauses-caused-by-background-io-traffic)
touching on some common causes, also mentioned in the [Oracle
documentation](https://docs.oracle.com/javase/9/gctuning/garbage-first-garbage-collector-tuning.htm#JSGCT-GUID-4914A8D4-DE41-4250-B68E-816B58D4E278).
* The JVM is fighting for CPU time with other processes in the system.
* The JVM is allocating memory pages.
* The JVM is returning memory pages to the OS, which swaps them out to
  disk making later accesses more expensive.
* GC workers may be recruited by the OS for disk flushes, hitting them
  with syscalls and waits in the IO stack.

I believe that we could discard the second because the JVM pre-allocated
the full heap capacity up front, so it's unlikely that we're in this
case.  The swap hypothesis may be coming into play, but having `sys` time
at 2s doesn't really explain the 77s in user land.

The main hypothesis I would try to confirm first are whether the system
is overloaded, and whether we're having heavy I/O on the disk where logs
are being written to.

We do find later in the logs some events that reinforce the overloaded
option:

    41034    [Eden: 804.0M(804.0M)->0.0B(804.0M) Survivors: 116.0M->116.0M Heap: 4585.4M(18.0G)->4620.3M(18.0G)]
    41035  [Times: user=7.52 sys=0.25, real=131.86 secs]

As seen here, the `sys` time is negligible, and instead GC threads are
spending a lot of time idle waiting for CPU time.

---

Quoting from [Monica Beckwith](http://www.oracle.com/technetwork/articles/java/g1gc-1984535.html)
"Avoid explicitly setting young generation size with the -Xmn option or
any or other related option such as -XX:NewRatio. Fixing the size of the
young generation overrides the target pause-time goal.""



Fat_1 at 2016-02-25T11:58:06.827+0800

