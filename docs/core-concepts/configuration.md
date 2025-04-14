---
sidebar_position: 7
title: Configuration
---

# Configuration in Boson

Configuring your Boson application is essential for customizing its behavior to match your specific requirements. This guide covers the configuration system and how to use it effectively.

## Configuration Architecture

Boson uses a hierarchical configuration system that loads settings from multiple sources in a defined order:

1. **Default configuration**: Built-in defaults provided by the framework
2. **Configuration files**: Settings specified in configuration files
3. **Environment variables**: Settings specified in environment variables
4. **Runtime configuration**: Settings modified programmatically at runtime

This layered approach allows for flexibility while maintaining sensible defaults.

## Configuration Files

Boson uses a directory-based configuration system. Configuration files are typically stored in the `config/` directory of your application.

### Basic Structure

A typical configuration directory structure:

```
config/
├── app.json           # Application configuration
├── database.json      # Database connections
├── cache.json         # Cache settings
├── logging.json       # Logging configuration
├── server.json        # HTTP server settings
├── auth.json          # Authentication settings
├── mail.json          # Mail configuration
├── services.json      # External services
└── environments/      # Environment-specific overrides
    ├── development/   # Development environment
    │   ├── app.json
    │   └── database.json
    ├── testing/       # Testing environment
    │   ├── app.json
    │   └── database.json
    └── production/    # Production environment
        ├── app.json
        └── database.json
```

### Configuration File Format

Boson supports multiple configuration file formats:

#### JSON (default)

```json
{
  "app": {
    "name": "My Application",
    "environment": "development",
    "debug": true,
    "url": "http://localhost:8080",
    "timezone": "UTC"
  }
}
```

#### YAML

```yaml
app:
  name: My Application
  environment: development
  debug: true
  url: http://localhost:8080
  timezone: UTC
```

#### TOML

```toml
[app]
name = "My Application"
environment = "development"
debug = true
url = "http://localhost:8080"
timezone = "UTC"
```

#### C++ Source

For advanced use cases, you can define configuration directly in C++:

```cpp
// config.cpp
#include <boson/config/provider.hpp>

void registerConfiguration(boson::ConfigProvider& config) {
    config.set("app.name", "My Application");
    config.set("app.environment", "development");
    config.set("app.debug", true);
    config.set("app.url", "http://localhost:8080");
    config.set("app.timezone", "UTC");
}
```

## Loading Configuration

### Automatic Loading

Boson automatically loads configuration files when you create an application:

```cpp
#include <boson/boson.hpp>

int main() {
    // Creates application and loads configuration from config/
    boson::Application app;
    
    // Start the server
    boson::Server server(app);
    server.listen(app.config()->get<int>("server.port", 8080));
    return 0;
}
```

### Manual Loading

You can manually specify configuration sources:

```cpp
#include <boson/boson.hpp>

int main() {
    // Create configuration provider
    auto config = std::make_shared<boson::ConfigProvider>();
    
    // Load from specific directory
    config->loadFromDirectory("/path/to/config");
    
    // Load specific file
    config->loadFromFile("/path/to/custom-config.json");
    
    // Create application with custom configuration
    boson::Application app(config);
    
    // Start the server
    boson::Server server(app);
    server.listen(8080);
    return 0;
}
```

## Accessing Configuration Values

### In Application Code

```cpp
#include <boson/boson.hpp>

class UserController : public boson::Controller {
public:
    UserController(boson::Application& app)
        : Controller(app) {
        // Access configuration in controller constructor
        appName_ = app.config()->get<std::string>("app.name");
        maxUploadSize_ = app.config()->get<int>("uploads.maxSize", 10485760); // Default: 10MB
    }
    
    // Route handler that uses configuration
    boson::Response index(const boson::Request& request) {
        return boson::Response::json({
            {"app_name", appName_},
            {"max_upload", maxUploadSize_}
        });
    }

private:
    std::string appName_;
    int maxUploadSize_;
};
```

### Using the Config Helper

