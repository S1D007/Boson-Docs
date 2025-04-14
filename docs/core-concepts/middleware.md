---
sidebar_position: 4
title: Middleware
---

# Middleware in Boson

Middleware components are a powerful feature of the Boson framework that allow you to filter HTTP requests and responses. This guide explains how middleware works and how to create and use middleware effectively in your applications.

## What is Middleware?

Middleware acts as a layer between the client request and your application's route handlers. It provides a mechanism to inspect, filter, or modify incoming requests and outgoing responses. Middleware can perform tasks such as:

- Authentication and authorization
- Request logging
- Input validation
- CORS handling
- Response compression
- Session management
- Rate limiting

## Middleware Execution Flow

In Boson, middleware components are executed in a pipeline:

```
          │ Request
          ▼
┌─────────────────────┐
│  Middleware 1       │
│  (pre-processing)   │
└──────────┬──────────┘
          │
          ▼
┌─────────────────────┐
│  Middleware 2       │
│  (pre-processing)   │
└──────────┬──────────┘
          │
          ▼
┌─────────────────────┐
│  Route Handler      │
└──────────┬──────────┘
          │
          ▼
┌─────────────────────┐
│  Middleware 2       │
│  (post-processing)  │
└──────────┬──────────┘
          │
          ▼
┌─────────────────────┐
│  Middleware 1       │
│  (post-processing)  │
└──────────┬──────────┘
          │
          ▼
         Response
```

The key aspects of this flow are:

1. Middleware executes in the order it's registered (for pre-processing)
2. After the route handler generates a response, middleware executes in reverse order (for post-processing)
3. Any middleware can terminate the flow by returning a response early

## Creating Middleware

### Basic Middleware Class

To create middleware in Boson, extend the `boson::Middleware` base class:

```cpp
#include <boson/middleware.hpp>
#include <chrono>
#include <iostream>

class TimingMiddleware : public boson::Middleware {
public:
    boson::Response process(const boson::Request& request, MiddlewareNext next) override {
        // Pre-processing: Record start time
        auto start = std::chrono::high_resolution_clock::now();
        
        // Call the next middleware or the route handler
        boson::Response response = next(request);
        
        // Post-processing: Calculate and log the elapsed time
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
        
        std::cout << "Request to " << request.path() 
                  << " took " << duration.count() << "ms" << std::endl;
        
        // Return the response to continue the chain
        return response;
    }
};
```

### Middleware with Constructor Parameters

You can create middleware that accepts configuration parameters:

```cpp
class RateLimitMiddleware : public boson::Middleware {
public:
    // Constructor with configuration parameters
    RateLimitMiddleware(int limit, int timeWindowSeconds)
        : limit_(limit), timeWindowSeconds_(timeWindowSeconds) {
        // Initialize rate limiting storage
    }
    
    boson::Response process(const boson::Request& request, MiddlewareNext next) override {
        // Get client IP
        const std::string& clientIp = request.clientIp();
        
        // Check if rate limit is exceeded
        if (isRateLimitExceeded(clientIp)) {
            return boson::Response::tooManyRequests()
                .json({{"error", "Rate limit exceeded"}});
        }
        
        // Increment request count for this client
        incrementRequestCount(clientIp);
        
        // Continue to next middleware
        return next(request);
    }

private:
    int limit_;
    int timeWindowSeconds_;
    
    bool isRateLimitExceeded(const std::string& clientIp);
    void incrementRequestCount(const std::string& clientIp);
};
```

### Functional Middleware

For simpler cases, you can use functional middleware:

```cpp
// Define middleware as a function
auto corsMiddleware = [](const boson::Request& request, boson::MiddlewareNext next) -> boson::Response {
    // Process the request through the rest of the chain
    auto response = next(request);
    
    // Add CORS headers to the response
    response.header("Access-Control-Allow-Origin", "*");
    response.header("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE");
    response.header("Access-Control-Allow-Headers", "Content-Type, Authorization");
    
    return response;
};

// Register functional middleware
app.useMiddleware(corsMiddleware);
```

## Registering Middleware

