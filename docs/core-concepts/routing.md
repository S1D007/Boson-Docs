---
sidebar_position: 2
title: Routing
---

# Routing in Boson

The routing system is a fundamental component of Boson that maps HTTP requests to the appropriate handlers based on their URLs and methods. This guide explains how routing works in Boson and demonstrates various routing techniques.

## Routing Basics

In Boson, routes define which code executes when a specific URL is requested. A route combines:
- A URL pattern
- An HTTP method (GET, POST, etc.)
- A handler function or controller method

## Defining Routes

### Simple Route Definition

The most basic way to define a route is with a lambda handler:

```cpp
#include <boson/boson.hpp>

int main() {
    boson::Application app;
    
    // Simple route with lambda handler
    app.route("GET", "/hello", [](const boson::Request& request) -> boson::Response {
        return boson::Response::ok("Hello, World!");
    });
    
    // Method-specific shortcuts are also available
    app.get("/hello", [](const boson::Request& request) {
        return boson::Response::ok("Hello from GET!");
    });
    
    app.post("/hello", [](const boson::Request& request) {
        return boson::Response::ok("Hello from POST!");
    });
    
    boson::Server server(app);
    server.listen(8080);
    return 0;
}
```

### Route Parameters

Routes can include parameters that extract values from the URL:

```cpp
// Route with a named parameter
app.get("/users/{id}", [](const boson::Request& request) {
    // Extract the parameter value
    std::string userId = request.param("id");
    return boson::Response::ok("User ID: " + userId);
});

// Multiple parameters
app.get("/users/{userId}/posts/{postId}", [](const boson::Request& request) {
    std::string userId = request.param("userId");
    std::string postId = request.param("postId");
    return boson::Response::ok("User " + userId + ", Post " + postId);
});
```

### Type-Safe Parameters

Boson supports automatic type conversion for route parameters:

```cpp
app.get("/users/{id}", [](const boson::Request& request) {
    // Extract and convert to integer
    int userId = request.param<int>("id");
    
    // Will throw a boson::InvalidParameterException if conversion fails
    return boson::Response::ok("User ID: " + std::to_string(userId));
});
```

### Optional Parameters

You can define optional route parameters:

```cpp
// Optional segment parameter
app.get("/products[/{category}]", [](const boson::Request& request) {
    if (request.hasParam("category")) {
        std::string category = request.param("category");
        return boson::Response::ok("Products in category: " + category);
    }
    return boson::Response::ok("All products");
});
```

### Route Constraints

You can add constraints to route parameters to match specific patterns:

```cpp
// Constrain 'id' to digits only
app.get("/users/{id:int}", [](const boson::Request& request) {
    int userId = request.param<int>("id");
    return boson::Response::ok("User ID: " + std::to_string(userId));
});

// Other built-in constraints
app.get("/posts/{slug:alpha}", handlerFunction);        // Alphabetic characters only
app.get("/files/{filename:alnum}", handlerFunction);    // Alphanumeric characters
app.get("/users/{guid:uuid}", handlerFunction);         // UUID format
app.get("/articles/{date:datetime}", handlerFunction);  // Date/time format
```

### Custom Constraints

You can define custom parameter constraints using regular expressions:

```cpp
// Define a custom constraint using regex
app.defineConstraint("zipcode", "^\\d{5}(-\\d{4})?$");

// Use custom constraint in routes
app.get("/locations/{zip:zipcode}", [](const boson::Request& request) {
    std::string zipCode = request.param("zip");
    return boson::Response::ok("Location: " + zipCode);
});
```

## Route Groups

Route groups help organize related routes and apply shared attributes:

```cpp
// Create a route group
auto apiGroup = app.group("/api");

// Add routes to the group
apiGroup->get("/users", getUsersHandler);
apiGroup->post("/users", createUserHandler);
apiGroup->get("/users/{id}", getUserByIdHandler);
apiGroup->put("/users/{id}", updateUserHandler);
apiGroup->del("/users/{id}", deleteUserHandler);  // 'delete' is a C++ keyword, so we use 'del'

// Nested groups
auto v1Group = apiGroup->group("/v1");
v1Group->get("/products", getProductsV1Handler);

auto v2Group = apiGroup->group("/v2");
v2Group->get("/products", getProductsV2Handler);
```

## Route Middleware

You can apply middleware to specific routes or groups:

```cpp
// Apply middleware to a single route
app.get("/admin/dashboard", 
    boson::middleware::auth(), 
    [](const boson::Request& request) {
        return boson::Response::ok("Admin Dashboard");
    }
);

// Apply middleware to a group
auto adminGroup = app.group("/admin", boson::middleware::auth());
adminGroup->get("/users", adminGetUsersHandler);
adminGroup->get("/settings", adminGetSettingsHandler);
```

## Named Routes

You can name routes for easier URL generation:

```cpp
// Define a named route
app.get("/users/{id}", userProfileHandler)
    .name("user.profile");

// Generate URLs using named routes in your handlers
boson::Response showUserList(const boson::Request& request) {
    // Generate URL for the "user.profile" route with parameter
    std::string url = request.route("user.profile", {{"id", "123"}});
    
    // url will be "/users/123"
    return boson::Response::ok("User profile URL: " + url);
}
```