```cpp
#include <boson/config.hpp>

void someFunction() {
    // Get a string value
    std::string appName = boson::config("app.name");
    
    // Get a value with a default
    bool debug = boson::config("app.debug", false);
    
    // Get a typed value
    int timeout = boson::config<int>("services.api.timeout", 30);
    
    // Check if a configuration value exists
    if (boson::configExists("mail.driver")) {
        // Use mail configuration
    }
}
```

### In Middleware and Services

```cpp
class CacheMiddleware : public boson::Middleware {
public:
    CacheMiddleware(std::shared_ptr<boson::ConfigProvider> config)
        : config_(std::move(config)) {
        // Initialize middleware with configuration
        enabled_ = config_->get<bool>("cache.enabled", true);
        ttl_ = config_->get<int>("cache.ttl", 3600);
    }
    
    boson::Response process(const boson::Request& request, MiddlewareNext next) override {
        if (!enabled_) {
            return next(request);
        }
        
        // Caching logic using configuration values
        // ...
    }

private:
    std::shared_ptr<boson::ConfigProvider> config_;
    bool enabled_;
    int ttl_;
};
```

## Configuration Structure

### Standard Configuration Files

#### app.json

Application-wide configuration:

```json
{
  "name": "My Boson Application",
  "environment": "development",
  "debug": true,
  "url": "http://localhost:8080",
  "timezone": "UTC",
  "locale": "en-US",
  "key": "base64:LJ9JOIobiPmwXQxA4CvRCGVnIkN0sk7nTIP6XUkRn6E="
}
```

#### server.json

Server-specific configuration:

```json
{
  "host": "0.0.0.0",
  "port": 8080,
  "workers": 4,
  "maxConnections": 1000,
  "keepAlive": true,
  "keepAliveTimeout": 5,
  "readTimeout": 60,
  "writeTimeout": 60,
  "requestBodyLimit": 10485760,
  "tls": {
    "enabled": false,
    "certificate": "/path/to/cert.pem",
    "key": "/path/to/key.pem"
  },
  "cors": {
    "enabled": true,
    "origins": ["*"],
    "methods": ["GET", "POST", "PUT", "DELETE"],
    "headers": ["Content-Type", "Accept", "Authorization"],
    "credentials": true,
    "maxAge": 3600
  }
}
```

#### database.json

Database connection configuration:

```json
{
  "default": "sqlite",
  "connections": {
    "sqlite": {
      "driver": "sqlite",
      "database": "database.sqlite",
      "prefix": ""
    },
    "mysql": {
      "driver": "mysql",
      "host": "localhost",
      "port": 3306,
      "database": "boson",
      "username": "root",
      "password": "",
      "charset": "utf8mb4",
      "collation": "utf8mb4_unicode_ci",
      "prefix": ""
    },
    "postgres": {
      "driver": "postgres",
      "host": "localhost",
      "port": 5432,
      "database": "boson",
      "username": "postgres",
      "password": "",
      "charset": "utf8",
      "prefix": ""
    }
  },
  "migrations": {
    "table": "migrations",
    "directory": "database/migrations"
  }
}
```

#### logging.json

Logging configuration:

```json
{
  "default": "stack",
  "channels": {
    "stack": {
      "driver": "stack",
      "channels": ["console", "file"]
    },
    "console": {
      "driver": "console",
      "level": "debug",
      "format": "{timestamp} [{level}] {message}"
    },
    "file": {
      "driver": "file",
      "path": "logs/boson.log",
      "level": "info",
      "days": 14,
      "format": "{timestamp} [{level}] {message}"
    },
    "syslog": {
      "driver": "syslog",
      "level": "error",
      "facility": "user"
    }
  }
}
```

#### cache.json

Cache configuration:

```json
{
  "default": "memory",
  "stores": {
    "memory": {
      "driver": "memory",
      "max_items": 10000
    },
    "file": {
      "driver": "file",
      "path": "cache/data",
      "ttl": 3600
    },
    "redis": {
      "driver": "redis",
      "host": "localhost",
      "port": 6379,
      "password": null,
      "database": 0,
      "prefix": "boson:"
    }
  },
  "ttl": 3600,
  "prefix": "boson_cache"
}
```