### Global Middleware

Global middleware applies to all routes in your application:

```cpp
#include <boson/boson.hpp>
#include "middleware/auth_middleware.hpp"
#include "middleware/logging_middleware.hpp"

int main() {
    boson::Application app;
    
    // Register global middleware (order matters)
    app.useMiddleware<LoggingMiddleware>();
    app.useMiddleware<AuthMiddleware>();
    
    // Middleware with constructor parameters
    app.useMiddleware<RateLimitMiddleware>(100, 60); // 100 requests per 60 seconds
    
    // Routes and controllers...
    
    boson::Server server(app);
    server.listen(8080);
    return 0;
}
```

### Route-Specific Middleware

You can apply middleware to specific routes:

```cpp
// Apply middleware to a single route
app.get("/admin/dashboard", 
    std::make_shared<AuthMiddleware>(), 
    [](const boson::Request& req) {
        return boson::Response::ok("Admin Dashboard");
    }
);

// Apply multiple middleware to a route
app.get("/reports", 
    std::vector<std::shared_ptr<boson::Middleware>>{
        std::make_shared<AuthMiddleware>(),
        std::make_shared<CacheMiddleware>(3600) // Cache for 1 hour
    },
    reportController
);
```

### Group Middleware

You can apply middleware to a group of routes:

```cpp
// Create a route group with middleware
auto adminGroup = app.group("/admin", std::make_shared<AuthMiddleware>());

// Add routes to the group (all will use the AuthMiddleware)
adminGroup->get("/dashboard", dashboardHandler);
adminGroup->get("/users", usersHandler);
adminGroup->get("/settings", settingsHandler);
```

### Controller Middleware

You can apply middleware to all routes in a controller:

```cpp
class AdminController : public boson::Controller {
public:
    void registerMiddleware() override {
        // Apply middleware to all routes in this controller
        useMiddleware<AuthMiddleware>();
        useMiddleware<LoggingMiddleware>();
    }
    
    void registerRoutes() override {
        GET("/dashboard", &AdminController::dashboard);
        GET("/users", &AdminController::users);
        // ...
    }
    
    // Handler methods...
};
```

## Built-in Middleware

Boson comes with several built-in middleware components:

### CORS Middleware

```cpp
// Register CORS middleware with default configuration (allow all origins)
app.useMiddleware<boson::middleware::Cors>();

// With custom configuration
boson::middleware::CorsConfig corsConfig;
corsConfig.allowedOrigins = {"https://example.com", "https://api.example.com"};
corsConfig.allowedMethods = {"GET", "POST", "PUT"};
corsConfig.allowedHeaders = {"Content-Type", "Authorization"};
corsConfig.exposedHeaders = {"X-Custom-Header"};
corsConfig.maxAge = 3600;
corsConfig.allowCredentials = true;

app.useMiddleware<boson::middleware::Cors>(corsConfig);
```

### Authentication Middleware

```cpp
// JWT authentication middleware
boson::middleware::JwtConfig jwtConfig;
jwtConfig.secret = "your-secret-key";
jwtConfig.algorithm = "HS256";
jwtConfig.issuer = "boson-api";
jwtConfig.audience = "web-client";

app.useMiddleware<boson::middleware::JwtAuth>(jwtConfig);
```

### Rate Limiting Middleware

```cpp
// Rate limit: 100 requests per minute per IP
app.useMiddleware<boson::middleware::RateLimit>(100, 60);

// With custom configuration
boson::middleware::RateLimitConfig rateLimitConfig;
rateLimitConfig.limit = 100;
rateLimitConfig.window = 60; // seconds
rateLimitConfig.keyPrefix = "rate_limit:";
rateLimitConfig.keyGenerator = [](const boson::Request& req) {
    // Generate keys based on user ID if logged in, otherwise use IP
    if (req.user()) {
        return "user:" + req.user()->id();
    }
    return "ip:" + req.clientIp();
};

app.useMiddleware<boson::middleware::RateLimit>(rateLimitConfig);
```

### Compression Middleware

