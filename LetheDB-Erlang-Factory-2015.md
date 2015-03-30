class: center, middle

LDB ~ Lethe DB
==============
LDB: 10x Performance Increase After Rewriting Linked-In C Module In Pure-Erlang
===============================================================================

or

Experiences With/Without Erlang Linked-in C Modules In A High Traffic Environment
=================================================================================

![Splunk logo](Splunk-logo-white.png)  
[Daniel Pezely](https://linkedin.com/in/dpezely)  
[Splunk Mobile Intelligence](https://splunk.com/mint)  
(formerly BugSense)

[Erlang Factory San Francisco](http://www.erlang-factory.com/sfbay2015)  
26-27 March 2015

---
class: center, middle

**The Far Side**, by *Gary Larson*

![an alien holds a jar containing humans, and the other says...](farside-holes-in-lid.jpg)

"Now Gorak! This time remember  
to punch holes in the lid!"

---
## Brief Background

BugSense: tracking crashes and events from mobile devices

- Founded September 2011 in Athens, Greece
- Acquired September 2013 by Splunk
- Re-branded in 2014 as Splunk Mobile Intelligence (MINT)

**The secret sauce for BugSense has always included LDB**

LDB is the Lethe Database: (pronounced LEE-thee)

- Functions as a *stream processor*
- Datastore tracks patterns and counts uniques:
    + Most are strictly increasing counters
    + Some read-modify-write operations
- Handles small updates from millions of mobile devices daily
- i.e., traffic pattern easily mistaken for DDoS attack

Earlier LDB presentations by Jon Vlachogiannis, co-founder of BugSense:

- Erlang Factory SF 2013, 2014 {[1]}
- HighScalability.com 2012 {[2]}

[1]: <http://www.erlang-factory.com/sfbay2014/jon-vlachogiannis>

[2]: <http://highscalability.com/blog/2012/11/26/bigdata-using-erlang-c-and-lisp-to-fight-the-tsunami-of-mobi.html>

---
## Presenter Background

Relevant early experience to shape a career:

Virtual Reality R&D, circa 1990

- Designed for massive number of participants; i.e., MMOG
- Large-scale distributed systems
- Building for "mutual, contradictory realities"


                   After modeling reality and
        Designing systems to sustain very high throughput,
            Everything since then seems like a subset!

Other experience:

- Main Street, Wall Street, Silicon Valley, and places in between
- Joined Splunk in January 2013 and BugSense group in 2014
- C and Lisp since 1987, Common Lisp since 2005, Python since 1999 
- **Started using Erlang as of Erlang Factory SF 2014**

---
## Measured In Production Environment

Cluster receives many **billions** of requests *per day*

Each LDB node at Amazon Web Services (AWS)

- [c4.4xlarge](https://aws.amazon.com/blogs/aws/now-available-new-c4-instances/) (16 core 30GiB Xeon Haswell)
- 30,000 customers potentially share one server (hence 30GiB RAM)
- Dedicated servers use same software, different config

**10x Performance Improvement**

- Was 200 requests/second
- Now 2,000+ requests/second

---
## LDB Use Cases ~ Purpose-Built Datastore

Our SDK resides on hundreds of millions of mobile devices,
each of which sends regular updates pertaining to:

- **Pings**
- **Crashes**
- **Events**
    + e.g., breadcrumbs dropped along visitor journey
- **Advanced Events**
    + customer defined key/value pairs
- **Network Performance**
    + SDK receives information from iOS, Android network calls
    + tracks API Exception and HTTP Response codes
    + tracks duration when successful
- **Transactions**
    + context is customer defined, as with transactions in Splunk
    + success/fail/cancel
    + tracks duration when successful

Customer access is only via Dashboard UI or Splunk App:  
i.e., LDB renders to JSON format

---
## Operational Criteria

Cluster receives *billions* of requests per day  
but use as few servers as reasonably possible

**Traffic is steady 24x7 from around the world**

- With this volume: queuing == death
- You'll never catch-up!

Prioritize for timely response back to each mobile device

Database must be highly available yet cheap & disposable:

- Serve from in-memory datastore on same node exclusively
- Platform must recover in milliseconds (and BEAM does!)
- Load customer data on-demand

Nature of data:

- Compound-keys mapped to strictly-increasing counters
- Probabilistic counters; e.g., HyperLogLog (HLL)
- Unordered sets, implemented as arrays

---
## Early LDB: Erlang, Lisp and C

**Erlang**

- Robustness, time to market, "Let it crash!"
- Returns to stable state within milliseconds
- Top-level code in Erlang
- Long-lived `port_command/2` for everything else

**Lisp** / **Scheme**

- Malleable language for queries
- Ideal for early stage start-up exploring problem & solution space

**C**

- Erlang linked-in module
- Implemented:
    + Scheme runtime
	+ Query language library
	+ Probabilistic counters
	+ hash tables
	+ Sets

---
## Earlier Architecture

One master process per customer API key:

- Spawn Erlang process on-demand as we see new API key
    + Owns port to linked-in C module
	+ No thread priority, no thread pool
	+ Inbound request to Cowboy spawns per-request worker
- Each process exits when idle

Datastore:

- Hash-tables implemented in C (local modifications to `uthash`)
- HyperLogLog (HLL) implemented in C
- Each customer gets trio of tables: key/value pairs, sets, HLL
- ETS table for Ops-related metrics; e.g., requests per second

Miscellaneous:

- Lethe Query Language (LQL) was Scheme R4 dialect of Lisp
- Upon expected full memory or unexpected issues: "Let it crash!"
- Recovery after crash is measured in milliseconds
- **High traffic customers may be throttled (sub-sampling)**

---
## Room For Improvement

**Problem:**

- Erlang servers "typically" measure *tens of thousands* requests per second
- Older LDB was merely *hundreds*

**Revelations:**

Queries are well understood now and may be specified precisely,  
so malleability of Lisp as query language (LQL) was no longer necessary.

While we *love Lisp*, we no longer need overhead of this C implementation.

**Premise:**

There were known limitations to the Scheme component:  
not compiled, not cached, no garbage collection, etc.

However, experience elsewhere suggested primary culprit might have been  
*misuse* of Compare-And-Swap (CAS) within a library...

---
## Bottlenecks within old LDB

Experimental branch for measurements:

- Eliminated parsing of query language on inbound requests
- Using only cheap counters (**no** service tier limits, etc.)
- Forked/rewrote linked-in driver C code for minimal code path
- Linked-in module returned control back to Erlang scheduler  
  within equivalent of a **few "reductions"**

Observed:

- **Only *marginal* increase in performance**

Initial Conclusions:

- Confirmed C implementation of query language runtime *not* culprit
- So-called "atomic" operations [*] implicated as primary bottleneck
- Compare-and-Swap (CAS) may be harmful on NUMA/many-cores

Course of Action:

- **Extend LDB core for pure-Erlang implementation**

[*] There are multiple definitions of "atomic" (more below)

---
## How Compare-And-Swap (CAS) Can Be Harmful

CAS operations *as simple replacement for locks* is **worse** than naïve,  
because naïve approaches are uninfluenced by locking methodology!

- Consider before using CAS family of operations:
    + On x86 since i486, CAS locks the bus (northbridge)
    + Negatively impacts entire NUMA Node (i.e., 4 CPUs x 16 cores)
    + Blocks *all* other memory operations for *hundreds* of CPU cycles {[3], [4]}
    + Compounded by cache miss, so ensure CAS isn't first access of address
- **Beware**:
	+ "Lock-free" is not necessarily *wait-free* {[5]}
    + CAS is "atomic" (transaction) in database sense: *all-or-nothing*
    + CAS is "atomic" (indivisible) as *one* line of Assembly: `LOCK CMPXCHG`
	+ (**Not** atomic in sense of occurring *instantaneously* or transparently)

More Background:

- CAS previously believed to be answer for "lock-free" mechanisms {[6]}
- CAS is very tempting for those new to deep concurrency
- CAS overused by various languages & libraries, so confirm source/docs
    + e.g., Look for `__sync_add_and_fetch()` in C source
	+ or better yet: `LOCK CMPXCHG` within disassembly dump on x86, x86-64
- CAS use cases are deeply intertwined with CPU cache lines, etc. {[7]}

[3]: http://www.azulsystems.com/blog/cliff/2011-11-16-a-short-conversation-on-biased-locking

[4]: http://danluu.com/new-cpu-features/

[5]: http://preshing.com/20120612/an-introduction-to-lock-free-programming/

[6]: http://people.csail.mit.edu/edya/publications/OptimisticFIFOQueue-journal.pdf

[7]: http://people.freebsd.org/~lstewart/articles/cpumemory.pdf

---
## Regarding Low-Hanging Fruit

Do these *last*:

- Each would likely bring **less than 2x-5x** performance improvement:
    + Thread pooling at level of Erlang port/driver
	+ `driver_async()` use non-NULL value for `key` param for thread reuse
    + LQL compiler
    + LQL caching
    + LQL eval with ephemeral memory allocation
	+ LQL garbage-collection
- If less than 10x, it automatically becomes *low priority*

Instead:

- **Strive for 10x, 100x, 1000x improvements**
- Otherwise wastes time compared to <strike>"Moore's Law"</strike> Adapteva *Parallella*
- Maybe cheat or punt:
    + Upgrade to next generation machine
	+ Upgrade to larger server(s)
	+ Introduce a load-balancer
	+ Terminate SSL via front-end component/hardware

---
## Revised Architecture

One master Erlang process per customer API key: (same as before)

- Spawn process on-demand as we see new API key
    + Owns ETS tables, manages snapshots, etc.
	+ Inbound RPC and/or Cowboy spawns per-request worker
- Each process exits when idle

Datastore: (complete overhaul)

- Uses ETS (not Mnesia) with `tab2file()` for snapshots
- Differentiates day of retention by database table (previously by row)
- *Lots* of ETS tables: 3 x 30,000 customers x Day of retention modulo
- Customers migrated out-of-band with respect to Erlang
- Database table naming uses modulo based upon days of retention

Miscellaneous: (much improved!)

- RPC between other systems (no HTTP beyond client connection)
- Supervisor uses `simple_one_for_one` (previously `one_for_one`)
- OTP: uses `gen_server` now, which wasn't in original LDB
- Eliminates need for Linked-in C modules
- **Accepts all data (no throttling, no sub-sampling)**

---
## What Went Right / Wrong / Could Be Better

**Went Right**

- Functional Programming (i.e., no side-effects)
- When new to any language, *test* all assumptions before starting feature
- Reached 85% code coverage in tests within 4 months (one person)
- Topped 90% coverage after another month-- asymptotic effort writing tests
- Learned to navigate docs tree (no search), and discovered other functions

**Went Wrong**

- Initial rewrite used Mnesia, but mailbox killed BEAM after 12 hours
- Mnesia-to-ETS migration hit architectural issue for our use cases:  
Required re-introducing bits of Mnesia; e.g., `wait_for_tables()` and synchronization for HLL's read-modify-write workflow
- Code reviews missed some Erlang idiom mistakes:  
With `gen_server`, create wrapper functions around all "public" messages, and export those functions as means to enforce our API

**Could Be Better**

- Yet to do releases in proper Erlang way, but we control our servers
- Yet to do hot code loading, but server bounce is painless

---
## Pains, Pitfalls & Other Points

- "`Why, Joe? Why?`" and taking the name of *Erlang* in vain  
   could be heard occasionally from Daniel's desk
    + **`stdlib`** has inconsistent/asymmetrical function names
	+ Some errors require **`try/catch`**, others are results from *expressions*
	+ Returning **`ok`** versus **`true`** versus **`{ok,Foo}`**, etc.
	+ Docs found under **`erts`**, **`kernel`**, **`stdlib`** may seem arbitrary at first
- Mnesia and ETS use atoms for table names
	+ Atoms are *never* garbage-collected within BEAM vm
    + LDB creates *lots* of database tables, thus potential problem
- Compiler directive macros can't be applied at granularity of an expression
    + It's fine for **`eunit`** type of scenario
	+ Can't have debug-only function clauses this way (use Guards instead)
- Constraints of syntax to a Common Lisp guy... (enough said)
    + Using **`when`** with *comma versus semicolon* seems fragile
    + Why not `Lisp Flavoured Erlang` (LFE) *???*
	    - Other in-house code already in Erlang (not LFE)
	    - Existing LDB code used Erlang R16
	    - LFE was still only `v0.7a` at the time; e.g.,  
		  colon operator didn't accommodate **`module:function`** yet

---
## Want 100x & Beyond?

Classic approaches-- **reducing contention**:

- Eliminate system calls that would cause OS to block BEAM process
- Avoid memory allocation/construction ("non-consing")
- Optimize queue or stack for one writer, one reader
- Use CPU affinity, thread-per-core, or unthreaded co-routines

**However, you would be doing things that Erlang does for you!**

Consider [c10k](http://www.kegel.com/c10k.html) / [c10m](http://c10m.robertgraham.com/p/manifesto.html) tricks such as user-land TCP/IP stack:

- FreeBSD libuinet, 2014 {[8], [9]} -- *shares networking code with OS*
- Geoff Cant's Enet, 2010 {[10], [11]}
- Erlang library from Javier Paris, et al, 2005 {[12]}
- Maybe port CMU's Foxnet from Standard ML, 1994, 1996 {[13]}

[8]: <https://github.com/pkelsey/libuinet>
[9]: <http://www.bsdcan.org/2014/schedule/attachments/260_libuinet_bsdcan2014.pdf>
[10]: <https://github.com/archaelus/enet>
[11]: <http://www.erlang-factory.com/conference/SFBay2010/speakers/geoffcant>
[12]: <http://www.erlang.org/workshop/2005/tcpip_presentation.pdf>
[13]: <http://www.cs.cmu.edu/~fox/foxnet.html>

---
## References

1. http://www.erlang-factory.com/sfbay2014/jon-vlachogiannis
2. http://highscalability.com/blog/2012/11/26/bigdata-using-erlang-c-and-lisp-to-fight-the-tsunami-of-mobi.html
3. http://www.azulsystems.com/blog/cliff/2011-11-16-a-short-conversation-on-biased-locking
4. http://danluu.com/new-cpu-features/
5. http://preshing.com/20120612/an-introduction-to-lock-free-programming/
6. http://people.csail.mit.edu/edya/publications/OptimisticFIFOQueue-journal.pdf
7. http://people.freebsd.org/~lstewart/articles/cpumemory.pdf
8. https://github.com/pkelsey/libuinet
9. http://www.bsdcan.org/2014/schedule/attachments/260_libuinet_bsdcan2014.pdf
10. https://github.com/archaelus/enet
11. http://www.erlang-factory.com/conference/SFBay2010/speakers/geoffcant
12. http://www.erlang.org/workshop/2005/tcpip_presentation.pdf
13. http://www.cs.cmu.edu/~fox/foxnet.html
