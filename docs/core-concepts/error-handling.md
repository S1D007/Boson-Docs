---
sidebar_position: 6
title: Error Handling
---

# Error Handling in Boson

Effective error handling is crucial for creating robust web applications. Boson provides a comprehensive error handling system that helps you gracefully manage exceptions and present appropriate responses to users. This guide explains how to use Boson's error handling features.

## Error Handling Architecture

Boson's error handling architecture operates at multiple levels:

1. **Global Error Handlers**: Application-wide handlers for specific exception types
2. **Middleware Error Handling**: Error handling in middleware pipeline
3. **Controller Error Handling**: Try-catch blocks in controller actions
4. **Route-Specific Error Handling**: Error handlers for specific routes
5. **Fallback Error Handlers**: Default handlers when no specific handler exists

## Exception Types

Boson includes several built-in exception types:

```cpp
// Base exception class for all Boson exceptions
class BosonException : public std::runtime_error { /* ... */ };

// HTTP exceptions with status codes
class HttpException : public BosonException { /* ... */ };
class NotFoundException : public HttpException { /* ... */ };
class UnauthorizedException : public HttpException { /* ... */ };
class ForbiddenException : public HttpException { /* ... */ };
class ValidationException : public HttpException { /* ... */ };

// Application logic exceptions
class DatabaseException : public BosonException { /* ... */ };
class ConfigurationException : public BosonException { /* ... */ };
class FileSystemException : public BosonException { /* ... */ };

// Request processing exceptions
class InvalidParameterException : public BosonException { /* ... */ };
class RouteNotFoundException : public NotFoundException { /* ... */ };
class MethodNotAllowedException : public HttpException { /* ... */ };
```

## Global Error Handlers

Register error handlers at the application level to handle specific exception types:

```cpp
#include <boson/boson.hpp>

int main() {
    boson::Application app;
    
    // Handle 404 Not Found exceptions
    app.handleException<boson::NotFoundException>([](const auto& exception, const boson::Request& request) {
        return boson::Response::view("errors.not-found", {{"path", request.path()}})
            .statusCode(404);
    });
    
    // Handle validation exceptions
    app.handleException<boson::ValidationException>([](const auto& exception, const boson::Request& request) {
        // Return validation errors as JSON or HTML based on the request
        if (request.wantsJson()) {
            return boson::Response::json({
                {"error", "Validation failed"},
                {"fields", exception.errors()}
            }).statusCode(422);
        }
        
        return boson::Response::view("errors.validation", {
            {"errors", exception.errors()},
            {"old", request.all()}
        }).statusCode(422);
    });
    
    // Handle database exceptions
    app.handleException<boson::DatabaseException>([](const auto& exception, const boson::Request& request) {
        // Log the error
        app.logger()->error("Database error: {}", exception.what());
        
        return boson::Response::serverError()
            .view("errors.database-error");
    });
    
    // Catch-all handler for any unhandled exceptions
    app.handleException<std::exception>([](const auto& exception, const boson::Request& request) {
        // Log the unexpected error
        app.logger()->critical("Unhandled exception: {}", exception.what());
        
        return boson::Response::serverError()
            .view("errors.generic");
    });
    
    // Rest of your application setup
    // ...
    
    boson::Server server(app);
    server.listen(8080);
    return 0;
}
```

## Custom Exception Classes

Create custom exception classes for your application's specific needs:

```cpp
#include <boson/exceptions.hpp>

// Custom domain exception
class UserNotFoundException : public boson::NotFoundException {
public:
    UserNotFoundException(int userId)
        : NotFoundException("User with ID " + std::to_string(userId) + " not found"),
          userId_(userId) {}
          
    int userId() const { return userId_; }
    
private:
    int userId_;
};

// Custom application exception
class PaymentFailedException : public boson::HttpException {
public:
    PaymentFailedException(std::string message, std::string errorCode)
        : HttpException(message, 402),  // 402 Payment Required
          errorCode_(std::move(errorCode)) {}
          
    const std::string& errorCode() const { return errorCode_; }
    
private:
    std::string errorCode_;
};
```

Register handlers for your custom exceptions:

