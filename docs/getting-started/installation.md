---
sidebar_position: 1
title: Installation
---

# Installing Boson Framework

---

## Prerequisites

Before installing Boson, make sure you have:

- **C++ Compiler** with C++17 support (GCC 7.3+, Clang 6.0+, or MSVC 2017+)
- **CMake** 3.14 or higher
- **OpenSSL** 1.1.1 or higher (for HTTPS support)


# Installing Boson CLI

The Boson CLI makes it easy to create, build, and run Boson C++ web projects. You can install it on macOS or Linux with a single command:

```bash
curl -fsSL https://raw.githubusercontent.com/S1D007/boson/main/install.sh | bash
```

This script will:
- Download the latest Boson CLI and framework for your platform
- Install it to `~/.local/bin` (or another directory in your PATH)
- Set up auto-update so you always have the latest version

## Verifying Installation

After installation, open a new terminal and run:

```bash
boson --help
```

You should see the Boson CLI help output. You can also check the version:

```bash
boson version
```

## Keeping Boson Up to Date

The installer automatically sets up auto-update for the CLI. Every time you open a new shell, the CLI will check for updates and install the latest version if available.

You can also update manually at any time:

```bash
boson update
```

## Uninstalling

To remove the Boson CLI, simply delete the installed files:

```bash
rm -rf ~/.local/bin/boson ~/.boson
```

---

This guide will help you install the Boson framework quickly and easily. Boson supports major operating systems and can be installed through package managers or built from source.

## Other Installation Options

For someone who wants to build it from scratch

#### Step 1: Clone the Repository

```bash
git clone https://github.com/S1D007/boson.git
cd boson
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

Ready to get started? [Create your first project →](./hello-world)