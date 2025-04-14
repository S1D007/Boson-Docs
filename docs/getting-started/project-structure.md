---
sidebar_position: 5
title: Project Structure
---

# Boson Project Structure

This guide outlines recommended project structures for Boson framework applications. Following these conventions will help you organize your code for clarity, maintainability, and scalability.

## Basic Project Structure

A typical Boson project has the following structure:

```
my-boson-app/
├── CMakeLists.txt              # Main CMake configuration file
├── src/                        # Application source code
│   ├── main.cpp                # Application entry point
│   ├── controllers/            # Route controllers
│   ├── middleware/             # Custom middleware
│   ├── models/                 # Data models
│   ├── services/               # Business logic services
│   ├── config/                 # Configuration management code
│   └── utils/                  # Utility functions/classes
├── include/                    # Public header files
│   └── my-boson-app/           # Namespaced headers
├── public/                     # Static assets (images, CSS, JS)
├── templates/                  # Template files
├── config/                     # Configuration files
│   ├── app.json                # Main application config
│   ├── routes.json             # (Optional) Route definitions
│   └── database.json           # Database configuration
├── tests/                      # Test files
│   ├── unit/                   # Unit tests
│   ├── integration/            # Integration tests
│   └── e2e/                    # End-to-end tests
├── docs/                       # Project documentation
├── scripts/                    # Utility scripts
├── .clang-format               # Code formatting rules
├── .gitignore                  # Git ignore rules
└── README.md                   # Project readme
```

## Directory Breakdown

### src/

The `src/` directory contains your application source code, organized as follows:

#### controllers/

Controllers are responsible for handling HTTP requests and generating responses. Each controller typically focuses on a specific resource or feature:

```cpp
// UserController.hpp
#pragma once

#include <boson/controller.hpp>

class UserController : public boson::Controller {
public:
    void registerRoutes() override;
    
    boson::Response listUsers(const boson::Request& request);
    boson::Response getUserById(const boson::Request& request);
    boson::Response createUser(const boson::Request& request);
    boson::Response updateUser(const boson::Request& request);
    boson::Response deleteUser(const boson::Request& request);
};
```

#### middleware/

Middleware handles cross-cutting concerns that should be applied to multiple routes:

```cpp
// AuthMiddleware.hpp
#pragma once

#include <boson/middleware.hpp>

class AuthMiddleware : public boson::Middleware {
public:
    boson::Response process(const boson::Request& request, MiddlewareNext next) override;

private:
    bool validateToken(const std::string& token);
};
```

#### models/

Models represent your data structures and include database interaction logic:

```cpp
// User.hpp
#pragma once

#include <boson/model.hpp>
#include <string>

class User : public boson::Model {
public:
    int id() const;
    const std::string& username() const;
    const std::string& email() const;
    
    void setUsername(std::string username);
    void setEmail(std::string email);
    
    static User findById(int id);
    static std::vector<User> all();
    
    bool save();
    bool remove();
    
    boson::json::Object toJson() const override;

private:
    int id_ = 0;
    std::string username_;
    std::string email_;
    std::string password_hash_;
};
```

#### services/

Services contain business logic that's shared across multiple controllers:

```cpp
// UserService.hpp
#pragma once

#include <string>
#include "../models/User.hpp"

class UserService {
public:
    User authenticate(const std::string& username, const std::string& password);
    std::string generateToken(const User& user);
    bool verifyToken(const std::string& token);
};
```

### config/

The `config/` directory contains JSON configuration files for your application:

```json
// app.json
{
  "app": {
    "name": "My Boson App",
    "environment": "development",
    "debug": true
  },
  "server": {
    "host": "127.0.0.1",
    "port": 8080,
    "workers": 4
  },
  "logging": {
    "level": "debug",
    "file": "logs/app.log"
  },
  "security": {
    "jwt_secret": "your-secret-key",
    "token_expiry": 3600
  }
}
```

### public/

The `public/` directory contains static files that are served directly by Boson:

```
public/
├── css/
│   └── styles.css
├── js/
│   └── app.js
├── images/
│   └── logo.png
└── index.html
```

### templates/

