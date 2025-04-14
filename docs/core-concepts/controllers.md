---
sidebar_position: 3
title: Controllers
---

# Controllers in Boson

Controllers in Boson provide a structured way to organize related route handlers. This guide explains how to create and use controllers effectively in your Boson applications.

## What are Controllers?

Controllers are classes that group related route handlers together. They follow the object-oriented approach to organize your application logic based on resources or features. Controllers help maintain a clean separation between routing and business logic.

## Creating a Basic Controller

To create a controller, extend the `boson::Controller` base class:

```cpp
#include <boson/controller.hpp>
#include <boson/response.hpp>

class HomeController : public boson::Controller {
public:
    void registerRoutes() override {
        // Register routes with their handlers
        GET("/", &HomeController::index);
        GET("/about", &HomeController::about);
        GET("/contact", &HomeController::contact);
    }

    // Handler methods
    boson::Response index(const boson::Request& request) {
        return boson::Response::ok("Welcome to the homepage!");
    }
    
    boson::Response about(const boson::Request& request) {
        return boson::Response::ok("About us page");
    }
    
    boson::Response contact(const boson::Request& request) {
        return boson::Response::ok("Contact us page");
    }
};
```

## Registering Controllers

After creating a controller, register it with your application:

```cpp
#include <boson/boson.hpp>
#include "controllers/home_controller.hpp"

int main() {
    boson::Application app;
    
    // Register the controller
    app.registerController<HomeController>();
    
    boson::Server server(app);
    server.listen(8080);
    return 0;
}
```

## Route Registration Methods

Controllers offer convenient methods for registering routes with different HTTP methods:

```cpp
void registerRoutes() override {
    // HTTP method-specific registration methods
    GET("/products", &ProductController::index);
    POST("/products", &ProductController::store);
    PUT("/products/{id}", &ProductController::update);
    PATCH("/products/{id}", &ProductController::partialUpdate);
    DELETE("/products/{id}", &ProductController::destroy);
    
    // For special cases or custom methods
    ROUTE("OPTIONS", "/products", &ProductController::options);
}
```

## Controller Route Prefixes

You can set a prefix for all routes in a controller:

```cpp
class ApiController : public boson::Controller {
public:
    // Set a route prefix for all routes in this controller
    ApiController() : boson::Controller("/api") {}
    
    void registerRoutes() override {
        // These routes will be prefixed with "/api"
        GET("/users", &ApiController::getUsers);         // Becomes "/api/users"
        GET("/products", &ApiController::getProducts);   // Becomes "/api/products"
    }
    
    // Handler methods
    boson::Response getUsers(const boson::Request& request);
    boson::Response getProducts(const boson::Request& request);
};
```

## Controller Middleware

You can apply middleware to all routes in a controller:

```cpp
class AdminController : public boson::Controller {
public:
    void registerMiddleware() override {
        // Apply auth middleware to all routes in this controller
        useMiddleware<AuthMiddleware>();
        
        // Apply multiple middleware
        useMiddleware<LoggingMiddleware>();
        useMiddleware<RateLimitMiddleware>(100); // With constructor arguments
    }
    
    void registerRoutes() override {
        // All these routes will go through the middleware
        GET("/dashboard", &AdminController::dashboard);
        GET("/users", &AdminController::users);
        GET("/settings", &AdminController::settings);
    }
    
    // Handler methods
    boson::Response dashboard(const boson::Request& request);
    boson::Response users(const boson::Request& request);
    boson::Response settings(const boson::Request& request);
};
```

## Route-Specific Middleware

You can also apply middleware to specific routes within a controller:

```cpp
void registerRoutes() override {
    // Apply middleware to a specific route
    GET("/reports", 
        &ReportController::generate,
        std::make_shared<CacheMiddleware>(3600) // Cache for 1 hour
    );
    
    // Multiple middleware for a route
    GET("/exports", 
        std::vector<std::shared_ptr<boson::Middleware>>{
            std::make_shared<RateLimitMiddleware>(10),
            std::make_shared<CompressionMiddleware>()
        },
        &ReportController::exportData
    );
    
    // Regular routes without additional middleware
    GET("/dashboard", &ReportController::dashboard);
}
```

