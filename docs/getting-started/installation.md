---
sidebar_position: 1
title: Installation
---

# Installing Boson Framework

This guide will walk you through installing the Boson framework on your system. Boson supports multiple operating systems and offers several installation methods to suit your needs.

## Prerequisites

Before installing Boson, make sure your system meets the following requirements:

- **C++ Compiler** with C++17 support:
  - GCC 7.3+ 
  - Clang 6.0+
  - MSVC 19.14+ (Visual Studio 2017 15.7+)
- **Build System**:
  - CMake 3.14+
  - Ninja (recommended) or Make
- **Development Libraries**:
  - OpenSSL 1.1.1+
  - zlib 1.2.11+

## Installation Methods

### 1. Using Package Manager (Recommended)

#### macOS (Homebrew)

```bash
brew install boson-framework
```

#### Ubuntu/Debian

```bash
sudo apt-add-repository ppa:boson/stable
sudo apt update
sudo apt install libboson-dev
```

#### Fedora/RHEL/CentOS

```bash
sudo dnf install boson-devel
```

#### Windows (vcpkg)

```bash
vcpkg install boson
```

### 2. Building from Source

#### Step 1: Clone the Repository

```bash
git clone https://github.com/boson/boson-framework.git
cd boson-framework
```

#### Step 2: Create Build Directory

```bash
mkdir build && cd build
```

#### Step 3: Configure with CMake

```bash
cmake .. -DCMAKE_BUILD_TYPE=Release -DBOSON_BUILD_EXAMPLES=OFF
```

#### Step 4: Build and Install

```bash
cmake --build . --config Release
cmake --install .
```

### 3. Using Docker

Boson provides official Docker images for development and production:

```bash
docker pull boson/boson:latest
```

To start a container with Boson:

```bash
docker run -p 8080:8080 boson/boson:latest
```

## Platform-Specific Instructions

### macOS

For macOS users, we recommend installing additional development tools:

```bash
xcode-select --install
brew install cmake ninja openssl
```

### Linux

On Linux systems, install the required development libraries:

#### Ubuntu/Debian:
```bash
sudo apt install build-essential cmake ninja-build libssl-dev libz-dev
```

#### Fedora/RHEL/CentOS:
```bash
sudo dnf install gcc-c++ cmake ninja-build openssl-devel zlib-devel
```

### Windows

For Windows development:

1. Install [Visual Studio](https://visualstudio.microsoft.com/) with "Desktop development with C++" workload
2. Install [CMake](https://cmake.org/download/)
3. Use [vcpkg](https://github.com/microsoft/vcpkg) for managing dependencies:
   ```bash
   git clone https://github.com/microsoft/vcpkg
   .\vcpkg\bootstrap-vcpkg.bat
   .\vcpkg\vcpkg integrate install
   .\vcpkg\vcpkg install openssl:x64-windows zlib:x64-windows
   ```

## Verifying Your Installation

To verify that Boson is correctly installed:

```bash
boson --version
```

You should see output similar to:

```
Boson Framework v1.5.2
```

You can also check if Boson is properly installed in your C++ project:

```cpp
#include <boson/boson.hpp>

int main() {
    boson::Application app;
    std::cout << "Boson version: " << BOSON_VERSION << std::endl;
    return 0;
}
```

## Next Steps

Now that you have Boson installed, you can:
- Explore the [Prerequisites](prerequisites.md) for building Boson applications
- Follow the [Quickstart](quickstart.md) guide to create your first Boson application
- Learn about the [Project Structure](project-structure.md) of a typical Boson application

If you encounter any issues during installation, please check the [Troubleshooting](#troubleshooting) section below or visit our [GitHub repository](https://github.com/boson/boson-framework/issues) for support.

## Troubleshooting

### Common Issues

#### Missing Dependencies

If you encounter errors about missing dependencies, ensure that all required libraries are installed:

```bash
# Ubuntu/Debian
sudo apt install libssl-dev libz-dev

# macOS
brew install openssl zlib

# Windows (via vcpkg)
.\vcpkg\vcpkg install openssl zlib
```

#### Compiler Version Too Old

If you get errors about unsupported C++ features, your compiler may be outdated:

```bash
# Check GCC version
g++ --version

# Check Clang version
clang++ --version

# Check MSVC version (Windows)
cl
```

#### CMake Can't Find Libraries

Specify paths to libraries if CMake can't find them:

```bash
cmake .. -DCMAKE_BUILD_TYPE=Release -DOPENSSL_ROOT_DIR=/path/to/openssl
```

#### Build Errors

Clean your build directory and try again:

```bash
rm -rf build/*
mkdir -p build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
cmake --build .
```

For more detailed troubleshooting, visit our [troubleshooting guide](../advanced/troubleshooting.md).