```cpp
// Enable response compression
app.useMiddleware<boson::middleware::Compression>();

// With custom configuration
boson::middleware::CompressionConfig compConfig;
compConfig.level = boson::middleware::CompressionLevel::Medium;
compConfig.minSize = 1024;  // Only compress responses larger than 1KB
compConfig.types = {"text/html", "text/css", "application/json"};

app.useMiddleware<boson::middleware::Compression>(compConfig);
```

### CSRF Protection Middleware

```cpp
// Add CSRF protection
app.useMiddleware<boson::middleware::CsrfProtection>();
```

### Body Parsing Middleware

```cpp
// Parse JSON request bodies
app.useMiddleware<boson::middleware::JsonBodyParser>();

// Parse form data (application/x-www-form-urlencoded)
app.useMiddleware<boson::middleware::FormBodyParser>();

// Parse multipart form data (for file uploads)
app.useMiddleware<boson::middleware::MultipartFormParser>();
```

### Session Middleware

```cpp
// Configure session middleware
boson::middleware::SessionConfig sessionConfig;
sessionConfig.name = "boson_session";
sessionConfig.lifetime = 3600;  // 1 hour in seconds
sessionConfig.secure = true;
sessionConfig.httpOnly = true;
sessionConfig.sameSite = "lax";

// Use default cookie-based sessions
app.useMiddleware<boson::middleware::Session>(sessionConfig);

// Or use Redis for session storage
app.useMiddleware<boson::middleware::RedisSession>(sessionConfig, redisConnection);
```

## Advanced Middleware Techniques

### Terminating Middleware

Middleware can terminate the request flow by returning a response without calling `next()`:

```cpp
class AuthMiddleware : public boson::Middleware {
public:
    boson::Response process(const boson::Request& request, MiddlewareNext next) override {
        // Check for authentication token
        if (!request.hasHeader("Authorization")) {
            // Terminate the middleware chain and return 401 Unauthorized
            return boson::Response::unauthorized()
                .json({{"error", "Authentication required"}});
        }
        
        // Extract token
        std::string token = extractToken(request.header("Authorization"));
        
        // Validate token
        if (!isValidToken(token)) {
            // Terminate with an error response
            return boson::Response::forbidden()
                .json({{"error", "Invalid token"}});
        }
        
        // Token is valid, continue the middleware chain
        return next(request);
    }

private:
    std::string extractToken(const std::string& authHeader);
    bool isValidToken(const std::string& token);
};
```

### Request Modification

Middleware can modify the request before it reaches the route handler:

```cpp
class UserResolverMiddleware : public boson::Middleware {
public:
    UserResolverMiddleware(std::shared_ptr<UserService> userService)
        : userService_(std::move(userService)) {}
        
    boson::Response process(const boson::Request& request, MiddlewareNext next) override {
        // Check if request has an auth token
        if (request.hasHeader("Authorization")) {
            std::string token = extractToken(request.header("Authorization"));
            
            // Resolve the user from the token
            auto user = userService_->findUserByToken(token);
            
            if (user) {
                // Create a new request with the user attached
                boson::Request newRequest = request.withAttribute("user", user);
                
                // Continue chain with the modified request
                return next(newRequest);
            }
        }
        
        // Continue the chain with the original request
        return next(request);
    }

private:
    std::shared_ptr<UserService> userService_;
    std::string extractToken(const std::string& authHeader);
};
```

### Response Modification

Middleware can modify the response after it's generated by the route handler:

```cpp
class SecurityHeadersMiddleware : public boson::Middleware {
public:
    boson::Response process(const boson::Request& request, MiddlewareNext next) override {
        // Process the request normally
        boson::Response response = next(request);
        
        // Add security headers to the response
        response.header("Content-Security-Policy", 
            "default-src 'self'; script-src 'self' https://trusted-cdn.com");
        response.header("X-XSS-Protection", "1; mode=block");
        response.header("X-Frame-Options", "DENY");
        response.header("X-Content-Type-Options", "nosniff");
        response.header("Referrer-Policy", "strict-origin-when-cross-origin");
        response.header("Permissions-Policy", 
            "camera=(), microphone=(), geolocation=()");
        
        return response;
    }
};
```