```cpp
// Handle user not found exceptions
app.handleException<UserNotFoundException>([](const auto& exception, const boson::Request& request) {
    return boson::Response::notFound()
        .view("errors.user-not-found", {{"userId", exception.userId()}});
});

// Handle payment failures
app.handleException<PaymentFailedException>([](const auto& exception, const boson::Request& request) {
    return boson::Response::json({
        {"error", exception.what()},
        {"errorCode", exception.errorCode()},
        {"paymentUrl", "/payments/retry"}
    }).statusCode(402);
});
```

## Throwing Exceptions

You can throw exceptions at any point in your application:

```cpp
boson::Response getUserProfile(const boson::Request& request) {
    int userId = request.param<int>("id");
    
    auto user = userRepository_->findById(userId);
    
    if (!user) {
        // Throw a custom exception
        throw UserNotFoundException(userId);
    }
    
    // Check authorization
    if (!request.user() || request.user()->id() != userId) {
        // Throw a built-in exception
        throw boson::ForbiddenException("You don't have permission to view this profile");
    }
    
    return boson::Response::view("users.profile", {{"user", *user}});
}
```

## HTTP Exception Shortcuts

Boson provides convenient methods for generating HTTP error responses without throwing exceptions:

```cpp
boson::Response updateUser(const boson::Request& request) {
    int userId = request.param<int>("id");
    
    // Find the user
    auto user = userRepository_->findById(userId);
    if (!user) {
        // Return a 404 response without throwing
        return boson::Response::notFound()
            .json({{"error", "User not found"}});
    }
    
    // Check permissions
    if (!request.user()->canEdit(user)) {
        // Return a 403 response without throwing
        return boson::Response::forbidden()
            .json({{"error", "You don't have permission to edit this user"}});
    }
    
    // Validate input
    auto validation = request.validate({
        {"name", "required|string|max:255"},
        {"email", "required|email|unique:users,email," + std::to_string(userId)}
    });
    
    if (validation.fails()) {
        // Return a 422 validation error response
        return boson::Response::unprocessableEntity()
            .json(validation.errors());
    }
    
    // Process the update
    // ...
    
    return boson::Response::ok(user);
}
```

## Middleware Error Handling

Middleware can catch and handle exceptions thrown by other middleware or route handlers:

```cpp
class ErrorHandlingMiddleware : public boson::Middleware {
public:
    boson::Response process(const boson::Request& request, MiddlewareNext next) override {
        try {
            // Process the request through the middleware chain
            return next(request);
        } catch (const boson::DatabaseException& e) {
            // Log the error
            logger_->error("Database error: {}", e.what());
            
            // Return an error response
            return boson::Response::serverError()
                .json({{"error", "A database error occurred"}});
        } catch (const std::exception& e) {
            // Log the error
            logger_->error("Unhandled exception: {}", e.what());
            
            // Return a generic error response
            return boson::Response::serverError()
                .json({{"error", "An unexpected error occurred"}});
        }
    }
};
```

Register the error handling middleware early in your middleware chain:

```cpp
// Register error handling middleware
app.useMiddleware<ErrorHandlingMiddleware>();

// Register other middleware and routes
// ...
```

## Route-Specific Error Handling

You can register error handlers for specific routes or route groups:

```cpp
// Create a route group for API endpoints
auto apiGroup = app.group("/api");

// Set a specific error handler for this group
apiGroup->onError([](const std::exception& e, const boson::Request& request) {
    // Always return JSON errors for API routes
    return boson::Response::json({
        {"error", e.what()},
        {"path", request.path()},
        {"timestamp", std::time(nullptr)}
    }).statusCode(500);
});

// Add routes to the group
apiGroup->get("/users", getUsersHandler);
apiGroup->post("/users", createUserHandler);
```

## HTTP Status Code Handlers

Register handlers for specific HTTP status codes:

```cpp
// Handler for 404 Not Found status
app.statusHandler(404, [](const boson::Request& request) {
    return boson::Response::view("errors.not-found", {
        {"path", request.path()},
        {"requestId", request.id()}
    }).statusCode(404);
});

// Handler for 500 Internal Server Error status
app.statusHandler(500, [](const boson::Request& request) {
    return boson::Response::view("errors.server-error", {
        {"requestId", request.id()}
    }).statusCode(500);
});

// Handler for maintenance mode (503 Service Unavailable)
app.statusHandler(503, [](const boson::Request& request) {
    return boson::Response::view("errors.maintenance", {
        {"estimatedDowntime", "2 hours"}
    }).statusCode(503)
      .header("Retry-After", "7200");
});
```

