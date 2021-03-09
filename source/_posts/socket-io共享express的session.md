---
title: socket.io共享express的session
date: 2021-03-09 17:41:23
tags: ['学习笔记', 'web', 'javascript']
---



## 开篇

**[socket.io](https://socket.io/)获取[express框架](http://expressjs.com/)下的session方式有很多**

**这里介绍的是借助express-session中间件的方法**

<br>



## 示例

* **app.js**

  ```javascript
  var http = require('http');
  var express = require('express');
  var cookieParser = require('cookie-parser');
  var session = require('express-session');
  // session中间件
  var sessionMiddleware = session({
      secret: "anything.;scuHJG",
      resave: false,
      saveUninitialized: true,
      cookie: { maxAge: 4 * 7 * 24 * 60 * 60 * 1000 },
  });
  
  var app = express();
  app.use(express.json());
  app.use(express.urlencoded({ extended: false }));
  app.use(cookieParser());
  app.use(sessionMiddleware);
  
  // 创建session
  app.get('/index', (req, res) => {
      req.session.test = { name: 'test session'}
      res.send(`
        <!DOCTYPE html>
        <html>
        <head>
          <title>Test session</title>
          <script src="https://cdn.socket.io/4.0.1/socket.io.min.js">
          </script>
        </head>
        <body>
          <h1>Test page</h1>
          <script>
            fetch('/session')
            const socket = io();
            socket.emit('session')
          </script>
        </body>
        </html>  
  	`)
  })
  // 获取session
  app.get('/session', (req, res) => {
      console.log('session in express', req.session.test);
      res.status(200).send();
  })
  
  app.set('port', 39999);
  var server = http.createServer(app);
  
  // 创建 socket.io
  var socketApp = require('socket.io')(server);
  
  // 使用session中间件
  socketApp.use((socket, next) => {
      sessionMiddleware(socket.request, {}, next);
  });
  
  // 正常监听
  socketApp.on('connection', socket => {
      socket.on('session', data => {
          // 获取session
          console.log('session in socket.io', socket.request.session.test)
      })
  })
  
  server.listen(39999)
  server.on('listening', () => {
      console.log('listen ', server.address());
  });
  ```

<br>



## 结尾

这里的方法实际上就是，把定义好的express-session中间件，再给socket.io使用，从而挂载到socket.request上。