## Domain and Subdomain Routing

You can route based on domains or subdomains:

```cpp
// Domain-specific route
app.domain("api.example.com")->get("/users", apiGetUsersHandler);

// Subdomain route with parameter
app.domain("{tenant}.example.com")->get("/dashboard", [](const boson::Request& request) {
    std::string tenant = request.param("tenant");
    return boson::Response::ok(tenant + " Dashboard");
});
```

## Method-based Routing

Boson supports all standard HTTP methods:

```cpp
app.get("/resource", getResourceHandler);
app.post("/resource", createResourceHandler);
app.put("/resource", updateResourceHandler);
app.patch("/resource", patchResourceHandler);
app.del("/resource", deleteResourceHandler);
app.options("/resource", optionsResourceHandler);
app.head("/resource", headResourceHandler);
```

### Method Spoofing

For clients that don't support all HTTP methods (like HTML forms), you can use method spoofing:

```cpp
// Enable method spoofing in your application
app.enableMethodSpoofing();
```

With method spoofing enabled, clients can send a POST request with a `_method` field to simulate other HTTP methods:

```html
<form method="POST" action="/users/123">
    <input type="hidden" name="_method" value="DELETE">
    <button type="submit">Delete User</button>
</form>
```

## Route Fallbacks

You can define fallback routes for handling 404 errors:

```cpp
// Define a fallback route for unmatched URLs
app.fallback([](const boson::Request& request) {
    return boson::Response::notFound()
        .body("<h1>Page not found</h1><p>The page you requested does not exist.</p>")
        .contentType("text/html");
});
```

## Advanced Routing Techniques

### Regex Routes

For complex matching patterns, you can use regular expression routes:

```cpp
// Route with regex pattern
app.get(boson::Route::regex("/files/.*\\.pdf$"), [](const boson::Request& request) {
    return boson::Response::ok("PDF file requested");
});
```

### Wildcard Routes

Wildcards can capture entire path segments:

```cpp
// Wildcard route to capture all paths under /docs
app.get("/docs/*path", [](const boson::Request& request) {
    std::string path = request.param("path");
    return boson::Response::ok("Documentation path: " + path);
});
```

### Route Priority

Routes have a default priority based on their specificity, but you can override it:

```cpp
// Increase priority (processed earlier)
app.get("/users/{id}", userHandler).priority(10);

// Decrease priority (processed later)
app.get("/users/admins", adminHandler).priority(-5);
```

## Route Registration in Controllers

In controller classes, routes are typically registered in the `registerRoutes` method:

```cpp
class UserController : public boson::Controller {
public:
    void registerRoutes() override {
        // Register routes with their handlers
        GET("/users", &UserController::index);
        GET("/users/{id}", &UserController::show);
        POST("/users", &UserController::create);
        PUT("/users/{id}", &UserController::update);
        DELETE("/users/{id}", &UserController::destroy);
    }

    // Handler methods
    boson::Response index(const boson::Request& request);
    boson::Response show(const boson::Request& request);
    boson::Response create(const boson::Request& request);
    boson::Response update(const boson::Request& request);
    boson::Response destroy(const boson::Request& request);
};

// Register the controller with the application
app.registerController<UserController>();
```

## Route Caching

For applications with many routes, route caching can improve performance:

```cpp
// Enable route caching
app.enableRouteCaching();

// Manually cache routes (typically in production)
app.cacheRoutes("cache/routes.cache");

// Load cached routes
app.loadRoutesFromCache("cache/routes.cache");
```

## Conditional Routes

You can define routes with conditions:

```cpp
// Route that only matches during specific hours
app.get("/store", [](const boson::Request& req) {
    return boson::Response::ok("Store is open");
})
.where([](const boson::Request& req) {
    auto now = std::chrono::system_clock::now();
    auto time = std::chrono::system_clock::to_time_t(now);
    auto tm = std::localtime(&time);
    
    // Only match during business hours (9am-5pm)
    return tm->tm_hour >= 9 && tm->tm_hour < 17;
});
```

## Testing Routes

Boson provides utilities for testing routes:

```cpp
#include <boson/testing/http_tester.hpp>

// In your test function
boson::HttpTester tester(app);

// Test a GET request
auto response = tester.get("/users/123");
assert(response.statusCode() == 200);
assert(response.json()["id"] == 123);

// Test a POST request with JSON body
auto createResponse = tester.post("/users")
    .json({{"name", "John"}, {"email", "john@example.com"}})
    .send();
assert(createResponse.statusCode() == 201);
```

## Best Practices

1. **Organize related routes** using route groups and controllers
2. **Use descriptive route names** that reflect the resource and action
3. **Follow REST conventions** for resource-based routes
4. **Apply middleware at the appropriate level** (global, group, or route)
5. **Use type-safe parameters** to avoid runtime type conversion errors
6. **Implement proper error handling** for parameter validation failures
7. **Cache routes in production** for improved performance
8. **Test all routes** to ensure they behave as expected

## Next Steps

Now that you understand Boson's routing system, you can explore related topics:

1. Learn about [Controllers](controllers.md) for organizing route handlers
2. Understand [Middleware](middleware.md) for processing requests and responses
3. Explore [Request and Response](request-response.md) objects in detail