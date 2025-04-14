---
sidebar_position: 8
title: Dependency Injection
---

# Dependency Injection in Boson

Dependency Injection (DI) is a design pattern that helps you write more maintainable, testable, and modular code by removing hard-coded dependencies and making them configurable. Boson includes a powerful dependency injection container that manages your application's components and their dependencies.

## Understanding Dependency Injection

At its core, dependency injection is about providing a component with its dependencies rather than having the component create them itself. This approach offers several benefits:

1. **Loose coupling**: Components depend on abstractions rather than concrete implementations.
2. **Testability**: Dependencies can be easily mocked or stubbed during testing.
3. **Modularity**: Components can be developed, tested, and maintained independently.
4. **Configuration flexibility**: Implementations can be swapped without changing dependent components.

## The Boson Container

Boson's DI container is responsible for:

1. **Registering** service definitions and implementations
2. **Resolving** services when they're needed
3. **Managing** the lifecycle of service instances
4. **Autowiring** dependencies when resolving services

The container is accessible via the `Application` class:

```cpp
#include <boson/boson.hpp>

int main() {
    boson::Application app;
    
    // Access the container
    auto& container = app.container();
    
    // Use the container...
    
    return 0;
}
```

## Registering Services

You can register services with the container in several ways:

### Basic Registration

Register a service with a unique identifier:

```cpp
// Register a concrete implementation
container.bind<UserRepository>("user.repository", []() {
    return std::make_shared<MySqlUserRepository>();
});

// Register an interface with an implementation
container.bind<IUserRepository>("user.repository", []() {
    return std::make_shared<MySqlUserRepository>();
});
```

### Singleton Registration

Register a service that should be instantiated only once:

```cpp
// Register a singleton service
container.singleton<DatabaseConnection>("db.connection", []() {
    auto config = boson::config("database");
    return std::make_shared<PostgresConnection>(config);
});
```

### Instance Registration

Register an already created instance:

```cpp
// Create the instance
auto logger = std::make_shared<FileLogger>("app.log");

// Register the instance
container.instance<ILogger>("logger", logger);
```

### Factory Registration

Register a factory function that creates instances with parameters:

```cpp
// Register a factory function
container.factory<UserService>("user.service", [](const boson::Container& container, const std::string& role) {
    auto repository = container.make<IUserRepository>("user.repository");
    auto logger = container.make<ILogger>("logger");
    
    return std::make_shared<UserService>(repository, logger, role);
});
```

### Binding Interfaces to Implementations

Bind an interface to a concrete implementation:

```cpp
// Bind interface to implementation
container.bind<IMailer, SmtpMailer>();

// Bind interface to implementation with constructor arguments
container.bind<IMailer, SmtpMailer>("smtp.host", 587, true);

// Bind interface to implementation with a factory function
container.bind<IPaymentProcessor, StripeProcessor>([](const boson::Container& container) {
    auto apiKey = container.make<ConfigProvider>()->get<std::string>("services.stripe.key");
    return std::make_shared<StripeProcessor>(apiKey);
});
```

## Resolving Dependencies

Once services are registered, you can resolve them from the container:

### Basic Resolution

Resolve a registered service by its identifier:

```cpp
// Resolve a service by its identifier
auto userRepo = container.make<IUserRepository>("user.repository");

// Use the resolved service
auto user = userRepo->findById(123);
```

### Autowired Resolution

Resolve a service by its type (autowiring):

```cpp
// Resolve by type (if only one implementation is registered)
auto logger = container.make<ILogger>();

// Use the resolved service
logger->info("Application started");
```

### Constructor Resolution

Resolve a service with constructor parameters:

```cpp
// Resolve with constructor parameters
auto service = container.make<ReportGenerator>("report.generator", "pdf", true);
```

### Factory Resolution

Resolve a service using a registered factory:

```cpp
// Resolve using a factory with parameters
auto adminService = container.make<UserService>("user.service", "admin");
auto regularService = container.make<UserService>("user.service", "user");
```

## Service Providers