If your application uses HTML templates, store them in the `templates/` directory:

```
templates/
├── layout/
│   └── base.html
├── partials/
│   ├── header.html
│   └── footer.html
└── pages/
    ├── home.html
    ├── about.html
    └── users/
        ├── list.html
        └── profile.html
```

### tests/

Organize your tests by type for easier management:

```
tests/
├── unit/
│   ├── services/
│   │   └── user_service_test.cpp
│   └── models/
│       └── user_test.cpp
├── integration/
│   └── api/
│       └── user_api_test.cpp
└── e2e/
    └── user_flow_test.cpp
```

## Advanced Project Structure

For larger applications, consider a more modular approach:

```
my-boson-app/
├── CMakeLists.txt
├── modules/                    # Feature modules
│   ├── auth/                   # Authentication module
│   │   ├── controllers/
│   │   ├── middleware/
│   │   ├── services/
│   │   └── models/
│   ├── users/                  # User management module
│   │   ├── controllers/
│   │   ├── services/
│   │   └── models/
│   └── products/               # Product management module
│       ├── controllers/
│       ├── services/
│       └── models/
├── core/                       # Core functionality
│   ├── config/
│   ├── database/
│   ├── logging/
│   └── utils/
├── public/
├── templates/
├── config/
└── tests/
```

## Multiple CMake Projects

For very large applications, split functionality into multiple CMake projects:

```
my-boson-app/
├── CMakeLists.txt              # Main CMakeLists.txt
├── app/                        # Main application
│   ├── CMakeLists.txt
│   └── src/
├── libraries/                  # Shared libraries
│   ├── common/                 # Common utilities
│   │   ├── CMakeLists.txt
│   │   └── src/
│   └── database/               # Database library
│       ├── CMakeLists.txt
│       └── src/
└── tools/                      # CLI tools
    ├── CMakeLists.txt
    └── src/
```

## CMakeLists.txt Example

A typical `CMakeLists.txt` for a Boson project might look like:

```cmake
cmake_minimum_required(VERSION 3.14)
project(my-boson-app VERSION 0.1.0)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Find Boson package
find_package(Boson REQUIRED)

# Collect source files
file(GLOB_RECURSE SOURCES "src/*.cpp")
file(GLOB_RECURSE HEADERS "include/*.hpp" "src/*.hpp")

# Create executable
add_executable(${PROJECT_NAME} ${SOURCES})

# Add include directories
target_include_directories(${PROJECT_NAME} PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/src
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)

# Link to Boson libraries
target_link_libraries(${PROJECT_NAME} PRIVATE Boson::Boson)

# Copy configuration files to build directory
file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/config DESTINATION ${CMAKE_BINARY_DIR})
file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/public DESTINATION ${CMAKE_BINARY_DIR})
file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/templates DESTINATION ${CMAKE_BINARY_DIR})

# Install targets
install(TARGETS ${PROJECT_NAME} DESTINATION bin)
install(DIRECTORY config public templates DESTINATION share/${PROJECT_NAME})

# Enable testing
enable_testing()
add_subdirectory(tests)
```

## Best Practices

1. **Separation of Concerns**: Keep controllers thin, with business logic in services
2. **Single Responsibility**: Each class should have a single responsibility
3. **Resource-based Organization**: Group code by resource/feature, not by type
4. **Consistent Naming**: Follow consistent naming conventions
5. **Error Handling**: Implement error handling at appropriate levels
6. **Configuration Management**: Use environment-specific configuration files
7. **Dependency Injection**: Use dependency injection for better testability
8. **Middleware Pipeline**: Use middleware for cross-cutting concerns
9. **Documentation**: Document public APIs and key components
10. **Testing**: Write tests for all critical functionality

## Project Templates

You can use the Boson CLI to generate a project with the recommended structure:

```bash
boson create --template api my-api-project
boson create --template web my-web-project
boson create --template microservice my-microservice
```

## Next Steps

With your project structure in place, you're ready to explore the core concepts of Boson framework:

1. [Server Configuration](../core-concepts/server.md)
2. [Routing System](../core-concepts/routing.md)
3. [Controller Implementation](../core-concepts/controllers.md)