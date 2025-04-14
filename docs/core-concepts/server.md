---
sidebar_position: 1
title: Server
---

# Boson Server

The server component is at the heart of any Boson application. It handles HTTP connections, manages the request lifecycle, and coordinates between different parts of your application. This guide explains how the Boson server works and how to configure it for optimal performance.

## Server Architecture

The Boson server uses an event-driven, asynchronous architecture for maximum performance and scalability:

```
                   ┌─────────┐
                   │ Client  │
                   └────┬────┘
                        │
                        ▼
┌──────────────────────────────────────────┐
│              Socket Listener             │
└──────────────┬───────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────┐
│           Connection Pool                │
└──────────────┬───────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────┐
│            HTTP Parser                   │
└──────────────┬───────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────┐
│          Request Object                  │
└──────────────┬───────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────┐
│        Middleware Pipeline               │
└──────────────┬───────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────┐
│           Router                         │
└──────────────┬───────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────┐
│         Controller/Handler               │
└──────────────┬───────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────┐
│           Response Object                │
└──────────────┬───────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────┐
│              Client                      │
└──────────────────────────────────────────┘
```

### Key Components

1. **Socket Listener**: Accepts incoming TCP connections
2. **Connection Pool**: Manages active connections for reuse
3. **HTTP Parser**: Parses raw HTTP requests into structured data
4. **Request Handler**: Routes requests to the appropriate controller
5. **Response Writer**: Serializes responses back to the client
6. **Worker Pool**: Distributes processing across multiple threads

## Basic Server Usage

Creating and starting a Boson server is straightforward:

```cpp
#include <boson/boson.hpp>
#include <boson/server.hpp>

int main() {
    // Create the application
    boson::Application app;
    
    // Register routes and controllers
    app.route("/hello", [](const boson::Request& req) {
        return boson::Response::ok("Hello, World!");
    });
    
    // Create server with application
    boson::Server server(app);
    
    // Start listening on port 8080
    server.listen(8080);
    
    return 0;
}
```

## Server Configuration

### Basic Configuration Options

The Boson server provides numerous configuration options:

```cpp
boson::ServerConfig config;
config.host = "127.0.0.1";      // Interface to bind to
config.port = 8080;             // Port to listen on
config.workers = 4;             // Number of worker threads
config.maxConnections = 1000;   // Maximum simultaneous connections
config.connectionTimeout = 30;  // Connection timeout in seconds
config.keepAliveTimeout = 5;    // Keep-alive timeout in seconds

boson::Server server(app, config);
```

### Configuration from JSON

You can also load configuration from a JSON file:

```cpp
boson::Application app;
app.loadConfig("config/server.json");

boson::Server server(app);
```

Example `server.json`:

```json
{
  "server": {
    "host": "0.0.0.0",
    "port": 8080,
    "workers": 8,
    "max_connections": 10000,
    "connection_timeout": 30,
    "keep_alive_timeout": 5,
    "backlog": 128
  }
}
```

## TLS/SSL Support

To enable HTTPS, configure SSL certificates:

```cpp
boson::ServerConfig config;
config.enableTls = true;
config.certFile = "certs/server.crt";
config.keyFile = "certs/server.key";

boson::Server server(app, config);
```

## HTTP/2 Support

Boson supports HTTP/2 for improved performance:

```cpp
boson::ServerConfig config;
config.enableHttp2 = true;  // Enable HTTP/2 support

boson::Server server(app, config);
```

## Performance Tuning

### Worker Threads

The number of worker threads directly impacts performance:

```cpp
boson::ServerConfig config;

// Auto-detect optimal number based on CPU cores
config.workers = boson::Server::getOptimalWorkerCount();

// Or set manually (typically 1-2x number of CPU cores)
config.workers = 8;
```

### Connection Pooling

Configure connection pooling for optimal resource usage:

```cpp
boson::ServerConfig config;
config.maxConnections = 10000;     // Maximum concurrent connections
config.connectionPoolSize = 1000;  // Size of the connection pool
config.connectionTimeout = 30;     // Timeout for idle connections (seconds)
```

### Buffer Sizes

Adjust buffer sizes for different workloads:

```cpp
boson::ServerConfig config;
config.readBufferSize = 16384;    // 16KB read buffer
config.writeBufferSize = 16384;   // 16KB write buffer
```

## Graceful Shutdown

