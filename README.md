# PPConsul

*Version 0.1*

A C++ client library for [Consul](http://consul.io). Consul is a distributed tool for discovering and configuring services in your infrastructure.

The goal of PPConsul is to:
* Fully cover version 1 of Consul [HTTP API](http://www.consul.io/docs/agent/http.html). Please check the current [implementation status](status.md).
* Provide simple, modular and effective API based on C++11.
* Support different platforms. At the moment, Linux, Windows and macOS platforms supported.
* Cover all the code with automated tests.
* Have documentation.

Note that this project is just started so it is under active developing, doesn't have a stable interface and was not tested enough.
So if you are looking for something stable and ready to be used in production then it's better to come back later or help me growing this project to the quality level you need.

Library tests are currently running against **Consul v0.7.2**. Library is known to work with Consul starting from version **0.4** (earlier versions might work as well but it has never been tested) although some tests fail for older versions because of not-supported features.

The library is written in C++11 and requires a quite modern compiler. Currently it has been tested with:
* Windows: Visual Studio 2013 Update 3
* OS X: Clang 7 (Xcode 7), Clang 6 (Xcode 6.1), Clang 5 (Xcode 5.1) all with libc++.
* Linux: GCC 5.3, GCC 4.9, GCC 4.8.2

Please check [PPConsul build status](https://136.243.151.173:4433/project.html?projectId=Ppconsul&guest=1).

The newer versions of specified compilers should work fine but was not tested. Earlier versions of GCC and Clang may work. Visual Studio before 2013 definitely gives up.

The library depends on:
* [Boost](http://www.boost.org/) 1.55 or later. PPConsul needs only headers with one exception: using of GCC 4.8 requires Boost.Regex library because [regex is not supported in GCC 4.8](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=53631).
* [libCURL](http://curl.haxx.se/libcurl/) **or** [C++ Network Library](http://cpp-netlib.org/) (aka cpp-netlib) to deal with HTTP. Note that the latter depends on compiled Boost libraries.

The library includes code of the following 3rd party libraries (look at `ext` directory): 
* Slightly tweaked [version](https://github.com/oliora/json11) of [json11](https://github.com/dropbox/json11) library to deal with JSON.
* [libb64](http://libb64.sourceforge.net/) library for base64 decoding.

For unit tests, the library uses [Catch](https://github.com/philsquared/Catch) framework. Many thanks to Phil Nash for this great product.

## Warm Up Examples

Register, deregister and report the state of your service in Consul:


```cpp
#include "ppconsul/agent.h"

using namespace ppconsul;

// Create a consul client with using of a default address ("127.0.0.1:8500") and default DC
Consul consul;
// We need the 'agent' endpoint for a service registration
agent::Agent agent(consul);

// Register a service with associated HTTP check:
agent.registerService(
    agent::keywords::name = "my-service",
    agent::keywords::port = 9876,
    agent::keywords::tags = {"tcp", "super_server"},
    agent::keywords::check = agent::HttpCheck{"http://localhost:80/", std::chrono::seconds(2)}
);

...

// Unregister service
agent.deregisterService("my-service");

...

// Register a service with TTL
agent.registerService(
    agent::keywords::name = "my-service",
    agent::keywords::port = 9876,
    agent::keywords::id = "my-service-1",
    agent::keywords::check = agent::TtlCheck{std::chrono::seconds(5)}
);

// Report service is OK
agent.servicePass("my-service-1");

// Report service is failed
agent.serviceFail("my-service-1", "Disk is full");
```

Use Key-Value storage:

```cpp
#include "ppconsul/kv.h"

using namespace ppconsul;

// Create a consul client with using of a default address ("127.0.0.1:8500") and default DC
Consul consul;

// We need the 'kv' endpoint
kv::Storage storage(consul);

// Read the value of a key from the storage
std::string something = storage.get("settings.something", "default-value");

// Read the value of a key from the storage with consistency mode specified
something = storage.get("settings.something", "default-value", keywords::consistency = Consistency::Consistent);

// Erase a key from the storage
storage.erase("settings.something-else");

// Set the value of a key
storage.put("settings.something", "new-value");
```

Blocking query to Key-Value storage:

```cpp
// Get key-value item
kv::KeyValue item = storage.item("status.last-event-id");

// Wait for the item change for no more than 1 minute:
item = storage.item("status.last-event-id", keywords::block_for = {std::chrono::minutes(1), item.modifyIndex});

// If key exists, print it:
if (item)
    std::cout << item.key << "=" << item.value << "\n";
```

## Documentation
TBD

## How To Build

### Get Dependencies
* Get C++11 compatible compiler. See above for the list of supported compilers.
* Install [CMake](http://www.cmake.org/) 3.0 or above.
* Install [Boost](http://www.boost.org/) 1.55 or later. You need compiled Boost libraries if you going to use cpp-netlib or GCC 4.8, otherwise you need Boost headers only.
* Install either [libCURL](http://curl.haxx.se/libcurl/) (any version is OK) or [cpp-netlib](http://cpp-netlib.org/) 0.11 or above.
* Install [Consul](http://consul.io) 0.4.0 or above. *It's only needed to run tests.*

### Build

Prepare project:  
```bash
mkdir workspace
cd workspace
cmake ..
```

* To use cpp-netlib instead of libCURL use `-DUSE_CPPNETLIB=1` when invoking CMake.
* To change where the build looks for Boost, pass `-DBOOST_ROOT=<path_to_boost>` parameter to CMake or set `BOOST_ROOT` environment variable.
* To change where the build looks for libCURL, pass `-DCURL_ROOT=<path_to_curl>` parameter to CMake or set `CURL_ROOT` environment variable.
* To change the default install location pass `-DCMAKE_INSTALL_PREFIX=<prefix>` parameter to CMake.

*Note about -G option of CMake to choose you favourite IDE to generate project files for.*

Build on Linux/macOS:  
```bash
make
```

Build on Windows:  
Either open generated solution file `workspace\ppconsul.sln` in the Visual Studio or build from the command line:  
```bash
cmake --build . --config Release
```

## How To Run Tests

### Run Consul

You need to run Consul with using of `ppconsul_test` datacenter. There is a helper script for this:  
```bash
tests/start_consul.sh start
```

If you have Consul 0.7 or above then it's probably easier to run it directly:  
```bash
consul agent -dev -datacenter=ppconsul_test
```

### Run Tests

Run tests on Linux/macOS:  
```bash
cd workspace
make test
```

Run tests on Windows:  
Either build `RUN_TESTS` target in the Visual Studio or build from the command line:  
```bash
cd workspace
ctest -C Release
```

There are the following environment variable used by the tests:

|Name|Default Value|Description|
|----|-------------|-----------|
|`PPCONSUL_TEST_ADDR`|"127.0.0.1:8500"|The Consul network address|
|`PPCONSUL_TEST_DC`|"ppconsul_test"|The Consul datacenter|

### Known Problems

Sometimes catalog tests failed on assertion `REQUIRE(index1 == resp1.headers().index());`. In this case, just rerun the tests.
The reason for the failure is Consul's internal idempotent write which cause a spurious wakeup of waiting blocking query. Check the critical note under the blocking queries documentation at https://www.consul.io/docs/agent/http.html.

## Using PPConsul In Your Application
TBD

For now you can check [`tests`](https://github.com/oliora/ppconsul/tree/master/tests) directory for PPConsul-dependent application examples.

## Found a bag? Got a feature request? Need help with PPConsul?
Use [issue tracker](https://github.com/oliora/ppconsul/issues).

## Contribute
First of all, welcome on board!

To contribute, please fork this repo, make your changes and create a pull request.

## License
The library released under [Boost Software License v1.0](http://www.boost.org/LICENSE_1_0.txt).
