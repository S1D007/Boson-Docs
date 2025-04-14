---
sidebar_position: 3
title: Quickstart
---

# Boson Quickstart Guide

This quickstart guide will help you create your first Boson application in minutes. We'll build a simple REST API that responds with JSON data.

## Step 1: Create a New Project

First, let's create a new Boson project using the Boson CLI tool:

```bash
boson create api-example
cd api-example
```

If you don't have the Boson CLI installed, you can manually set up a project:

```bash
mkdir api-example
cd api-example
```

## Step 2: Set Up the Project Structure

Create these files and directories to structure your application:

```
api-example/
├── CMakeLists.txt
├── src/
│   ├── main.cpp
│   ├── controllers/
│   │   └── user_controller.hpp
│   └── models/
│       └── user.hpp
└── config/
    └── app.json
```

## Step 3: Configure CMakeLists.txt

Create a `CMakeLists.txt` file in your project root:

```cmake
cmake_minimum_required(VERSION 3.14)
project(api-example VERSION 0.1.0)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(Boson REQUIRED)

add_executable(${PROJECT_NAME} src/main.cpp)
target_link_libraries(${PROJECT_NAME} PRIVATE Boson::Boson)

# Copy configuration files to build directory
file(COPY ${CMAKE_SOURCE_DIR}/config DESTINATION ${CMAKE_BINARY_DIR})
```

## Step 4: Define Your Model

Create a simple User model in `src/models/user.hpp`:

```cpp
#pragma once

#include <boson/model.hpp>
#include <string>

class User : public boson::Model {
public:
    User() = default;
    User(int id, std::string name, std::string email)
        : id_(id), name_(std::move(name)), email_(std::move(email)) {}

    int id() const { return id_; }
    const std::string& name() const { return name_; }
    const std::string& email() const { return email_; }

    // Serialization for JSON responses
    boson::json::Object toJson() const override {
        return {
            {"id", id_},
            {"name", name_},
            {"email", email_}
        };
    }

private:
    int id_ = 0;
    std::string name_;
    std::string email_;
};
```

## Step 5: Create a Controller

Create a controller to handle user-related requests in `src/controllers/user_controller.hpp`:

```cpp
#pragma once

#include <boson/controller.hpp>
#include <boson/response.hpp>
#include <vector>
#include "../models/user.hpp"

class UserController : public boson::Controller {
public:
    UserController() {
        // Initialize with some sample data
        users_.emplace_back(1, "Alice Smith", "alice@example.com");
        users_.emplace_back(2, "Bob Johnson", "bob@example.com");
        users_.emplace_back(3, "Carol Williams", "carol@example.com");
    }

    void registerRoutes() override {
        // Define routes for this controller
        GET("/users", &UserController::getAllUsers);
        GET("/users/{id}", &UserController::getUserById);
    }

    // Handler for getting all users
    boson::Response getAllUsers(const boson::Request& request) {
        return boson::Response::json(users_);
    }

    // Handler for getting a user by ID
    boson::Response getUserById(const boson::Request& request) {
        auto id = request.param<int>("id");
        
        for (const auto& user : users_) {
            if (user.id() == id) {
                return boson::Response::json(user);
            }
        }

        return boson::Response::notFound()
            .json({{"error", "User not found"}});
    }

private:
    std::vector<User> users_;
};
```

## Step 6: Set Up the Application

Create the main application file at `src/main.cpp`:

```cpp
#include <boson/boson.hpp>
#include <boson/server.hpp>
#include <iostream>
#include "controllers/user_controller.hpp"

int main() {
    try {
        // Create the application
        boson::Application app;
        
        // Load configuration
        app.loadConfig("config/app.json");
        
        // Register controllers
        app.registerController<UserController>();
        
        // Create and start the server
        boson::Server server(app);
        std::cout << "Server running at http://127.0.0.1:8080" << std::endl;
        server.listen(8080);
        
        return 0;
    } catch (const std::exception& e) {
        std::cerr << "Error: " << e.what() << std::endl;
        return 1;
    }
}
```

## Step 7: Create a Configuration File

Create a basic configuration in `config/app.json`:

```json
{
  "app": {
    "name": "Boson API Example",
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
  }
}
```

## Step 8: Build and Run

Build your application:

```bash
mkdir build && cd build
cmake ..
cmake --build .
```

Run your application:

```bash
./api-example
```

## Step 9: Test Your API

Open a browser or use curl to test your API endpoints:

```bash
# Get all users
curl http://127.0.0.1:8080/users

# Get user by ID
curl http://127.0.0.1:8080/users/1
```

You should see JSON responses with user data!

## Next Steps

Now that you've created your first Boson application, you can:

1. Add more routes and controllers
2. Connect to a database using Boson's database adapters
3. Implement authentication middleware
4. Explore more advanced features in the [Core Concepts](../core-concepts/server.md) section

For a more detailed walkthrough, continue to the [Hello World](hello-world.md) tutorial, which explores each component in greater depth.

## Troubleshooting

### Common Issues

- **Compilation errors**: Ensure your compiler fully supports C++17
- **Runtime errors**: Check your configuration file syntax
- **Missing headers**: Verify Boson is correctly installed and CMake can find it

If you encounter problems, refer to the [Troubleshooting](../advanced/troubleshooting.md) guide or open an issue on our GitHub repository.