Boson supports graceful shutdown to handle in-flight requests:

```cpp
#include <signal.h>

boson::Server* serverPtr = nullptr;

void signalHandler(int signal) {
    if (serverPtr) {
        // Allow 5 seconds for graceful shutdown
        serverPtr->shutdown(5);
    }
    exit(signal);
}

int main() {
    // Create application and server
    boson::Application app;
    boson::Server server(app);
    
    // Store server pointer for signal handler
    serverPtr = &server;
    
    // Set up signal handlers
    signal(SIGINT, signalHandler);
    signal(SIGTERM, signalHandler);
    
    // Start server
    server.listen(8080);
    
    return 0;
}
```

## Server Events

Boson provides hooks for server lifecycle events:

```cpp
server.onStart([](const boson::Server& s) {
    std::cout << "Server started on " << s.host() << ":" << s.port() << std::endl;
});

server.onShutdown([](const boson::Server& s) {
    std::cout << "Server shutting down..." << std::endl;
});

server.onError([](const std::exception& e) {
    std::cerr << "Server error: " << e.what() << std::endl;
});
```

## Static File Serving

Boson can efficiently serve static files:

```cpp
// Serve a specific file
app.route("/favicon.ico", [](const boson::Request& req) {
    return boson::Response::file("public/favicon.ico", "image/x-icon");
});

// Serve a directory of static files
app.serveStatic("/assets", "public/assets");

// Serve a directory with custom options
boson::StaticFileOptions options;
options.cacheControl = "max-age=86400";
options.etag = true;
options.gzip = true;
app.serveStatic("/static", "public", options);
```

## Multiple Servers

You can run multiple servers for different purposes:

```cpp
// Main application server (HTTP)
boson::Server appServer(app);
appServer.listen(8080);

// API server (HTTPS)
boson::ServerConfig apiConfig;
apiConfig.enableTls = true;
apiConfig.certFile = "certs/api.crt";
apiConfig.keyFile = "certs/api.key";

boson::Server apiServer(app, apiConfig);
apiServer.listen(8443);
```

## Advanced Server Features

### Custom Socket Options

For fine-tuned control, set custom socket options:

```cpp
boson::ServerConfig config;

// Set TCP_NODELAY (disable Nagle's algorithm)
config.socketOptions.push_back({SOL_TCP, TCP_NODELAY, 1});

// Increase buffer sizes
config.socketOptions.push_back({SOL_SOCKET, SO_RCVBUF, 262144});
config.socketOptions.push_back({SOL_SOCKET, SO_SNDBUF, 262144});
```

### UNIX Domain Sockets

For local communication, use UNIX domain sockets:

```cpp
boson::ServerConfig config;
config.unixSocketPath = "/tmp/boson.sock";
config.useUnixSocket = true;

boson::Server server(app, config);
server.listen();  // No port needed for UNIX socket
```

### Custom Protocol Handlers

Register handlers for custom protocols:

```cpp
server.registerProtocolHandler("ws", [](boson::Connection& conn) {
    // Handle WebSocket upgrade
    // ...
});
```

## Monitoring and Metrics

Boson provides built-in performance metrics:

```cpp
// Get current server stats
boson::ServerStats stats = server.stats();
std::cout << "Active connections: " << stats.activeConnections << std::endl;
std::cout << "Requests per second: " << stats.requestsPerSecond << std::endl;
std::cout << "Average response time: " << stats.avgResponseTime << "ms" << std::endl;

// Register a metrics callback
server.onMetrics([](const boson::ServerStats& stats) {
    // Log or send metrics to monitoring system
    // ...
});
```

## Best Practices

1. **Worker Threads**: Use 1-2 threads per CPU core for optimal performance
2. **Connection Timeouts**: Set appropriate timeouts to prevent resource exhaustion
3. **Buffer Sizes**: Tune buffer sizes based on your application's needs
4. **TLS Configuration**: Use modern TLS protocols and cipher suites
5. **Graceful Shutdown**: Always implement graceful shutdown to prevent request interruption
6. **Error Handling**: Set up proper error handlers for server exceptions
7. **Monitoring**: Use metrics to identify performance bottlenecks

## Next Steps

Continue exploring core Boson concepts:

1. Learn about the [Routing System](routing.md)
2. Understand [Controllers](controllers.md) for organizing route handlers
3. Explore [Middleware](middleware.md) for extending the request pipeline