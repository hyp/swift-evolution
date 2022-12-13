# Bridging Swift modules and top-level functions to C++

*   Proposal: [SE-03xy](03xy-swift-to-cxx-module-and-function-interop.md)
*   Authors: [Alex Lorenz](https://github.com/hyp)
*   Review Manager:
*   Status: 
*   Implementation: <TODO> Implemented on main as experimental feature (requires `-enable-experimental-cxx-interop` flag)

## Introduction

Swift-to-C++ interoperability requires C++ code to import Swift APIs into C++ in order
to be able to call them. Per the [vision document](TODO), Swift-to-C++ interoperability feature is modeled
on the existing Objective-C interoperability feature, as it uses the same generated header file in order to represent Swift APIs of a specific Swift module in C++. This proposal presents this header-based user model for Swift-to-C++ interoperability and talks about how it differs to the user model of Objective-C interoperability. It also describes how Swift module namespacing is handled when Swift APIs are bridged to C++. Finally, this proposal describes how top-level synchronous Swift functions that use primitive types in their signature are represented in the C++ section of the generated header file.

This proposal is the initial proposal for Swift-to-C++ interoperability feature of Swift. As such, it builds a foundation which future proposals will use to describe how different Swift language constructs and API patterns get mapped to C++. Certain design decisions presented in the proposal will be revisited in the future, <TODO: specific list>, and as such this proposal does not present the final design for the foundation layer of Swift-to-C++ interoperability.

Swift-evolution thread: [Pitch #1](TODO)

## Proposed solution

Rename the existing `-emit-objc-header` Swift compiler flag to `-emit-clang-header`. Keep `-emit-objc-header` flag for compatibility with existing build systems and makefiles.

Add additional C++ code section to the currently generated C/Objective-C header. This section provides C++ representation for the exposed Swift APIs. Wrap the C++ code section in preprocessor guards to ensure it's active only when the header is included in a C++ source file.

In the new C++ code section, generate a namespace declaration that represents the exposed Swift module to provide encapsulation for the exposed Swift APIs.

For each exposed top-level function, generate a C declaration that corresponds to the native Swift functions and matches its calling convention and ABI. These declarations can be accessed by any C/Objective-C/C++ client that includes the header.

For each exposed top-level function, generate a C++ inline thunk function whose name and signature are appropriately translated into C++. The inline thunk calls the C declaration that corresponds to the native Swift function. It's generated inside the module's namespace declaration.

For example, given the following module and function:

```swift
// Module Greeter

public func sayHello() {
  print("Hello world!")
}
```

The generated header file will include a C++ code section that resembles this code snippet:

```c++
// Greeter-Swift.h

#ifdef __cplusplus

namespace Greeter {

SWIFT_INLINE_THUNK void sayHello() {
  // call Swift's Greeter.sayHello
}

} // namespace Greeter

#endif // __cplusplus
```

The exposed `sayHello` function can then be called directly from C++ via the
inline thunk that's shown above:

```c++
#include "Greeter-Swift.h"

int main() {
  Greeter::sayHello();
  return 0;
}
```

## Detailed design


### Header generation

The Swift compiler generates a header file with C/Objective-C/C++ representation of
the exposed APIs in the Swift module when the `-emit-clang-header` flag is used. (`-emit-objc-header`).
The compiler can generate such header from different inputs:
- Swift module source files
- Serialized Swift module file
- Textual Swift module interface file