## RESTful Resource Controllers

Boson supports RESTful resource controllers with a convenient registration method:

```cpp
class UserController : public boson::Controller {
public:
    void registerRoutes() override {
        // Register all RESTful resource routes at once
        RESOURCE("users", true); // true enables the named routes
    }
    
    // Standard RESTful action methods
    boson::Response index(const boson::Request& request);   // GET /users
    boson::Response create(const boson::Request& request);  // GET /users/create
    boson::Response store(const boson::Request& request);   // POST /users
    boson::Response show(const boson::Request& request);    // GET /users/{id}
    boson::Response edit(const boson::Request& request);    // GET /users/{id}/edit
    boson::Response update(const boson::Request& request);  // PUT/PATCH /users/{id}
    boson::Response destroy(const boson::Request& request); // DELETE /users/{id}
};
```

You can also register only specific resource actions:

```cpp
void registerRoutes() override {
    // Register only specific actions
    RESOURCE("comments", {"index", "store", "destroy"});
}
```

## Accessing Services in Controllers

Controllers can access shared services through dependency injection:

```cpp
class UserController : public boson::Controller {
public:
    // Constructor with dependency injection
    UserController(
        std::shared_ptr<UserService> userService,
        std::shared_ptr<AuthService> authService
    ) : userService_(std::move(userService)), 
        authService_(std::move(authService)) {}
    
    void registerRoutes() override {
        GET("/users", &UserController::getUsers);
        GET("/users/{id}", &UserController::getUser);
    }
    
    boson::Response getUsers(const boson::Request& request) {
        // Use the injected service
        auto users = userService_->getAllUsers();
        return boson::Response::json(users);
    }
    
    boson::Response getUser(const boson::Request& request) {
        int id = request.param<int>("id");
        
        // Verify authorization using authService
        if (!authService_->canViewUser(request.user(), id)) {
            return boson::Response::forbidden();
        }
        
        auto user = userService_->getUserById(id);
        if (!user) {
            return boson::Response::notFound();
        }
        
        return boson::Response::json(*user);
    }

private:
    std::shared_ptr<UserService> userService_;
    std::shared_ptr<AuthService> authService_;
};
```

To register a controller with dependencies:

```cpp
// Create service instances
auto userService = std::make_shared<UserService>(db);
auto authService = std::make_shared<AuthService>();

// Register controller with dependencies
app.registerController<UserController>(userService, authService);
```

## Accessing the Application

Controllers can access the application instance:

```cpp
void initialize() override {
    // Called after the controller is registered
    // You can access the application instance here
    auto& config = application().config();
    logger_ = application().logger();
    
    // Or get shared services
    dbConnection_ = application().getService<DatabaseConnection>();
}
```

## Controller Lifecycle Methods

Boson controllers have several lifecycle methods you can override:

```cpp
class MyController : public boson::Controller {
public:
    // Called when the controller is constructed
    MyController() {
        // Initialization work
    }
    
    // Called after the controller is registered with the application
    void initialize() override {
        // Access application services
    }
    
    // Register middleware for this controller
    void registerMiddleware() override {
        // Add middleware
    }
    
    // Register routes for this controller
    void registerRoutes() override {
        // Define routes
    }
    
    // Called before the controller is destroyed
    ~MyController() {
        // Cleanup work
    }
};
```

## Request Validation

Controllers can validate incoming requests:

```cpp
boson::Response store(const boson::Request& request) {
    // Validate request data
    auto result = validate(request, {
        {"name", "required|string|max:255"},
        {"email", "required|email|unique:users,email"},
        {"password", "required|string|min:8|confirmed"},
        {"age", "nullable|integer|min:18"}
    });
    
    // Check if validation failed
    if (result.failed()) {
        return boson::Response::unprocessableEntity()
            .json(result.errors());
    }
    
    // Get validated data
    auto data = result.validated();
    
    // Continue with storing the user
    auto user = userService_->createUser(
        data["name"].get<std::string>(),
        data["email"].get<std::string>(),
        data["password"].get<std::string>(),
        data["age"].get<int>(0)  // Default to 0 if age is null
    );
    
    return boson::Response::created()
        .json(user);
}
```

