title: flask-socketio 项目部署

把使用了flak-SocketIO的flask项目部署在 Apache + mod_wsgi 上面，总是阻塞，无法正常访问。

网搜后再这里找到了原因：[Flask-SocketIO Not Working On Apache/WSGI](https://stackoverflow.com/questions/29921671/flask-socketio-not-working-on-apache-wsgi) 

Apache 的wsgi服务器无法使用socketio，Gunicorn则可以使用。

### 查看flask-socketio文档的部署部分：

部署Flask-SocketIO服务器有很多选择，方法有简单的也有及其复杂的。 在本节中，将介绍最常用的选项。

1. 嵌入式服务器

   最简单的部署策略是安装eventlet或gevent，并通过调用socketio.run（app）来启动Web服务器。 无论哪个安装，这将在eventlet或gevent Web服务器上运行应用程序。请注意，在安装eventlet或gevent时，使用socketio.run（app）运行的服务器是可以用于生产的服务器。 如果这两个都没有安装，那么应用程序将运行在Flask的开发Web服务器（*werkzeug* 默认是单线程）上，这不适合于生产使用。不幸的是，uWSGI和gevent一起使用时，此选项不可用。 有关此选项的信息，请参阅下面的uWSGI部分。

2. Gunicorn Web服务器
   socketio.run（app）的替代方法是使用gunicorn作为Web服务器，使用eventlet或gevent作为worker。 对于这个选项，除了gunicorn之外，还需要安装eventlet或gevent。 通过gunicorn启动eventlet服务器的命令行是：

   ```
   gunicorn --worker-class eventlet -w 1 module:app
   ```

   如果您更喜欢使用gevent，则启动服务器的命令是：

   ```
   gunicorn -k gevent -w 1 module:app
   ```

   当gunicorn使用gevent worker并且使用gevent-websocket提供的WebSocket支持时，必须更改启动服务器的命令以选择支持WebSocket协议的自定义gevent Web服务器。 修改后的命令是：

   ```
   gunicorn -k geventwebsocket.gunicorn.workers.GeventWebSocketWorker -w 1 module:app
   ```

   在所有这些命令中，module是定义应用程序实例的Python模块或包，而app是应用程序实例本身。

   Gunicorn版本18.0是Flask-SocketIO推荐的版本。 已知19.x版在包含WebSocket的某些部署方案中具有不兼容性。

   由于gunicorn使用有限的负载均衡算法，因此使用此Web服务器时不可能使用多个工作进程。 因此，以上所有示例都包含-w 1选项。

   这是因为Gunicorn不支持粘性会话（sticky sessions）

   但这并不可怕，因为你可以使用nginx作为负载平衡器，运行几个gunicorn进程，全部都有一个worker。

3. uWSGI Web服务器
   将uWSGI服务器与gevent结合使用时，Socket.IO服务器可以利用uWSGI原生支持WebSocket的优点。

   有关uWSGI服务器的配置和使用的完整说明超出了本文档的范围。 uWSGI服务器是一个相当复杂的软件包，提供了大量全面的选项。 必须使用WebSocket和SSL支持编译WebSocket传输才能使用。 作为介绍，以下命令为在端口为5000上的示例应用程序app.py启动uWSGI服务器：

   ```
   $ uwsgi --http :5000 --gevent 1000 --http-websockets --master --wsgi-file app.py --callable app
   ```

##### 使用nginx作为WebSocket反向代理

可以使用nginx作为将请求传递给应用程序的前端反向代理。 但是，只有nginx 1.4和更新版本支持WebSocket协议代理。 以下是代理HTTP和WebSocket请求的基本nginx配置：

```
server {
    listen 80;
    server_name _;

    location / {
        include proxy_params;
        proxy_pass http://127.0.0.1:5000;
    }

    location /socket.io {
        include proxy_params;
        proxy_http_version 1.1;
        proxy_buffering off;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_pass http://127.0.0.1:5000/socket.io;
    }
}
```

下一个示例添加了对负载平衡多个Socket.IO服务器的支持：

```
upstream socketio_nodes {
    ip_hash;

    server 127.0.0.1:5000;
    server 127.0.0.1:5001;
    server 127.0.0.1:5002;
    # to scale the app, just add more nodes here!
}

server {
    listen 80;
    server_name _;

    location / {
        include proxy_params;
        proxy_pass http://127.0.0.1:5000;
    }

    location /socket.io {
        include proxy_params;
        proxy_http_version 1.1;
        proxy_buffering off;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_pass http://socketio_nodes/socket.io;
    }
}
```

虽然上述示例可以作为初始配置工作，但请注意，nginx的生产安装需要更完整的配置，涵盖其他部署方面，如提供静态文件资产和SSL支持。



##### 使用多个Workers

从2.0版开始，Flask-SocketIO支持负载均衡器后面的多个workers。部署多个workers使得使用Flask-SocketIO的应用程序能够在多个进程和主机之间传播客户端连接，并以这种方式进行扩展以支持大量的并发客户端。

使用多个Flask-SocketIO Workers有两个要求：

1. 负载均衡器必须配置为将来自给定客户端的所有HTTP请求始终转发给同一个worker。这有时被称为“粘性会话”。对于nginx，使用ip_hash指令来实现这一点。 Gunicorn不能与多个工作人员一起使用，因为其负载平衡器算法不支持粘性会话。

2. 由于每台服务器只拥有客户端连接的一部分，因此服务器使用Redis或RabbitMQ等消息队列来协调诸如广播和房间之类的复杂操作。使用消息队列时，还需要安装其他依赖项：

   - 对于Redis，必须安装软件包redis（pip install redis）。
   - 对于RabbitMQ，必须安装软件包kombu（pip install kombu）。
   - 对于Kombu支持的其他消息队列，请参阅Kombu文档以了解需要哪些依赖关系。
   - 如果使用eventlet或gevent，那么通常需要修补Python标准库来强制消息队列包使用协程友好的函数和类。

   ​

   要启动多个Flask-SocketIO服务器，您必须首先确保您的消息队列服务正在运行。 要启动Socket.IO服务器并将其连接到消息队列，请将message_queue参数添加到SocketIO构造函数中：

   ```
   socketio = SocketIO（app，message_queue ='redis：//'）
   ```

   message_queue参数的值是使用的队列服务的连接URL。 对于在与服务器相同的主机上运行的redis队列，可以使用'redis：//'URL。 同样，对于默认的RabbitMQ队列，可以使用'amqp：//'URL。 Kombu软件包有一个文档部分，描述了所有受支持队列的URL格式。