Service providers are a way to organize related service registrations. They help structure your application's bootstrap process and keep related services together:

```cpp
#include <boson/service_provider.hpp>

class DatabaseServiceProvider : public boson::ServiceProvider {
public:
    void register(boson::Container& container) override {
        // Register database-related services
        container.singleton<ConnectionPool>("db.pool", [](const boson::Container& container) {
            auto config = container.make<ConfigProvider>();
            auto host = config->get<std::string>("database.host", "localhost");
            auto port = config->get<int>("database.port", 5432);
            auto user = config->get<std::string>("database.username");
            auto pass = config->get<std::string>("database.password");
            auto dbName = config->get<std::string>("database.name");
            
            return std::make_shared<ConnectionPool>(host, port, user, pass, dbName);
        });
        
        container.bind<IUserRepository, MySqlUserRepository>();
        container.bind<IProductRepository, MySqlProductRepository>();
        container.bind<IOrderRepository, MySqlOrderRepository>();
    }
    
    void boot(boson::Container& container) override {
        // Perform any initialization after all providers are registered
        auto pool = container.make<ConnectionPool>("db.pool");
        pool->initialize();
    }
};

// Register the service provider with the application
app.registerServiceProvider<DatabaseServiceProvider>();
```

### Built-in Service Providers

Boson includes several built-in service providers for core functionality:

```cpp
// In your application's bootstrap process
#include <boson/providers/all.hpp>

void registerProviders(boson::Application& app) {
    // Register core service providers
    app.registerServiceProvider<boson::LogServiceProvider>();
    app.registerServiceProvider<boson::DatabaseServiceProvider>();
    app.registerServiceProvider<boson::CacheServiceProvider>();
    app.registerServiceProvider<boson::EventServiceProvider>();
    app.registerServiceProvider<boson::ValidationServiceProvider>();
    app.registerServiceProvider<boson::ViewServiceProvider>();
    app.registerServiceProvider<boson::MailServiceProvider>();
    app.registerServiceProvider<boson::QueueServiceProvider>();
    
    // Register your application's service providers
    app.registerServiceProvider<AppServiceProvider>();
    app.registerServiceProvider<AuthServiceProvider>();
    app.registerServiceProvider<RouteServiceProvider>();
}
```

## Contextual Binding

Sometimes, the same interface may need different implementations based on context. Boson's container supports contextual binding:

```cpp
// Register different repository implementations for different controllers
container.when<UserController>()->needs<IUserRepository>()->give<MySqlUserRepository>();
container.when<AdminController>()->needs<IUserRepository>()->give<CachedUserRepository>();

// Register with factory functions for more control
container.when<ReportController>()->needs<IReportGenerator>()->give([](const boson::Container& container) {
    auto format = container.make<Request>()->param("format", "pdf");
    
    if (format == "pdf") {
        return std::make_shared<PdfReportGenerator>();
    } else if (format == "csv") {
        return std::make_shared<CsvReportGenerator>();
    } else {
        return std::make_shared<HtmlReportGenerator>();
    }
});
```

## Method Injection

Boson supports method injection in addition to constructor injection:

```cpp
class ReportService {
public:
    // Constructor injection
    ReportService(std::shared_ptr<ILogger> logger)
        : logger_(std::move(logger)) {}
    
    // Method injection
    void generateReport(std::shared_ptr<IReportGenerator> generator, const ReportData& data) {
        logger_->info("Generating report");
        auto report = generator->generate(data);
        logger_->info("Report generated");
        return report;
    }

private:
    std::shared_ptr<ILogger> logger_;
};

// In your controller
void ReportController::download(const boson::Request& request) {
    // The container will inject the appropriate IReportGenerator based on context
    auto reportData = buildReportData(request);
    auto report = reportService_->generateReport(container_->make<IReportGenerator>(), reportData);
    
    return boson::Response::download(report.path())
        .filename("report." + report.extension());
}
```

## Property Injection

For situations where constructor injection isn't practical, Boson supports property injection:

```cpp
class ComplexService {
public:
    ComplexService() = default;
    
    // Property setters for injection
    void setLogger(std::shared_ptr<ILogger> logger) {
        logger_ = std::move(logger);
    }
    
    void setRepository(std::shared_ptr<IDataRepository> repository) {
        repository_ = std::move(repository);
    }
    
    void setMailer(std::shared_ptr<IMailer> mailer) {
        mailer_ = std::move(mailer);
    }
    
    // Service methods...

private:
    std::shared_ptr<ILogger> logger_;
    std::shared_ptr<IDataRepository> repository_;
    std::shared_ptr<IMailer> mailer_;
};

// Register with property injection
container.bind<ComplexService>("complex.service", [](const boson::Container& container) {
    auto service = std::make_shared<ComplexService>();
    
    // Inject dependencies via setters
    service->setLogger(container.make<ILogger>());
    service->setRepository(container.make<IDataRepository>());
    service->setMailer(container.make<IMailer>());
    
    return service;
});
```

## Tagged Services

Register services with tags to retrieve them as collections:

```cpp
// Register services with tags
container.bind<IEventListener, UserCreatedListener>()->tag("event.user.created");
container.bind<IEventListener, UserUpdatedListener>()->tag("event.user.updated");
container.bind<IEventListener, UserDeletedListener>()->tag("event.user.deleted");

container.bind<IReport, SalesReport>()->tag("reports");
container.bind<IReport, UserActivityReport>()->tag("reports");
container.bind<IReport, InventoryReport>()->tag("reports");

// Resolve all services with a specific tag
auto userListeners = container.tagged<IEventListener>("event.user.*");  // Uses pattern matching
auto allReports = container.tagged<IReport>("reports");

// Use the resolved services
for (const auto& listener : userListeners) {
    event_bus.registerListener(listener);
}

for (const auto& report : allReports) {
    reportManager.addReport(report);
}
```

## Scoped Services

Register services with specific scopes for controlled lifecycle management:

```cpp
// Register a service that is scoped to an HTTP request
container.scoped<RequestContext>("context.request", [](const boson::Container& container) {
    auto request = container.make<Request>();
    return std::make_shared<RequestContext>(request);
});

// Register a service that is scoped to the current user session
container.scoped<UserSession>("context.session", boson::Scope::Session, [](const boson::Container& container) {
    auto session = container.make<Session>();
    auto userId = session.get<int>("user_id", 0);
    
    if (userId > 0) {
        auto userRepo = container.make<IUserRepository>();
        auto user = userRepo->findById(userId);
        return std::make_shared<UserSession>(user);
    }
    
    return std::make_shared<GuestSession>();
});
```

## Container Events

Listen to container events for advanced integration scenarios:

```cpp
// Listen for service resolution events
container.onResolved<IUserRepository>([](std::shared_ptr<IUserRepository> repo, const boson::Container& container) {
    // Do something when an IUserRepository is resolved
    auto logger = container.make<ILogger>();
    logger->debug("UserRepository was resolved");
});

// Listen for service registration events
container.onRegistered<IPaymentProcessor>([](const std::string& id, const boson::Container& container) {
    // Do something when an IPaymentProcessor is registered
    auto logger = container.make<ILogger>();
    logger->info("PaymentProcessor was registered with ID: {}", id);
});
```

## Lazy Loading

Create lazy-loading proxies for expensive services:

```cpp
// Register a lazy-loaded service
container.lazy<ExpensiveService>("service.expensive", []() {
    // This factory is only called when methods on the service are actually used
    return std::make_shared<ExpensiveService>();
});

// Get a proxy that loads the real service only when needed
auto service = container.make<ExpensiveService>("service.expensive");

// The real service is only instantiated when you call a method
service->doSomething();  // Triggers lazy loading
```

## Service Aliasing

Create aliases for services to improve readability and flexibility:

```cpp
// Register a service
container.singleton<MySqlDatabase>("db.mysql");

// Create aliases
container.alias("db.mysql", "database");
container.alias("db.mysql", "db.default");

// Resolve using any of the names
auto db1 = container.make<Database>("db.mysql");
auto db2 = container.make<Database>("database");     // Same instance
auto db3 = container.make<Database>("db.default");   // Same instance
```