## Environment Variables

Boson can load configuration from environment variables, which is particularly useful for production deployments and containerized environments.

### Environment Variable Format

Environment variables use a specific format to override configuration values:

```
BOSON_{CATEGORY}_{KEY}={VALUE}
```

Examples:

```bash
# Override app.name
BOSON_APP_NAME="Production Application"

# Override database connection
BOSON_DATABASE_CONNECTIONS_MYSQL_HOST="db.example.com"
BOSON_DATABASE_CONNECTIONS_MYSQL_PASSWORD="secure-password"

# Override server port
BOSON_SERVER_PORT=80
```

### Loading Environment Variables

Boson automatically looks for a `.env` file in your application's root directory:

```
# .env file
APP_NAME=My Boson App
APP_ENV=development
APP_DEBUG=true
APP_URL=http://localhost:8080

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=boson
DB_USERNAME=root
DB_PASSWORD=

CACHE_DRIVER=file
SESSION_DRIVER=file
```

You can also manually load environment variables:

```cpp
#include <boson/env.hpp>

int main() {
    // Load from specific .env file
    boson::loadEnvFile("/path/to/.env.production");
    
    boson::Application app;
    // ...
    return 0;
}
```

### Environment-Specific Configuration

Boson supports different configurations based on the current environment:

```cpp
// In main.cpp
#include <boson/boson.hpp>

int main() {
    // Set environment (defaults to value from BOSON_APP_ENV or "development")
    boson::setEnvironment("production");
    
    boson::Application app;
    // App will load config/environments/production/* files
    
    boson::Server server(app);
    server.listen(8080);
    return 0;
}
```

## Dynamic Configuration

### Runtime Configuration Changes

You can modify configuration at runtime:

```cpp
void adjustConfiguration(boson::Application& app) {
    // Change configuration value
    app.config()->set("logging.level", "debug");
    
    // Set a structured value
    app.config()->set("services.analytics", {
        {"enabled", true},
        {"trackingId", "UA-123456-7"}
    });
    
    // Remove a configuration value
    app.config()->remove("feature.experimental");
}
```

### Configuration Callbacks

Register callbacks for when configuration changes:

```cpp
app.config()->onChange("cache.ttl", [&cacheService](const auto& newValue) {
    int ttl = std::get<int>(newValue);
    cacheService->setDefaultTtl(ttl);
});
```

## Configuration Providers

Boson supports custom configuration providers for integrating with external configuration sources:

```cpp
class RemoteConfigProvider : public boson::ConfigProvider {
public:
    RemoteConfigProvider(std::string serviceUrl, std::string apiKey)
        : serviceUrl_(std::move(serviceUrl)), apiKey_(std::move(apiKey)) {}
        
    void load() override {
        // Fetch configuration from remote API
        auto client = boson::HttpClient::create();
        auto response = client->get(serviceUrl_)
            .header("Authorization", "Bearer " + apiKey_)
            .send();
            
        if (response.isSuccessful()) {
            // Parse JSON response
            auto json = response.jsonBody();
            
            // Add all items to configuration store
            for (const auto& [key, value] : json.items()) {
                set(key, value);
            }
        }
    }
    
private:
    std::string serviceUrl_;
    std::string apiKey_;
};

// Using the custom provider
auto remoteConfig = std::make_shared<RemoteConfigProvider>(
    "https://config.example.com/api/config",
    "api-key-123"
);
remoteConfig->load();

boson::Application app(remoteConfig);
```

### Multiple Providers

Use multiple configuration providers with prioritization:

```cpp
// Create a composite configuration provider
auto compositeConfig = std::make_shared<boson::CompositeConfigProvider>();

// Add default configuration with lowest priority
auto defaultConfig = std::make_shared<boson::FileConfigProvider>("config/defaults");
compositeConfig->addProvider(defaultConfig, 0);

// Add environment-specific configuration with medium priority
auto envConfig = std::make_shared<boson::FileConfigProvider>("config/environments/production");
compositeConfig->addProvider(envConfig, 1);

// Add remote configuration with highest priority
auto remoteConfig = std::make_shared<RemoteConfigProvider>(
    "https://config.example.com/api/config",
    "api-key-123"
);
compositeConfig->addProvider(remoteConfig, 2);

// Create application with composite configuration
boson::Application app(compositeConfig);
```