## Controller Helpers

Boson provides helper methods for common controller tasks:

```cpp
// Redirecting to a URL or named route
boson::Response store(const boson::Request& request) {
    // Process the form submission...
    
    // Redirect to another URL
    return redirect("/users");
    
    // Or redirect to a named route with parameters
    return redirectToRoute("users.show", {{"id", "123"}});
    
    // Redirect back to the previous page
    return redirectBack();
}

// Flash messages for redirects
boson::Response update(const boson::Request& request) {
    // Update the resource...
    
    // Redirect with a flash message
    return redirect("/dashboard")
        .withFlash("success", "Profile updated successfully!");
}

// Returning views
boson::Response show(const boson::Request& request) {
    int id = request.param<int>("id");
    auto user = userService_->getUserById(id);
    
    // Return a view with data
    return view("users.show", {
        {"user", user},
        {"title", "User Profile"}
    });
}
```

## Error Handling in Controllers

Controllers can include error handling logic:

```cpp
boson::Response getUserById(const boson::Request& request) {
    int id = request.param<int>("id");
    
    try {
        auto user = userService_->getUserById(id);
        return boson::Response::json(user);
    } catch (const UserNotFoundException& e) {
        return boson::Response::notFound()
            .json({{"error", "User not found"}});
    } catch (const DatabaseException& e) {
        // Log the error
        logger_->error("Database error: {}", e.what());
        
        return boson::Response::serverError()
            .json({{"error", "Internal server error"}});
    }
}
```

## Controller Testing

Boson makes it easy to test controllers:

```cpp
#include <boson/testing/controller_tester.hpp>
#include <gtest/gtest.h>

// Test fixture
class UserControllerTest : public ::testing::Test {
protected:
    UserControllerTest() {
        // Create mock services
        auto mockUserService = std::make_shared<MockUserService>();
        auto mockAuthService = std::make_shared<MockAuthService>();
        
        // Create the controller with mock services
        controller_ = std::make_shared<UserController>(
            mockUserService,
            mockAuthService
        );
        
        // Create controller tester
        tester_ = std::make_unique<boson::ControllerTester>(controller_);
    }
    
    std::shared_ptr<UserController> controller_;
    std::unique_ptr<boson::ControllerTester> tester_;
};

// Test method
TEST_F(UserControllerTest, GetUserReturnsCorrectUser) {
    // Call the controller method with a test request
    auto request = boson::Request::fromUrl("GET", "/users/123");
    auto response = tester_->callAction(&UserController::getUser, request);
    
    // Assert the response
    ASSERT_EQ(200, response.statusCode());
    ASSERT_EQ("application/json", response.contentType());
    
    // Parse and check the JSON response
    auto json = response.jsonBody();
    ASSERT_EQ(123, json["id"].get<int>());
    ASSERT_EQ("Test User", json["name"].get<std::string>());
}
```

## Best Practices

1. **Single Responsibility**: Keep controllers focused on a single resource or feature
2. **Thin Controllers**: Keep business logic in services, use controllers just for HTTP handling
3. **Consistent Naming**: Follow a consistent naming convention for controller methods
4. **Resource Organization**: Structure controllers around resources for RESTful design
5. **Dependency Injection**: Use constructor injection for controller dependencies
6. **Validation**: Always validate incoming request data
7. **Error Handling**: Implement proper error handling in controller methods
8. **Testing**: Write tests for all controller actions

## Next Steps

Now that you understand controllers in Boson, explore these related topics:

1. Learn about [Middleware](middleware.md) for processing requests and responses
2. Understand [Request and Response](request-response.md) objects in detail
3. Explore [Error Handling](error-handling.md) for robust error management