### Conditional Middleware Execution

You can create middleware that conditionally executes based on the request:

```cpp
class ConditionalMiddleware : public boson::Middleware {
public:
    boson::Response process(const boson::Request& request, MiddlewareNext next) override {
        // Check if this middleware should run based on the request
        if (shouldProcess(request)) {
            // Do middleware processing
            // ...
        }
        
        // Always continue the chain
        return next(request);
    }

private:
    bool shouldProcess(const boson::Request& request) {
        // Example: Only process API requests
        return request.path().starts_with("/api");
    }
};
```

### Middleware with Dependencies

You can create middleware that depends on application services:

```cpp
class CacheMiddleware : public boson::Middleware {
public:
    CacheMiddleware(std::shared_ptr<CacheService> cacheService)
        : cacheService_(std::move(cacheService)) {}
        
    boson::Response process(const boson::Request& request, MiddlewareNext next) override {
        // Only cache GET requests
        if (request.method() != "GET") {
            return next(request);
        }
        
        // Generate cache key based on request
        std::string cacheKey = generateCacheKey(request);
        
        // Check if response is cached
        if (auto cachedResponse = cacheService_->get(cacheKey)) {
            // Return cached response
            return *cachedResponse;
        }
        
        // Generate fresh response
        boson::Response response = next(request);
        
        // Cache response if it was successful
        if (response.statusCode() == 200) {
            cacheService_->set(cacheKey, response, 300); // Cache for 5 minutes
        }
        
        return response;
    }

private:
    std::shared_ptr<CacheService> cacheService_;
    
    std::string generateCacheKey(const boson::Request& request) {
        // Combine method, path, and query string
        return request.method() + ":" + request.fullPath();
    }
};
```

## Testing Middleware

Boson makes it easy to test middleware in isolation:

```cpp
#include <boson/testing/middleware_tester.hpp>
#include <gtest/gtest.h>
#include "middleware/auth_middleware.hpp"

// Test fixture
class AuthMiddlewareTest : public ::testing::Test {
protected:
    AuthMiddlewareTest() {
        // Create middleware instance
        middleware_ = std::make_unique<AuthMiddleware>();
        
        // Create middleware tester
        tester_ = std::make_unique<boson::MiddlewareTester>(middleware_.get());
    }
    
    std::unique_ptr<AuthMiddleware> middleware_;
    std::unique_ptr<boson::MiddlewareTester> tester_;
};

// Test methods
TEST_F(AuthMiddlewareTest, RejectsRequestWithoutToken) {
    // Create a request without an Authorization header
    auto request = boson::Request::fromUrl("GET", "/protected");
    
    // Process the request through middleware
    auto response = tester_->process(request);
    
    // Assert the response indicates unauthorized
    ASSERT_EQ(401, response.statusCode());
}

TEST_F(AuthMiddlewareTest, AcceptsRequestWithValidToken) {
    // Create a request with a valid Authorization header
    auto request = boson::Request::fromUrl("GET", "/protected")
        .header("Authorization", "Bearer valid-token");
    
    // Process the request through middleware
    auto response = tester_->process(request);
    
    // Assert the middleware allowed the request to continue
    ASSERT_EQ(200, response.statusCode());
}
```

## Best Practices

1. **Single Responsibility**: Each middleware should focus on a single task
2. **Order Matters**: Register middleware in the correct order based on dependencies
3. **Performance**: Keep middleware lightweight to minimize impact on request processing time
4. **Error Handling**: Implement proper error handling in all middleware
5. **Testing**: Write thorough tests for middleware components
6. **Reusability**: Design middleware to be reusable across different applications
7. **Configuration**: Make middleware configurable rather than hardcoding values
8. **Documentation**: Document the purpose and behavior of custom middleware

## Next Steps

Now that you understand middleware in Boson, explore these related topics:

1. Learn about [Request and Response](request-response.md) objects in detail
2. Understand [Error Handling](error-handling.md) for robust error management
3. Explore [Security](../security/authentication.md) features of the framework