## Configuration Security

### Sensitive Configuration

Sensitive values should not be committed to version control. Use environment variables or a separate non-versioned configuration file for sensitive data:

```json
// config/app.json - Safe to commit
{
  "name": "My Application",
  "environment": "${APP_ENV}"
}

// config/database.json - Safe to commit with placeholders
{
  "connections": {
    "mysql": {
      "host": "${DB_HOST}",
      "database": "${DB_DATABASE}",
      "username": "${DB_USERNAME}",
      "password": "${DB_PASSWORD}"
    }
  }
}
```

### Encrypted Configuration

For highly sensitive data, use Boson's encryption system:

```cpp
// Encrypt a configuration value
std::string encryptedValue = boson::encrypt("sensitive-data", app.config()->get<std::string>("app.key"));

// Store the encrypted value
app.config()->set("services.api.secret", "encrypted:" + encryptedValue);

// Later, use the value with automatic decryption
std::string apiSecret = app.config()->getDecrypted("services.api.secret");
```

## Configuration Validation

Validate your configuration to ensure all required values are present and valid:

```cpp
// Define validation rules
auto validation = boson::ConfigValidator::validate(app.config(), {
    {"app.name", "required|string"},
    {"app.environment", "required|in:development,testing,production"},
    {"server.port", "required|integer|min:1|max:65535"},
    {"database.connections.*.host", "required|string"},
    {"database.connections.*.port", "required|integer"}
});

// Check for validation errors
if (validation.fails()) {
    auto errors = validation.errors();
    for (const auto& [key, messages] : errors) {
        std::cerr << "Configuration error in " << key << ": " 
                  << messages.front() << std::endl;
    }
    return 1;
}
```

## Configuration Caching

In production environments, cache your configuration to avoid parsing configuration files on each request:

```cpp
#include <boson/config/cache.hpp>

int main(int argc, char** argv) {
    // Check for config:cache command
    if (argc > 1 && std::string(argv[1]) == "config:cache") {
        boson::Application app;
        boson::cacheConfig(app.config(), "bootstrap/cache/config.cache");
        std::cout << "Configuration cached successfully." << std::endl;
        return 0;
    }
    
    // Check for config:clear command
    if (argc > 1 && std::string(argv[1]) == "config:clear") {
        boson::clearConfigCache("bootstrap/cache/config.cache");
        std::cout << "Configuration cache cleared." << std::endl;
        return 0;
    }
    
    // Normal application startup
    auto config = std::make_shared<boson::ConfigProvider>();
    
    // Try to load from cache first
    if (!config->loadFromCache("bootstrap/cache/config.cache")) {
        // If cache loading failed, load from files
        config->loadFromDirectory("config");
    }
    
    boson::Application app(config);
    // ...
}
```

## Best Practices

1. **Use Environment Variables for Sensitive Data**: Don't store secrets in configuration files.

2. **Default Values**: Always provide sensible defaults for optional configuration values.

3. **Environment-Specific Configuration**: Use separate configuration files for different environments.

4. **Configuration Validation**: Validate your configuration at application startup.

5. **Cache in Production**: Use configuration caching in production for better performance.

6. **Documentation**: Document all configuration options and their possible values.

7. **Namespacing**: Use namespaced keys to avoid conflicts (e.g., `service.aws.key` instead of `key`).

8. **Typed Access**: Always use typed access methods like `get<int>()` for type safety.

9. **Configuration Change Detection**: Implement monitoring for critical configuration changes.

10. **Version Control**: Keep configuration templates in version control, but not actual configurations with sensitive data.

## Next Steps

Now that you understand configuration in Boson, explore these related topics:

1. Learn about [Environment Management](../advanced/environments.md) for managing different deployment environments
2. Explore [Dependency Injection](dependency-injection.md) to see how configuration values can be injected into your services
3. Study [Security](../security/overview.md) for more details on protecting sensitive configuration