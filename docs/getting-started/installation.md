---
sidebar_position: 1
title: Installation
---

# Installing Boson Framework

This guide will help you install the Boson framework quickly and easily. Boson supports major operating systems and can be installed through package managers or built from source.

## Prerequisites

Before installing Boson, make sure you have:

- **C++ Compiler** with C++17 support (GCC 7.3+, Clang 6.0+, or MSVC 2017+)
- **CMake** 3.14 or higher
- **OpenSSL** 1.1.1 or higher (for HTTPS support)

## Installation Options

### Option 1: Using Package Managers (Recommended)

#### macOS

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

### Option 2: Building from Source

For the latest features or if your platform doesn't have a package available:

#### Step 1: Clone the Repository

```bash
git clone https://github.com/boson/boson-framework.git
cd boson-framework
```

#### Step 2: Build with CMake

```bash
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
cmake --build .
sudo cmake --install .
```

## Verifying Your Installation

To confirm that Boson was installed correctly:

```cpp
#include <boson/boson.hpp>
#include <iostream>

int main() {
    std::cout << "Boson installed successfully!" << std::endl;
    boson::Server app;
    // If this compiles, Boson is installed correctly
    return 0;
}
```

Compile with:

```bash
g++ -std=c++17 test.cpp -lboson -o test
```

## Troubleshooting

### Common Issues

#### Missing Libraries
If you encounter errors about missing libraries during compilation:

```bash
# Ubuntu/Debian
sudo apt install libssl-dev

# macOS
brew install openssl
```

#### CMake Can't Find Boson
Add the installation directory to your CMAKE_PREFIX_PATH:

```bash
cmake .. -DCMAKE_PREFIX_PATH=/path/to/boson/installation
```

## Next Steps

Now that you have Boson installed, continue to the [Quick Start Guide](quickstart) to create your first Boson application.