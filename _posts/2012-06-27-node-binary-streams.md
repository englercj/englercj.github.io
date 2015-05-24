---
layout: post_simple
title:  Node.js Binary HTTP Streams
date:   2012-06-27 5:59:00

tags:
- programming
- node.js
- javascript
---

<div class="alert alert-info" role="alert">
    <strong>Note:</strong> This article is about old versions of both node and express, and it was a dumb way of doing things.
    It is only here for historical purposes.
</div>

Recently I implemented a app in Node.js that served statistics data to remote relays. I decided to roll with HTTP to keep it simple.
Using [expressjs](http://expressjs.com/) I quickly threw up my server, and found that express
allows you to pass a buffer as the response. Here is something like what I did:

```js
//includes
var express = require('express'),
crypto = require('crypto'),
app = express.createServer();

//in the real app this area runs many times (~1 /sec)
var pack = {},
md5 = crypto.createHash('md5');

pack.data = new Buffer('some binary data', 'binary');
pack.size = pack.data.length;
md5.update(pack.data);
pack.sum = md5.digest('hex');

//The routes to serve the data
app.get('/ask/:md5', function(req, res) {
    res.json({
        need: req.params.md5 != pack.sum,
        size: pack.data.length,
        md5: pack.sum
    });
});

app.get('/package', function(req, res) {
     res.send(pack.data);
});
```

So basically I would do my calculations, form this package, and serve it. The `'/ask/:md5'` route was for a remote server to ask if it
needed a new package, if their md5s matched then the package hadn't changed and there is no need to download. However, if they didn't
match (ie, need was true) then the remote would store the new md5 and size that is expected to get when it finished downloading the
package using the `'/package'`  route. All of this was pretty simple, but what I
didn't expect came when I wrote the remote side of this.

Generally when I download something via http in Node.js I do something like the following:

```js
var http = require('http'),
url = require('url');

http.get(url.parse('http://myserver.com:9999/package'), function(res) {
    var data;

    res.on('data', function(chunk) {
        if(!data) data = chunk;
        else data += chunk;
    }).on('end', function() {
        console.log(data);
    });
});
```

Now this all works well and great when your content is some text like HTML, json, or whatever else. But when this is a
`Content-Type: application/octet-stream`, which is what express returned for my binary buffer download, you will get
seriously weird results. Specifically my app would rehash this download to make sure it matched the md5 and size that it
got before when it hit the ask URL; and they never matched.

Eventually what I found out is that the `chunk` sent to each data event was actually a Buffer instance. Normally the above code works
fine because the Buffer is converted to a UTF8 string and *then* concatenated. The `+=` operator does not concatenate buffers, but I
thought it did. After some research I wrote the following to track the chunk buffers and then concatenate them together into one buffer
after the stream ended:

```js
var http = require('http'),
url = require('url');

http.get(url.parse('http://myserver.com:9999/package'), function(res) {
    var data = [];

    res.on('data', function(chunk) {
        data.push(chunk);
    }).on('end', function() {
        //at this point data is an array of Buffers
        //so we take each octet in each of the buffers
        //and combine them into one big octet array to pass to a
        //new buffer instance constructor
        var buffer = new Buffer(data.reduce(function(prev, current) {
            return prev.concat(Array.prototype.slice.call(current));
        }, []);
        console.log(buffer.toString('base64'));
    });
});
```

Now if I did my md5 and size checks on the new buffer I created in the end event, they all matched up perfectly.

-----

**Edit 10/11/2012**

Recently I have noticed that my previous solution of having 2 requests (1 to ask, the other to download) causes a race condition.
Since the code generating the package runs every second, it is not uncommon for the package to change between requests. This causes
the receiving node to think the package is corrupt, when really it is just out of date. I have fixed the issue by combining the process
into a single request, where the md5 is passed via headers:

```js
//includes
var express = require('express'),
crypto = require('crypto'),
app = express.createServer();

//in the real app this area runs many times (~1 /sec)
var pack = {},
md5 = crypto.createHash('md5');

pack.data = new Buffer('some binary data', 'binary');
pack.size = pack.data.length;
md5.update(pack.data);
pack.sum = md5.digest('hex');

//The route to serve the data
app.get('/package/:md5', function(req, res) {
    if(pack.sum != req.params.md5) {
        res.header('x-package-md5', pack.sum);
        res.send(pack.data); //200 OK
    } else {
        res.send(); //204 No Response
    }
});
```

```js
var http = require('http'),
url = require('url'),
lastMd5 = 0;

http.get(url.parse('http://myserver.com:9999/package/' + lastMd5), function(res) {
    if(res.statusCode === 204) return;

    lastMd5 = res.headers['x-package-md5'];

    var data = [];

    res.on('data', function(chunk) {
        data.push(chunk);
    }).on('end', function() {
        //at this point data is an array of Buffers
        //so we take each octet in each of the buffers
        //and combine them into one big octet array to pass to a
        //new buffer instance constructor
        var buffer = new Buffer(data.reduce(function(prev, current) {
            return prev.concat(Array.prototype.slice.call(current));
        }, []);
        console.log(buffer.toString('base64'));
    });
});
```

With this code, the request just returns 204 with an empty response if no package is needed. Otherwise you get a 200, with the full package, and the new MD5 and a header.

-----

**Edit 10/17/2012**

Here is an even better way to concat the client buffers:

```js
var http = require('http'),
url = require('url'),
lastMd5 = 0;

http.get(url.parse('http://myserver.com:9999/package/' + lastMd5), function(res) {
    if(res.statusCode === 204) return;

    lastMd5 = res.headers['x-package-md5'];

    var data = [];

    res.on('data', function(chunk) {
        data.push(chunk);
    }).on('end', function() {
        //at this point data is an array of Buffers
        //so Buffer.concat() can make us a new Buffer
        //of all of them together
        var buffer = Buffer.concat(data);
        console.log(buffer.toString('base64'));
    });
});
```

You can also create a single buffer using `Content-Length` as the size, then copy each chunk into the buffer on each data event.