## Error Pages

Create custom error pages for different HTTP status codes:

```
templates/
└── errors/
    ├── 404.html       # Not Found
    ├── 500.html       # Server Error
    ├── 403.html       # Forbidden
    ├── 401.html       # Unauthorized
    ├── maintenance.html # For maintenance mode
    └── generic.html   # Fallback error page
```

Register these error pages with the application:

```cpp
// Set up error pages
app.errorPage(404, "errors/404.html");
app.errorPage(500, "errors/500.html");
app.errorPage(403, "errors/403.html");
app.errorPage(401, "errors/401.html");
app.errorPage(503, "errors/maintenance.html");
```

## Logging Errors

Boson integrates with logging systems to record errors:

```cpp
// Configure application logger
app.configureLogger([](boson::Logger& logger) {
    // Set log level
    logger.setLevel(boson::LogLevel::Info);
    
    // Add console output
    logger.addHandler(std::make_unique<boson::ConsoleLogHandler>());
    
    // Add file output
    logger.addHandler(std::make_unique<boson::FileLogHandler>("logs/app.log"));
    
    // Add error-specific file logger
    auto errorLogger = std::make_unique<boson::FileLogHandler>("logs/errors.log");
    errorLogger->setMinLevel(boson::LogLevel::Error);
    logger.addHandler(std::move(errorLogger));
});

// Use the logger in your error handlers
app.handleException<std::exception>([&app](const auto& exception, const boson::Request& request) {
    // Get logger from application
    auto logger = app.logger();
    
    // Log the error with context
    logger->error("Exception in request to {}: {}", 
                  request.path(), 
                  exception.what());
    
    // Log extra details for severe errors
    if (typeid(exception) == typeid(boson::DatabaseException) || 
        typeid(exception) == typeid(boson::ServerException)) {
        logger->critical("Stack trace: {}", boson::getStackTrace());
        logger->critical("Request details: {}", request.toString());
    }
    
    return boson::Response::serverError();
});
```

## Error Reporting Services

Integrate with error reporting services to track errors in production:

```cpp
// Create error reporting middleware
class ErrorReportingMiddleware : public boson::Middleware {
public:
    ErrorReportingMiddleware(std::shared_ptr<ErrorReportService> reportService)
        : reportService_(std::move(reportService)) {}
        
    boson::Response process(const boson::Request& request, MiddlewareNext next) override {
        try {
            // Process the request
            return next(request);
        } catch (const std::exception& e) {
            // Report the error to the service
            reportService_->report(e, {
                {"url", request.url()},
                {"method", request.method()},
                {"user", request.user() ? request.user()->id() : "guest"},
                {"timestamp", std::time(nullptr)}
            });
            
            // Re-throw to let other handlers process it
            throw;
        }
    }
    
private:
    std::shared_ptr<ErrorReportService> reportService_;
};

// Set up the error reporting service
auto errorReporter = std::make_shared<SentryErrorReportService>(
    "https://your-sentry-dsn.ingest.sentry.io/project-id"
);

// Add the middleware
app.useMiddleware<ErrorReportingMiddleware>(errorReporter);
```

## Validation Errors

Handle validation errors with Boson's validation system:

```cpp
boson::Response createUser(const boson::Request& request) {
    // Validate request data
    auto validation = request.validate({
        {"name", "required|string|max:255"},
        {"email", "required|email|unique:users,email"},
        {"password", "required|string|min:8|confirmed"},
        {"role", "in:admin,user,guest"}
    });
    
    // Check if validation failed
    if (validation.fails()) {
        // For API requests, return JSON errors
        if (request.wantsJson()) {
            return boson::Response::unprocessableEntity()
                .json({
                    {"message", "Validation failed"},
                    {"errors", validation.errors()}
                });
        }
        
        // For form submissions, redirect back with errors and input
        return boson::Response::redirect()
            .back()
            .withErrors(validation)
            .withInput(request.except({"password"}));
    }
    
    // Continue with valid data
    auto data = validation.validated();
    
    // Create user...
    
    return boson::Response::created()
        .json(newUser);
}
```

## Handling Maintenance Mode

Boson allows you to put your application into maintenance mode:

```cpp
// Check if app is in maintenance mode
if (app.isDownForMaintenance()) {
    // Return maintenance response
}

// Enable maintenance mode programmatically
app.downForMaintenance(true);

// Specify allowed IPs during maintenance
app.setMaintenanceWhitelist({"192.168.1.1", "10.0.0.5"});

// Create a custom maintenance mode handler
app.setMaintenanceHandler([](const boson::Request& request) {
    return boson::Response::view("errors.maintenance", {
        {"startedAt", "2025-04-14 09:00 UTC"},
        {"estimatedDuration", "2 hours"}
    }).statusCode(503)
      .header("Retry-After", "7200");
});
```

## Error Handling for Different Environments

Adjust error handling based on the current environment:

```cpp
// Configure environment-specific error handling
if (app.environment() == "production") {
    // In production, hide detailed errors
    app.hideErrorDetails(true);
    
    // Use custom error pages
    app.useCustomErrorPages(true);
    
    // Report errors to external service
    app.useMiddleware<ErrorReportingMiddleware>(errorReporter);
} else {
    // In development, show detailed errors
    app.hideErrorDetails(false);
    
    // Use debug error pages with stack traces
    app.useCustomErrorPages(false);
    
    // Enable whoops-style error pages
    app.useWhoopsErrorPages(true);
}
```

## Error Response Formats

Customize the format of error responses based on the request type:

```cpp
// Create a middleware for handling the error response format
class ErrorFormatMiddleware : public boson::Middleware {
public:
    boson::Response process(const boson::Request& request, MiddlewareNext next) override {
        try {
            // Process the request normally
            return next(request);
        } catch (const std::exception& e) {
            // Determine response format based on Accept header
            if (request.wantsJson()) {
                return jsonErrorResponse(e, request);
            } else if (request.header("Accept").find("text/xml") != std::string::npos) {
                return xmlErrorResponse(e, request);
            } else {
                return htmlErrorResponse(e, request);
            }
        }
    }
    
private:
    boson::Response jsonErrorResponse(const std::exception& e, const boson::Request& request);
    boson::Response xmlErrorResponse(const std::exception& e, const boson::Request& request);
    boson::Response htmlErrorResponse(const std::exception& e, const boson::Request& request);
};

// Register the middleware
app.useMiddleware<ErrorFormatMiddleware>();
```

## Testing Error Handling

Boson provides tools for testing error handling:

```cpp
#include <boson/testing/http_tester.hpp>
#include <gtest/gtest.h>

// Test exception handling
TEST(ErrorHandlingTest, HandlesNotFoundException) {
    // Create a test application
    boson::Application app;
    
    // Register a controller that throws an exception
    app.route("/users/{id}", [](const boson::Request& request) {
        throw boson::NotFoundException("User not found: " + request.param("id"));
    });
    
    // Register an exception handler
    app.handleException<boson::NotFoundException>([](const auto& exception, const boson::Request&) {
        return boson::Response::notFound()
            .json({{"error", exception.what()}});
    });
    
    // Create HTTP tester
    boson::HttpTester tester(app);
    
    // Make a request to trigger the exception
    auto response = tester.get("/users/123");
    
    // Assert the response
    ASSERT_EQ(404, response.statusCode());
    ASSERT_EQ("application/json", response.contentType());
    
    // Check the JSON response
    auto json = response.jsonBody();
    ASSERT_EQ("User not found: 123", json["error"].get<std::string>());
}
```

## Best Practices

1. **Layer your error handling**: Use a combination of specific and generic handlers.
2. **Handle exceptions appropriately**: Use appropriate HTTP status codes for different errors.
3. **Log errors with context**: Include request details, user information, and other context.
4. **Hide sensitive information**: Don't expose stack traces or internal details in production.
5. **Format errors consistently**: Use consistent error response formats across your API.
6. **Validate input early**: Catch validation errors before they cause problems deeper in your code.
7. **Provide helpful error messages**: Error messages should help users understand what went wrong.
8. **Use custom exception classes**: Create specific exception classes for your domain logic.

## Next Steps

Now that you understand error handling in Boson, explore these related topics:

1. Learn about [Logging](logging.md) for detailed application event tracking
2. Explore [Validation](validation.md) for comprehensive request data validation
3. Study [Security](../security/overview.md) for building secure applications