## Extended Binding Syntax

Boson provides a fluent API for more complex binding scenarios:

```cpp
// Using the fluent API for configuration
container.bind<IMailer>()
    ->to<SmtpMailer>()
    ->withConstructorArguments("smtp.example.com", 587)
    ->asSingleton()
    ->whenNeeded<NotificationService>()
    ->tag("mailers");

container.bind<ILogger>()
    ->using([](const boson::Container& container) {
        auto config = container.make<ConfigProvider>();
        auto logLevel = config->get<std::string>("logging.level", "info");
        auto logPath = config->get<std::string>("logging.path", "logs/app.log");
        
        return std::make_shared<FileLogger>(logPath, logLevel);
    })
    ->asSingleton();
```

## Container Configuration

Configure the container's behavior to match your needs:

```cpp
// Configure the container
container.configure([](boson::ContainerConfig& config) {
    // Set default scope for services that don't specify one
    config.setDefaultScope(boson::Scope::Singleton);
    
    // Enable or disable autowiring
    config.setAutowiring(true);
    
    // Configure circular dependency detection
    config.setCircularDependencyDetection(true);
    
    // Set resolution timeout for detecting potential deadlocks
    config.setResolutionTimeout(std::chrono::seconds(5));
    
    // Configure proxy generation
    config.enableProxyGeneration(true);
});
```

## Automatic Injection in Controllers

Boson's controller resolution system automatically injects dependencies:

```cpp
class UserController : public boson::Controller {
public:
    // Dependencies are automatically injected
    UserController(
        std::shared_ptr<IUserRepository> userRepo,
        std::shared_ptr<IAuthService> authService,
        std::shared_ptr<ILogger> logger
    ) : userRepo_(std::move(userRepo)),
        authService_(std::move(authService)),
        logger_(std::move(logger)) {}
    
    boson::Response index(const boson::Request& request) {
        logger_->info("Listing users");
        auto users = userRepo_->all();
        return boson::Response::view("users.index", {{"users", users}});
    }
    
    boson::Response show(const boson::Request& request) {
        auto id = request.param<int>("id");
        auto user = userRepo_->findById(id);
        
        if (!user) {
            return boson::Response::notFound();
        }
        
        return boson::Response::view("users.show", {{"user", *user}});
    }

private:
    std::shared_ptr<IUserRepository> userRepo_;
    std::shared_ptr<IAuthService> authService_;
    std::shared_ptr<ILogger> logger_;
};

// Register routes with automatic controller resolution
app.get("/users", &UserController::index);
app.get("/users/{id}", &UserController::show);
```

## Interface Segregation

Apply the Interface Segregation Principle with specialized interfaces:

```cpp
// Base repository interface
class IRepository {
public:
    virtual ~IRepository() = default;
    virtual void save(const Entity& entity) = 0;
    virtual void remove(const Entity& entity) = 0;
};

// Reader interface
class IRepositoryReader {
public:
    virtual ~IRepositoryReader() = default;
    virtual std::optional<Entity> findById(int id) = 0;
    virtual std::vector<Entity> all() = 0;
};

// Writer interface
class IRepositoryWriter {
public:
    virtual ~IRepositoryWriter() = default;
    virtual void save(const Entity& entity) = 0;
    virtual void remove(const Entity& entity) = 0;
};

// Complete repository implementing all interfaces
class Repository : public IRepository, public IRepositoryReader, public IRepositoryWriter {
    // Implementation...
};

// Register with multiple interfaces
container.bind<Repository>("repository");
container.alias<Repository, IRepository>("repository");
container.alias<Repository, IRepositoryReader>("repository");
container.alias<Repository, IRepositoryWriter>("repository");

// Services can depend on only what they need
class QueryService {
public:
    QueryService(std::shared_ptr<IRepositoryReader> repo)
        : repo_(std::move(repo)) {}
private:
    std::shared_ptr<IRepositoryReader> repo_;  // Only needs read access
};

class CommandService {
public:
    CommandService(std::shared_ptr<IRepositoryWriter> repo)
        : repo_(std::move(repo)) {}
private:
    std::shared_ptr<IRepositoryWriter> repo_;  // Only needs write access
};
```

