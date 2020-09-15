---
title: Debugging node with devtools
published: true
description: How to use the built in node inspector to debug your node applications, even across multiple processes
tags: [node,webdev,javascript,productivity]
---

**Fun fact**: *you don't necessarily have to just use console.log to debug your application, node has a built-in debugger that works with DevTools.*

What's great, is that this feature has been around since Node 6.3 (that's 4 years ago!). Prior to this there was a package that was around that let you do the same thing. Even still, you could use the console based debugger like many other languages let you. I'm going to show you how to use both.

## Running with `--inspect`

You can run any node script with the following flag.

```
node --inspect index.js
```

But if you're like me, you probably have a more complicated setup that involves `gulp` or some other system that forks a number of processes. When you run `nodemon` with `--inspect=5858` and your process ends up forking, the ports will increment 1 by 1. If this happens, and your debugging port is `5858` then the debug ports for the other processes will be `5859`, `5860`...etc.

## Chrome Inspector

Open up Chrome to `chrome://inspect` and you may notice the link with the following:

```
Open dedicated DevTools for Node
```

Click on that link and you'll get a DevTools inspector. All you need to do is go to the *Connection* tab and add the various debug ports to the connection list. For me, this was `localhost:5858`, `localhost:5859`, `localhost:5860`, 'localhost:5861', `localhost:5862`.

![DevTools Connections](/assets/8chevd5fdiww1plmmskb.png)

Once you've done that you are good to go. You may notice in your terminal that a debugger was attached.

![Chrome Node Inspector](/assets/21gwkio7vkpv2p81yrpn.png)

## Features

The great thing about the node inspector is that it comes with loads of features:

- Breakpoint debugging, stepping and blackboxing
- Source maps for transpiled code
- LiveEdit w/ hotspot evaluation
- Console evaluation
- Profiling and sampling
- Heap snapshots, allocation, memory profiling
- Async stacks/promises

There you go!

> *Bonus* If you ssh you can forward your debugging for a remote debugging session if you're trying to debug something running on a server.

```
ssh -L 9221:localhost:9229 user@remote.example.com
```

For more details check out the [documentation guide](https://nodejs.org/en/docs/guides/debugging-getting-started/)!

## Debugger Command Line

If you don't have access to chrome devtools and you can't do any ssh tunneling, you can also use the normal command line debugger. All you have to do is run:

```
node inspect 127.0.0.1:5858
```

You also have the option of attaching directly to a process id:

```
node inspect -p <process id>
```

![Command Line Debugger](/assets/u1mql83p2ngab265grwj.png)
