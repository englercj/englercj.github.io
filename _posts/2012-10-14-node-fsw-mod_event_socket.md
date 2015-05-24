---
layout: post_simple
title:  Node.js, FreeSWITCH, and mod_event_socket
date:   2012-10-17 17:17:00

excerpt: >
    Recently I had been tasked with learning more about Telephony and [FreeSWITCH](http://freeswitch.org/);
    specifically how to send/receive text messages. After reading around on the
    [FreeSWITCH Wiki](http://wiki.freeswitch.org/wiki/Main_Page) I found a module in FreeSWITCH called `mod_event_socket`.

tags:
- programming
- node.js
- javascript
---

Recently I had been tasked with learning more about Telephony and [FreeSWITCH](http://freeswitch.org/);
specifically how to send/receive text messages. After reading around on the
[FreeSWITCH Wiki](http://wiki.freeswitch.org/wiki/Main_Page) I found a module in FreeSWITCH called `mod_event_socket`.

This module allows programs to connect to FreeSWITCH via a TCP Socket API and control FreeSWITCH. The big
benefit here being is that you can listen for events coming through and apply logic and perform actions that
maybe a dialplan can't do (unlikely), or are easier to accomplish in your favorite language (quite likely).
On the mod_event_socket wiki page they had a nice list of all the supported languages, and there are quite a bit,
including my favorite Node.js. Unfortunately, the module linked there doesn't actually implement the ESL Interface
and I encountered a few issues trying to use it. I really wanted to use ESL, but I really wanted to do it from node 
s well...so I wrote my own.

Enter [modesl](https://github.com/englercj/node-esl), my own ESL Implementation for Node.js. This is a pure JavaScript
library built to interact with FreeSWITCH on a low-level using the ESL specification. With this library you can bring
the power of Node and FreeSWITCH together! It actually turns out to be a good fit as well, Node is evented and integrates
nicely into FreeSWITCH's event system. But enough talk, lets see some code:

```js
var esl = require('modesl'),
conn = new esl.Connection('127.0.0.1', 8021, 'ClueCon', function() {
    conn.api('status', function(res) {
        //res is an esl.Event instance
        console.log(res.getBody());
        process.exit(0);
    });
});
```

This example connects to FreeSWITCH, sends the `status` api command, and then logs the result; pretty simple. Here is a
line by line breakdown, and what each is doing behind the scenes:

- `conn = new esl.Connection('127.0.0.1', 8021, 'ClueCon', function() {`
 * modesl connects to FreeSWITCH, authenticates, subscribes to ALL events, then calls the callback.
- `conn.api('status', function(res) {`
 * modesl writes the status api command to the Event Socket and waits for a message back with `Content-Type: 'command/reply'`.
 * It then passes this event to the callback.
- `console.log(res.getBody());`
 * since res is an instance of `esl.Event` we just log the body

Some readers familiar with Node.js probably noticed that the `esl.Connection` object takes the params `(host, port, password[, cb])`;
even though convention in Node.js is `(port[, host][, cb])`. This is because the entire point of the library was to be compatable with
the ESL Specification (found [here](http://wiki.freeswitch.org/wiki/Event_Socket_Library#ESLconnection_Object)), which calls for it to be this way.

You can read more about the module in it's [README](https://github.com/englercj/node-esl#readme) on GitHub, and in the
[Documentation](https://github.com/englercj/node-esl/wiki), there are also a couple of [examples](https://github.com/englercj/node-esl/tree/master/examples) you can try out.

-----

For the FreeSWITCH nerds out there, I will be giving a quick talk about the module on October 17th 2012 on the weekly FreeSWITCH Conference Call.
