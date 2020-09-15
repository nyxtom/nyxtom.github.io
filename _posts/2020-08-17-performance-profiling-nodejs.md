---
title: Performance profiling in Node.js
published: true
description: We will take a look at the built-in node profiler to get an overview of how V8 and node is performing
tags: [webdev,nodejs,beginners,performance]
---

Node provides a built-in profiler with `--prof` that makes it **relatively** straightforward to pinpoint bottlenecks. We'll go over what to expect from the output, what a flame graph is, and how to properly setup a test scenario for optimizing application performance. I will go over scenarios that you might be running into and talk about strategies for knowing where to look when it comes to performance.

## Performance Bottlenecks

Let's start by first stating that you might not actually need to be do this. Never over optimize if you don't have to. At the same time, it's good to have a complete picture of what your application is doing under load scenarios that are bogging down. There's a number of areas that you can consider that have nothing to do with the code:

* System defined user limits
* Network utilization and latency
* Packet loss and dns issues
* Disk latency, write/read throughput
* Cache misses, page faults, table or collection scans
* Connection keep-alive issues, load balancing

I don't claim to be an expert in any of these areas, but what I can tell you is that it is usually a decent chance that your problem will be in one of these areas *before* you have to go an optimize your code (let alone decide on an entirely different language or framework).

In fact, the entire networking stack itself is a lot more tunable and complicated than you might even imagine at first. It's a good idea to get an idea of what's happening in the network stack before you decide your code is the problem:

> ["Illustrated Guide to Monitoring and Tuning the Linux Networking Stack: Receiving Data"](https://blog.packagecloud.io/eng/2016/10/11/monitoring-tuning-linux-networking-stack-receiving-data-illustrated/)

## Profiling Node.js

When you reach the point where you are running out of options and it's time to start profiling your code base for bottlenecks - take a look at `--prof`.

> The built in profiler uses the [profiler in V8](https://v8.dev/docs/profile) which samples the stack at regular intervals during program execution. It records the results of these samples, along with important optimization events such as jit compiles, as a series of ticks.

```
code-creation,LazyCompile,0,0x2d5000a337a0,396,"bp native array.js:1153:16",0x289f644df68,~
code-creation,LazyCompile,0,0x2d5000a33940,716,"hasOwnProperty native v8natives.js:198:30",0x289f64438d0,~
code-creation,LazyCompile,0,0x2d5000a33c20,284,"ToName native runtime.js:549:16",0x289f643bb28,~
code-creation,Stub,2,0x2d5000a33d40,182,"DoubleToIStub"
code-creation,Stub,2,0x2d5000a33e00,507,"NumberToStringStub"
```

With any script you can run the following:

```
NODE_ENV=production node --prof script.js
```

If you are happening to run multiple processes (from process forking) you will see that the output will include the process ids along with the ticks. You will see the files `isolate-0xnnnnnnnnnnnn-v8.log` outputted once you stop the script.

## Making sense of `--prof` with `--prof-process`

To make any sense of this you'll need to run:

```
node --prof-process isolate-0xnnnnnnnnnnnn-v8.log > processed.txt
```

This will give you a brief summary of tick percentages by language followed by individual sections per language identifying hotspots in the codebase.

```
 [Summary]:
   ticks  total  nonlib   name
     79    0.2%    0.2%  JavaScript
  36703   97.2%   99.2%  C++
      7    0.0%    0.0%  GC
    767    2.0%          Shared libraries
    215    0.6%          Unaccounted

 [C++]:
   ticks  total  nonlib   name
  19557   51.8%   52.9%  node::crypto::PBKDF2(v8::FunctionCallbackInfo<v8::Value> const&)
   4510   11.9%   12.2%  _sha1_block_data_order
   3165    8.4%    8.6%  _malloc_zone_malloc
...
```

At times you may find the output a bit difficult to understand and that's alright. If you spend time to understand what sort of problem you are trying to solve, you might be able to narrow down the issue.

What I mean by this is, reduce your search space of the problem. Sometimes when I think I'm running into a performance bottleneck I make attempts to reduce any variables that might be getting in the way of understanding what kind of bottleneck I really have. I do this by eliminating as much as possible (turning off various streams, conditional branches..etc) and re-run my performance test.

One example of where I've run into this is in stream processing. I will often turn off as much as I can, run the performance test and compare results to see if my usage can be optimized. It takes a combination of intuition about what your code is doing and these sort of tests to make progress.

## Conclusion

If you are doing any kind of performance profiling I highly recommend you have the `--prof` in your toolbelt. Take a look at guide in the [node.js documentation](https://nodejs.org/en/docs/guides/simple-profiling/) for more details.
