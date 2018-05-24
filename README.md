# http-proxy-stream

a proxy tool which damn convenient with stream pipe. also with 90+% test coverage. inspired by [request](https://github.com/request/request)

## features

* pipe http proxy
* modify response before pipe to destination stream
* cache request data and response data for later usage

# Install

    npm i --save http-proxy-stream

# API

### proxy(request, [options, response])

- **request**: `http.IncomingMessage` instance or other instance of request.
- **options**: `Object`, all options of http.request.
- **options.url**: `String`, the target url with protocal and search part. like http://www.google.com?search=foo
- **options.modifyResponse**: `Function(response)`, `Optional`,
  modify response before pipe to destination stream. you can change the **reponse.body** and **response.headers**.
  **response.body** have decoded to text, object, Buffer. the `this` keyword is reference of `request.Request` instance. you can return a promise.
- **options.skipModifyResponse**: `Function(response)`, return true will skip modifyResponse.
- **options.onResponse**: `Function(response)`, call once when http.ClientRequest emit 'response' event.
- **options.cache**: `Boolean|Function(response)`, default false, if true will cache resopnse data for later usage. if function will call with response as first argument, return true will cache response.
- **response**: `Optional`, writable stream, like http.ServerResponse instance.
- **Return**: `Promise`

### proxy.mime

#### proxy.mime.isText(mimeType)
the miem type mean the http body is text which need deconding with charset

#### proxy.mime.isJSON(mimeType)
`true` the content can been json stringified, `false` otherwise.


#### proxyReturn.contentType
a object parsed from http header 'content-type' field which like `{type: 'text/plain', charset: 'utf-8', parameters: {}}`

#### proxyReturn.contentEncoding
then value of 'Content-Encoding' header


# Usage

### Normal Proxy

```javascript
const proxy = require('http-proxy-stream');
const http = require('http');

// http useage
http.createServer((req, res) => {
  proxy(req, {
    url: `http://www.google.com${req.url}`,
    onResponse() {
      // A chance to rewrite response headers before pipe
      response.headers.test = 'test';
    }
  }, res);
}).listen(8000);

http.createServer((req, res) => {
  proxy(req, {
    url: `http://www.google.com${req.url}`,
    onResponse() {
      // A chance to rewrite response headers before pipe
      response.headers.test = 'test';
    }
  })
    .then(resp => resp.pipe(res))
    .catch(err => {
      res.writeHead(500);
      res.end(err)
    });
}).listen(8000);

// koa middleware
function* koaProxy(next) {
  this.body = yield proxy(this.req, {
    url: `http://www.google.com${this.req.url}`,
    onResponse(response) {
      response.headers.test = 'test';
    }
  });
}
```

### Modify Reponse Before Pipe

Some times we want modify the response of the backend services. you can define a function as `options.modifyResponse`.

```javascript
const proxy = require('http-proxy-stream');
const http = require('http');

// http useage
http.createServer((req, res) => {
  proxy(req, {
    url: `http://www.google.com${req.url}`,
    onResponse(response) {
      response.headers.test = 'test';
    },
    modifyResponse(response) {
      // change headers
      response.headers['content-type'] = 'application/json; charset: utf8';
      // use new content, can be string, buffer, null, undefined or object;
      response.body = {content: body};
      // can return a promise if necessary
      // return Promise.resolve();
    }
  }, res);
}).listen(8000);
```

### Skip Modify Reponse
modify response will cause unzip and charset decode, some response like downloading big file is bad performance. we can use `options.skipModifyResponse` to skip it;

```javascript
const proxy = require('http-proxy-stream');
const http = require('http');

// http useage
http.createServer((req, res) => {
  proxy(req, {
    url: `http://www.google.com${req.url}`,
    onResponse(response) {
      response.headers.test = 'test';
    },
    skipModifyResponse(response) {
      return proxy.mime.isText(response.contentType.type);
    },
    modifyResponse(response) {
      // change headers
      response.headers['content-type'] = 'application/json; charset: utf8';
      // use new content, can be string, buffer, null, undefined or object;
      response.body = {content: body};
    }
  }, res);
}).listen(8000);
```

### Get Data After Pipe
for debug reason, you may want log the response status or monitor the content.
but if we read content everytime that may be negative to performance.
so we can use `options.cache` for a chance to read data after http reponse finished.
options.cache can be a function or boolean, default is false.
let's see a example.

```javascript
http.createServer((req, res) => {
  proxy(req, {
    url: `http://www.google.com${req.url}`,
    cache(response) {
      // only when the content is text type, cache the data
      return proxy.isText(response.contentType.type);
    }
  }, res)
    .then(resp => {
      res.on('finish', () => {
        resp.resetReadable();
        let responseChunks = [];
        resp.on('data', (chunk) => {
          responseChunks.push(chunk);
        });
        resp.on('end', (chunk) => {
          chunk && responseChunks.push(chunk);
        });

        resp.reqCacheStream.resetReadable();
        let requestChunks = [];
        resp.reqCacheStream.on('data', (chunk) => {
          requestChunks.push(chunk);
        });
        resp.reqCacheStream.on('end', (chunk) => {
          chunk && requestChunks.push(chunk);
        });
      });
    });
}).listen(8000);
```


### Auto Redirect
autoSameOriginRedirect


# Develop

    $> npm i
    $> npm test
    $> npm publish

# Release Note
v1.2.1 2018-05-24
    * fix proxy https bug

v1.2.0 2018-05-17
    * modifyResponse support return promise

v1.1.0 2018-04-12
    * support https proxy

v1.0.3 2017-02-16
    * catch inner error

v1.0.2 2017-02-07
    * refactor readBody method of read-content.js

v1.0.1 2017-02-07

    * CacheStream add clearCahce method
    * tiny fix
    * README update

v1.0.0 2017-02-06

    * new api and new lib name

v0.1.3 2016-09-22

    * catch some errors and reject them

v0.1.2 2016-09-17

    * modifyResponse can set response.body to object, null and undefined.

v0.1.1 2016-09-17

    * fix some bugs in koa flow.

v0.1.0 2016-09-12

    * first version
