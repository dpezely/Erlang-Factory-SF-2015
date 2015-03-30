Slides from Erlang Factory San Francisco 2015
=============================================

Slides are in Markdown, suitable for use with [remark](http://remarkjs.com/)
[v0.10.2](https://github.com/gnab/remark/releases/tag/v0.10.2).

Mirrored from:
<http://www.erlang-factory.com/sfbay2015/daniel-pezely>

## LDB: 10x Performance Increase After Rewriting Linked-In C Module In Pure-Erlang

In previous Erlang Factory talks, Jon Vlachogiannis presented BugSense's LDB involving Erlang, Lisp & C. This is the next iteration performed by someone else, yet talk assumes no prior knowledge of the project.

An experienced programmer who was new to Erlang adopted this project under guidance from its original author. After initially maintaining Lisp and C parts, within four months rewriting the pure-Erlang version, LDB reached 10x performance increase. We attribute this increase to a few reasons: 

1. Dropping the linked-in module meant less interference for Erlang scheduler
2. Reached 85% test coverage via Eunit within four months and >90% one month later; therefore, we moved quickly with confidence
3. Our implementation of Scheme was parsed, not interpreted or compiled (fixable but seemed like a tangent)
4. Our implementation of Scheme lacked garbage-collection.  We would "Let it Crash" hard, and BEAM runtime would come back within milliseconds thus not really a problem.

The talk then is about our experience identifying and measuring actual point of performance issues with old system, making an informed decision about what to fix versus what to ignore, and covering design & implementation of the new system.

Talk objectives:

1. Insights helpful to others weighing pros/cons of Linked-in modules
2. Convey experiences of an experienced developer picking up Erlang

Target audience:

- Those considering Erlang, coming from C, Lisp, Python, etc.
- Those considering Linked-in modules
- Those involved in very high traffic scenarios*

[*] Traffic into our cluster is well over a billion update messages per day.  While relatively small messages, traffic pattern from millions of mobile devices daily appears to our service provider as a DDoS attack

## Video

[on YouTube](https://www.youtube.com/watch?v=l6wXRmjI0RU&index=31&list=PLWbHc_FXPo2h0sJW6X2RZDtT1ndw6KKpQ)

## About Daniel

Daniel has been active for decades in the industry: Main Street, Wall Street, Silicon Valley, and places in between. Long ago, did near-realtime and distributed systems the hard way (C and C++). Common Lisp was his primary language for high-performance spanning six years prior to joining Splunk. After Splunk acquired BugSense, Daniel joined that group and learned Erlang.