## Testing with the Container

The dependency injection container simplifies testing by allowing easy replacement of dependencies:

```cpp
#include <gtest/gtest.h>
#include <boson/testing/container.hpp>

class UserServiceTest : public ::testing::Test {
protected:
    void SetUp() override {
        // Create a test container
        container_ = std::make_shared<boson::Container>();
        
        // Register a mock repository
        mockRepo_ = std::make_shared<MockUserRepository>();
        container_->instance<IUserRepository>("user.repository", mockRepo_);
        
        // Create the service under test with the mock dependency
        service_ = container_->make<UserService>();
    }
    
    std::shared_ptr<boson::Container> container_;
    std::shared_ptr<MockUserRepository> mockRepo_;
    std::shared_ptr<UserService> service_;
};

TEST_F(UserServiceTest, FindActiveUsers) {
    // Configure mock expectations
    EXPECT_CALL(*mockRepo_, findByStatus("active"))
        .WillOnce(Return(std::vector<User>{
            User{1, "Alice", "active"},
            User{2, "Bob", "active"}
        }));
    
    // Call the service method
    auto activeUsers = service_->findActiveUsers();
    
    // Verify the result
    ASSERT_EQ(2, activeUsers.size());
    EXPECT_EQ("Alice", activeUsers[0].name());
    EXPECT_EQ("Bob", activeUsers[1].name());
}
```

## Bootstrapping the Container

Here's a complete example of bootstrapping your application's container:

```cpp
#include <boson/boson.hpp>
#include "app/providers/app_service_provider.hpp"
#include "app/providers/auth_service_provider.hpp"
#include "app/providers/route_service_provider.hpp"

int main() {
    // Create the application
    boson::Application app;
    
    // Configure the container
    auto& container = app.container();
    container.configure([](boson::ContainerConfig& config) {
        config.setDefaultScope(boson::Scope::Singleton);
        config.setAutowiring(true);
    });
    
    // Register core service providers
    app.registerServiceProvider<boson::LogServiceProvider>();
    app.registerServiceProvider<boson::DatabaseServiceProvider>();
    app.registerServiceProvider<boson::CacheServiceProvider>();
    app.registerServiceProvider<boson::EventServiceProvider>();
    
    // Register application service providers
    app.registerServiceProvider<AppServiceProvider>();
    app.registerServiceProvider<AuthServiceProvider>();
    app.registerServiceProvider<RouteServiceProvider>();
    
    // Boot the application (calls boot() on all service providers)
    app.boot();
    
    // Start the server
    boson::Server server(app);
    server.listen(app.config()->get<int>("server.port", 8080));
    
    return 0;
}
```

## Best Practices

1. **Depend on abstractions**: Use interfaces instead of concrete implementations in your constructor parameters.

2. **Keep services focused**: Services should have a single responsibility.

3. **Use constructor injection**: Constructor injection makes dependencies explicit and ensures they're available when the service is created.

4. **Register services in service providers**: Organize related services in dedicated service providers.

5. **Use appropriate scopes**: Choose the right scope (singleton, request, etc.) for each service.

6. **Avoid the service locator pattern**: Directly injecting dependencies is better than pulling them from the container within a class.

7. **Consider lazy loading**: Use lazy loading for expensive services that aren't always needed.

8. **Write testable services**: Design services to be easily testable with mock dependencies.

9. **Use contextual binding**: Register different implementations for different contexts.

10. **Apply interface segregation**: Split large interfaces into smaller, focused ones.

## Next Steps

Now that you understand dependency injection in Boson, explore these related topics:

1. Learn about [Service Providers](../advanced/service-providers.md) for organizing your application's bootstrap process
2. Explore [Testing](../advanced/testing.md) to see how dependency injection simplifies unit testing
3. Study [Configuration](configuration.md) to integrate configuration values with your dependency injection container