## Start with an example

```python
from http.server import HTTPServer, BaseHTTPRequestHandler


SERVER_ADDRESS = (HOST, PORT) = 'localhost', 2017
RESPONSE = b"Hello there"


class MyHandler(BaseHTTPRequestHandler):

	def parse_request(self):
		print(self.raw_requestline)    # b'GET /hello_from_browser HTTP/1.1\r\n'
		return super(MyHandler, self).parse_request()

	def do_GET(self):
		self.send_response(200)
		self.send_header('Content-type', 'text/html')
		self.end_headers()
		self.wfile.write(RESPONSE)
		return


httpd = HTTPServer(SERVER_ADDRESS, MyHandler)
httpd.serve_forever()
```

Now if we go `http://localhost:2017/hello_from_browser` in browser, we will:

* get `Hello there` in browser
* console will print `b'GET /hello_from_browser HTTP/1.1\r\n'`

Now let see what happened.

## `httpd = HTTPServer(SERVER_ADDRESS, MyHandler)` and `httpd.serve_forever()`

Here we create a http server, and by checking the source code, it's just a wrape of **`TCPServer`**, but over writting `server_bind` method:

```python
class HTTPServer(socketserver.TCPServer):

    allow_reuse_address = 1    # Seems to make sense in testing environment

    def server_bind(self):
        """Override server_bind to store the server name."""
        socketserver.TCPServer.server_bind(self)
        host, port = self.server_address[:2]
        self.server_name = socket.getfqdn(host)
        self.server_port = port

```

So, nothing new here and we could just follow the same steps as [socketserver.py link](  https://github.com/alexthemonkey/read_source_code_python_01_socketserver/blob/master/01_non_threading_TCPServer.md)

The main difference between `TCPServer` and `httpServer` is the handler.

Both eventually will call: **`self.RequestHandlerClass()`**

which eventually will call the **`__init__`** of handler class

which eventually calls 3 functions:

* **setup()**
* **handle()**
* **finish()**

```python
class BaseRequestHandler:

    def __init__(self, request, client_address, server):
        self.request = request
        self.client_address = client_address
        self.server = server
        self.setup()
        try:
            self.handle()
        finally:
            self.finish()

    def setup(self):
        pass

    def handle(self):
        pass

    def finish(self):
        pass



class StreamRequestHandler(BaseRequestHandler):
  
    rbufsize = -1
    wbufsize = 0

    timeout = None
    disable_nagle_algorithm = False

    def setup(self):
        self.connection = self.request
        if self.timeout is not None:
            self.connection.settimeout(self.timeout)
        if self.disable_nagle_algorithm:
            self.connection.setsockopt(socket.IPPROTO_TCP,
                                       socket.TCP_NODELAY, True)
        self.rfile = self.connection.makefile('rb', self.rbufsize)
        if self.wbufsize == 0:
            self.wfile = _SocketWriter(self.connection)
        else:
            self.wfile = self.connection.makefile('wb', self.wbufsize)

    def finish(self):
        if not self.wfile.closed:
            try:
                self.wfile.flush()
            except socket.error:
                pass
        self.wfile.close()
        self.rfile.close()
```

## `BaseHTTPRequestHandler`

**`BaseHTTPRequestHandler`** inherists **`StreamRequestHandler`** above, so **`setup()`** and **`finish()`** will be the same because **`BaseHTTPRequestHandler`** does not over-writes those two.

The differences are in **`handle()`**

In **`handle()`**, it just keeps on call **`handle_one_request`** until `close_connection` becomes `True`.

```python
class BaseHTTPRequestHandler(socketserver.StreamRequestHandler):

	...
    
    def handle(self):
        """Handle multiple requests if necessary."""
        self.close_connection = True

        self.handle_one_request()
        while not self.close_connection:
            self.handle_one_request()
```

In **`handle_one_request()`**, it does

* parse request in **`parse_request`** to get **self.command** (`GET`, `POST`, ... etc) and **self.path** (`/hello_from_browser` in our case) etc.
* based on the **self.command**, calling corresponding method. For example, if it's `GET`, then it will call **`slef.do_GET()`**, which should be implemented by child class. In our case, we just write `hello there`. 

```python
class BaseHTTPRequestHandler(socketserver.StreamRequestHandler):

	...
    
    def handle_one_request(self):

        try:
            self.raw_requestline = self.rfile.readline(65537)
            if len(self.raw_requestline) > 65536:
                self.requestline = ''
                self.request_version = ''
                self.command = ''
                self.send_error(HTTPStatus.REQUEST_URI_TOO_LONG)
                return
            if not self.raw_requestline:
                self.close_connection = True
                return
            if not self.parse_request():
                return
            mname = 'do_' + self.command  # this where we differs verbs.
            if not hasattr(self, mname):
                self.send_error(
                    HTTPStatus.NOT_IMPLEMENTED,
                    "Unsupported method (%r)" % self.command)
                return
            method = getattr(self, mname)
            method()
            self.wfile.flush() #actually send the response if not already done.
        except socket.timeout as e:
            self.log_error("Request timed out: %r", e)
            self.close_connection = True
            return
```

But where do we send the data as response? See the code in **`do_GET()`** in our **`MyHandler`** class, we used:

* self.send_response(200)
* self.send_header('Content-type', 'text/html')
* self.end_headers()

All those methods are help sending data as response.

```python
class BaseHTTPRequestHandler(socketserver.StreamRequestHandler):

	...
    
    def send_response(self, code, message=None):
        self.log_request(code)
        self.send_response_only(code, message)
        self.send_header('Server', self.version_string())
        self.send_header('Date', self.date_time_string())
        
        
    def send_response_only(self, code, message=None):
        """Send the response header only."""
        if self.request_version != 'HTTP/0.9':
            if message is None:
                if code in self.responses:
                    message = self.responses[code][0]
                else:
                    message = ''
            if not hasattr(self, '_headers_buffer'):
                self._headers_buffer = []
            self._headers_buffer.append(("%s %d %s\r\n" %
                    (self.protocol_version, code, message)).encode(
                        'latin-1', 'strict'))
    
    
    def send_header(self, keyword, value):
        """Send a MIME header to the headers buffer."""
        if self.request_version != 'HTTP/0.9':
            if not hasattr(self, '_headers_buffer'):
                self._headers_buffer = []
            self._headers_buffer.append(
                ("%s: %s\r\n" % (keyword, value)).encode('latin-1', 'strict'))

        if keyword.lower() == 'connection':
            if value.lower() == 'close':
                self.close_connection = True
            elif value.lower() == 'keep-alive':
                self.close_connection = False
```


![alt text](overview_of_function_calls.png)