---
sidebar_position: 5
title: Request & Response
---

# Request and Response in Boson

The Request and Response classes are fundamental building blocks in Boson. They provide a clean, object-oriented interface for working with HTTP requests and responses. This guide explains their features and how to use them effectively.

## The Request Object

The `boson::Request` class represents an incoming HTTP request from a client. It provides methods to access all aspects of the request, including URL parameters, query strings, headers, cookies, and request body.

### Creating Request Objects

Typically, you'll work with Request objects that Boson creates automatically from incoming HTTP requests. However, for testing or special cases, you can create custom Request objects:

```cpp
// Create a request from a URL (for testing)
auto request = boson::Request::fromUrl("GET", "/users/123");

// Create a request with headers
auto requestWithHeaders = boson::Request::fromUrl("GET", "/api/data")
    .header("Authorization", "Bearer token123")
    .header("Accept", "application/json");

// Create a request with JSON body
auto postRequest = boson::Request::fromUrl("POST", "/api/users")
    .header("Content-Type", "application/json")
    .body("{\"name\":\"John\",\"email\":\"john@example.com\"}");
```

### Request Properties

#### HTTP Method

```cpp
// Get the HTTP method
std::string method = request.method();  // "GET", "POST", etc.

// Check the method
if (request.isMethod("POST")) {
    // Handle POST request
}
```

#### URL and Path

```cpp
// Get the full URL
std::string url = request.url();  // "https://example.com/users/123?sort=name"

// Get just the path
std::string path = request.path();  // "/users/123"

// Get the path without query string
std::string fullPath = request.fullPath();  // "/users/123?sort=name"

// Check if the path matches a pattern
if (request.pathIs("/users/*")) {
    // Path starts with "/users/"
}
```

#### Route Parameters

```cpp
// Get a route parameter by name
std::string userId = request.param("id");  // "123" from "/users/{id}"

// Get a parameter with type conversion
int userIdInt = request.param<int>("id");  // 123 as integer

// Check if a parameter exists
if (request.hasParam("id")) {
    // Parameter exists
}

// Get all parameters as a map
auto params = request.allParams();  // {"id": "123", ...}
```

#### Query Parameters

```cpp
// Get a query parameter by name
std::string sortBy = request.query("sort");  // "name" from "?sort=name"

// Get a query parameter with default value
std::string orderBy = request.query("order", "asc");  // "asc" if not provided

// Get a query parameter with type conversion
int page = request.query<int>("page", 1);  // Default to page 1

// Check if a query parameter exists
if (request.hasQuery("filter")) {
    // Parameter exists
}

// Get all query parameters
auto queryParams = request.allQueryParams();  // {"sort": "name", "page": "1"}
```

#### Headers

```cpp
// Get a header by name (case-insensitive)
std::string contentType = request.header("Content-Type");

// Get a header with default value
std::string acceptLanguage = request.header("Accept-Language", "en-US");

// Check if a header exists
if (request.hasHeader("Authorization")) {
    // Header exists
}

// Get all headers
auto headers = request.allHeaders();
```

#### Cookies

```cpp
// Get a cookie by name
std::string sessionId = request.cookie("session_id");

// Get a cookie with default value
std::string theme = request.cookie("theme", "light");

// Check if a cookie exists
if (request.hasCookie("user_preferences")) {
    // Cookie exists
}

// Get all cookies
auto cookies = request.allCookies();
```

#### Request Body

```cpp
// Get raw request body as string
std::string rawBody = request.body();

// Parse JSON body
if (request.contentType() == "application/json") {
    auto jsonBody = request.jsonBody();
    std::string name = jsonBody["name"].get<std::string>();
    std::string email = jsonBody["email"].get<std::string>();
}

// Parse form data
if (request.contentType() == "application/x-www-form-urlencoded") {
    auto formData = request.formData();
    std::string username = formData["username"];
    std::string password = formData["password"];
}
```

#### Files

```cpp
// Check if the request has file uploads
if (request.hasFiles()) {
    // Get an uploaded file by field name
    auto file = request.file("profile_image");
    
    // Get file properties
    std::string fileName = file.name();          // Original filename
    std::string contentType = file.mimeType();   // "image/jpeg"
    size_t size = file.size();                   // File size in bytes
    
    // Move the file to a permanent location
    file.moveTo("/path/to/uploads/" + file.name());
    
    // Or get the file content as a stream
    auto& stream = file.stream();
    // Process the stream...
}

// Get all uploaded files for a field (multiple uploads with the same name)
auto files = request.files("documents");
for (const auto& file : files) {
    // Process each file
}
```

#### Client Information

```cpp
// Get client IP address
std::string ip = request.clientIp();

// Get the hostname from the request
std::string host = request.host();  // "example.com"

// Get the port
int port = request.port();  // 443

// Check if the request is HTTPS
bool isSecure = request.isSecure();
```

#### Attributes

Request attributes enable passing data between middleware components and controller actions:

```cpp
// Set an attribute
request = request.withAttribute("user_id", 123);

// Get an attribute
if (request.hasAttribute("user_id")) {
    int userId = request.attribute<int>("user_id");
}

// Get all attributes
auto attributes = request.allAttributes();
```

### Request Validation

Boson provides convenient methods to validate request data:

```cpp
// Validate request data
auto validation = request.validate({
    {"name", "required|string|max:255"},
    {"email", "required|email|unique:users,email"},
    {"age", "required|integer|min:18"},
    {"website", "url|nullable"},
    {"profile_image", "file|image|max:2048"}
});

// Check if validation passed
if (validation.fails()) {
    // Get validation errors
    auto errors = validation.errors();
    return boson::Response::unprocessableEntity()
        .json(errors);
}

// Get validated data
auto validated = validation.validated();
std::string name = validated["name"].get<std::string>();
int age = validated["age"].get<int>();
```

## The Response Object

The `boson::Response` class represents an HTTP response that will be sent back to the client. It provides methods to set status codes, headers, cookies, and the response body.

### Creating Response Objects

Boson offers several factory methods to create common response types:

```cpp
// Basic responses
auto okResponse = boson::Response::ok("Hello, World!");
auto notFoundResponse = boson::Response::notFound();
auto serverErrorResponse = boson::Response::serverError();

// JSON responses
auto jsonResponse = boson::Response::json({
    {"id", 123},
    {"name", "John Doe"},
    {"email", "john@example.com"}
});

// File responses
auto fileResponse = boson::Response::file("/path/to/document.pdf", "application/pdf");

// View responses
auto viewResponse = boson::Response::view("users.show", {
    {"user", userModel},
    {"title", "User Profile"}
});
```

### Response Status Codes

```cpp
// Create a response with specific status code
auto response = boson::Response(200);

// Set status code on an existing response
response.statusCode(404);

// Convenience methods for common status codes
auto okResponse = boson::Response::ok();                    // 200
auto createdResponse = boson::Response::created();          // 201
auto acceptedResponse = boson::Response::accepted();        // 202
auto noContentResponse = boson::Response::noContent();      // 204
auto badRequestResponse = boson::Response::badRequest();    // 400
auto unauthorizedResponse = boson::Response::unauthorized();// 401
auto forbiddenResponse = boson::Response::forbidden();      // 403
auto notFoundResponse = boson::Response::notFound();        // 404
auto serverErrorResponse = boson::Response::serverError();  // 500
```

### Response Body

```cpp
// Set the response body
response.body("Hello, World!");

// Chain methods for fluent API
response.statusCode(200)
        .body("Success!");

// JSON body
response.json({
    {"success", true},
    {"data", {
        {"id", 123},
        {"name", "Product Name"},
        {"price", 19.99}
    }}
});
```

### Response Headers

```cpp
// Set a single header
response.header("Content-Type", "text/html");

// Set multiple headers
response.headers({
    {"Content-Type", "application/json"},
    {"X-Custom-Header", "Value"},
    {"Cache-Control", "no-cache, no-store"}
});

// Common headers have convenience methods
response.contentType("application/json");
response.cacheControl("public, max-age=3600");
```

### Response Cookies

```cpp
// Set a basic cookie
response.cookie("session_id", "abc123");

// Set a cookie with options
response.cookie("user_preferences", "theme=dark", {
    {"expires", 3600},            // Lifetime in seconds
    {"path", "/"},                // Cookie path
    {"domain", "example.com"},    // Cookie domain
    {"secure", true},             // HTTPS only
    {"httpOnly", true},           // Not accessible to JavaScript
    {"sameSite", "lax"}           // SameSite policy
});

// Set a raw cookie string
response.rawCookie("CookieName=CookieValue; Path=/; HttpOnly");

// Remove a cookie
response.removeCookie("old_cookie");
```

### Redirects

```cpp
// Simple redirect
auto redirectResponse = boson::Response::redirect("/dashboard");

// Redirect with status code
auto temporaryRedirectResponse = boson::Response::redirect("/login", 307);

// Redirect to a named route
auto namedRouteRedirect = boson::Response::redirectToRoute("user.profile", {{"id", "123"}});

// Redirect back to the previous page
auto backRedirect = boson::Response::back();

// Redirect with flash data
auto redirectWithFlash = boson::Response::redirect("/dashboard")
    .withFlash("success", "Profile updated successfully!");
```

### File Downloads

```cpp
// File download with automatic content type detection
auto fileResponse = boson::Response::download("/path/to/file.pdf");

// File download with custom filename
auto namedFileResponse = boson::Response::download("/path/to/file.pdf", "report.pdf");

// File download with content type and disposition
auto customFileResponse = boson::Response::download("/path/to/file.pdf")
    .contentType("application/pdf")
    .header("Content-Disposition", "attachment; filename=\"annual-report.pdf\"");
```

### Streaming Responses

For large responses, you can use streaming to avoid loading the entire content into memory:

```cpp
// Create a streaming response
auto streamingResponse = boson::Response::stream([](boson::ResponseStream& stream) {
    // Stream content in chunks
    stream.write("<html>");
    stream.write("<body>");
    
    // Generate large content in chunks
    for (int i = 0; i < 1000; i++) {
        stream.write("<p>Line " + std::to_string(i) + "</p>");
        
        // Flush periodically to send data to the client
        if (i % 100 == 0) {
            stream.flush();
        }
    }
    
    stream.write("</body>");
    stream.write("</html>");
});

// Set headers for streaming response
streamingResponse.contentType("text/html");
```

### Server-Sent Events

```cpp
// Create a Server-Sent Events response
auto sseResponse = boson::Response::sse([](boson::SseStream& stream) {
    // Send periodic updates
    for (int i = 0; i < 10; i++) {
        // Send a named event with data
        stream.event("update", "{\"progress\":" + std::to_string(i * 10) + "}");
        
        // Sleep between events
        std::this_thread::sleep_for(std::chrono::seconds(1));
    }
    
    // Send a completion event
    stream.event("complete", "{\"status\":\"done\"}");
});
```

### Response Macros

Boson allows defining response macros for commonly used response patterns:

```cpp
// Define a response macro
boson::Response::macro("apiSuccess", [](auto data) {
    return boson::Response::json({
        {"success", true},
        {"data", data},
        {"timestamp", std::time(nullptr)}
    });
});

boson::Response::macro("apiError", [](int code, std::string message) {
    return boson::Response::json({
        {"success", false},
        {"error", {
            {"code", code},
            {"message", message}
        }},
        {"timestamp", std::time(nullptr)}
    }).statusCode(code);
});

// Use the macros
auto successResponse = boson::Response::apiSuccess(userData);
auto errorResponse = boson::Response::apiError(404, "User not found");
```

## Working with Request and Response Together

Here's a typical example of processing a request and generating a response in a controller action:

```cpp
boson::Response updateUser(const boson::Request& request) {
    // Get route parameter
    int userId = request.param<int>("id");
    
    // Validate input
    auto validation = request.validate({
        {"name", "required|string|max:255"},
        {"email", "required|email|unique:users,email," + std::to_string(userId)},
        {"role", "in:admin,user,editor"}
    });
    
    if (validation.fails()) {
        return boson::Response::unprocessableEntity()
            .json(validation.errors());
    }
    
    try {
        // Get validated data
        auto data = validation.validated();
        
        // Update the user
        auto user = userService_->updateUser(
            userId,
            data["name"].get<std::string>(),
            data["email"].get<std::string>(),
            data["role"].get<std::string>("user")
        );
        
        // Return success response
        return boson::Response::json(user)
            .header("X-Resource-Version", user.version())
            .cookie("last_action", "update_user");
            
    } catch (const UserNotFoundException& e) {
        return boson::Response::notFound()
            .json({{"error", "User not found"}});
            
    } catch (const std::exception& e) {
        logger_->error("Failed to update user: {}", e.what());
        
        return boson::Response::serverError()
            .json({{"error", "Failed to update user"}});
    }
}
```

## Testing Requests and Responses

Boson provides utilities for testing controllers with requests and verifying responses:

```cpp
#include <boson/testing/http_tester.hpp>
#include <gtest/gtest.h>

// In your test
TEST(UserControllerTest, UpdateUser) {
    // Create the application
    boson::Application app;
    app.registerController<UserController>();
    
    // Create HTTP tester
    boson::HttpTester tester(app);
    
    // Make a request to the controller
    auto response = tester.put("/users/123")
        .json({
            {"name", "Updated Name"},
            {"email", "updated@example.com"},
            {"role", "editor"}
        })
        .send();
    
    // Assert response properties
    ASSERT_EQ(200, response.statusCode());
    ASSERT_EQ("application/json", response.contentType());
    
    // Assert JSON response
    auto json = response.jsonBody();
    ASSERT_EQ(123, json["id"].get<int>());
    ASSERT_EQ("Updated Name", json["name"].get<std::string>());
    ASSERT_EQ("updated@example.com", json["email"].get<std::string>());
}
```

## Best Practices

### Request Handling

1. **Validate all input**: Always validate incoming request data before using it.
2. **Use type-safe parameter access**: Use `request.param<T>()` for automatic type conversion.
3. **Keep raw data access minimal**: Prefer validated data over direct request access.
4. **Be careful with file uploads**: Always validate file types and sizes.
5. **Check content types**: Verify request content types before parsing bodies.

### Response Generation

1. **Set appropriate status codes**: Use the correct HTTP status code for each response.
2. **Set content types**: Always specify the content type of your responses.
3. **Consider security headers**: Set appropriate security headers for all responses.
4. **Use streaming for large responses**: Avoid loading large files into memory.
5. **Be consistent**: Maintain a consistent response format across your API.

## Next Steps

Now that you understand request and response handling in Boson, explore these related topics:

1. Learn about [Error Handling](error-handling.md) for robust error management
2. Explore [Middleware](middleware.md) for processing requests and responses
3. Understand [Controllers](controllers.md) for organizing route handlers