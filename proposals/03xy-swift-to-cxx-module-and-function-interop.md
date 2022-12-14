# Bridging Swift modules and top-level functions to C++

*   Proposal: [SE-03xy](03xy-swift-to-cxx-module-and-function-interop.md)
*   Authors: [Alex Lorenz](https://github.com/hyp)
*   Review Manager:
*   Status:
*   Implementation: <TODO> Implemented on main as experimental feature (requires `-enable-experimental-cxx-interop` flag)
*   Vision document: [Using Swift from C++](https://github.com/apple/swift-evolution/blob/main/visions/using-swift-from-c%2B%2B.md)

## Introduction

Swift-to-C++ interoperability requires C++ code to import Swift APIs into C++ in order
to be able to call them. Per the [vision document](https://github.com/apple/swift-evolution/blob/main/visions/using-swift-from-c%2B%2B.md), Swift-to-C++ interoperability feature is modeled
on the existing Objective-C interoperability feature, as it uses the same generated header file in order to represent Swift APIs of a specific Swift module in C++. This proposal presents this header-based user model for Swift-to-C++ interoperability and talks about how it differs to the user model of Objective-C interoperability. It also describes how Swift module namespacing is handled when Swift APIs are bridged to C++. Finally, this proposal describes how top-level synchronous Swift functions that use primitive types in their signature are represented in the C++ section of the generated header file.

This proposal is the initial proposal for Swift-to-C++ interoperability feature of Swift. As such, it builds a foundation which future proposals will use to describe how different Swift language constructs and API patterns get mapped to C++. Certain design decisions presented in the proposal will be revisited in the future, <TODO: specific list>, and as such this proposal does not present the final design for the foundation layer of Swift-to-C++ interoperability.

Swift-evolution thread: [Pitch #1](TODO)

## Proposed solution

Rename the existing `-emit-objc-header` Swift compiler flag to `-emit-clang-header`. Rename the existing `-emit-objc-header-path` Swift compiler flag to `-emit-clang-header-path`. Keep `-emit-objc-header` and `-emit-objc-header-path` flags for compatibility with existing build systems and makefiles.

Add additional C++ code section to the currently generated C/Objective-C header. This section will provide C++ representation for the exposed Swift APIs. Wrap the C++ code section in a `__cplusplus` preprocessor guard to ensure it's active only when the header is included in a C++ source file. Wrap the previously emitted Objective-C code section in an `__OBJC__` preprocessor guard to ensure it's active only when the header is included in an Objective-C[++] source file.

In the new C++ code section, generate a `namespace` declaration that represents the exposed Swift module to provide encapsulation for the exposed Swift APIs.

For each exposed top-level function, generate a C declaration that corresponds to the native Swift function and matches its calling convention and ABI. These declarations can be accessed by any C/Objective-C/C++ client that includes the header.

For each exposed top-level function, generate a C++ inline thunk function whose name and signature are appropriately translated into C++. The inline thunk calls the C declaration that corresponds to the native Swift function. It's generated inside the module's namespace declaration.

### Example 1: calling Swift function from C++

This example demonstrates how to call a top-level Swift function from C++.
Given the following module `Greeter` and function:

```swift
public func sayHello() {
  print("Hello world!")
}
```

The exposed Swift `sayHello` function can then be called directly from C++:

```c++
#include "Greeter-Swift.h"

int main() {
  Greeter::sayHello();
  return 0;
}
```

### Example 2: Unified C/Objective-C/C++ header generation

This example demonstrates how existing `@objc` and `@_cdecl` annotated APIs
will coexist with other APIs exposed to C++ in a single unified generated header.
Given the following module and function:

```swift
// Module MixedLanguage

@_cdecl("factorial")
public func factorial(_ x: CInt) -> CInt { ... }

@objc
class ObjClass: NSObject {

  @objc
  public func objcMethod() { ... }
}

public func isPrime(_ x: CInt) -> Bool { ... }
```

The generated header file will then expose the APIs shown above in a way that resembles this code snippet:

```c++
// MixedLanguage-Swift.h

SWIFT_EXTERN int factorial(int);

#ifdef __OBJC__

@interface ObjClass

- (void)objcMethod;

@end

#endif // __OBJC__

#ifdef __cplusplus

namespace MixedLanguage {

SWIFT_INLINE_THUNK bool isPrime(int x) {
  // call Swift's MixedLanguage.isPrime
}

} // namespace MixedLanguage

#endif // __cplusplus
```

## Detailed design

The detailed design section of this proposal describes the updated multi-language
layout of the generated header that was previously used for Objective-C interoperability only.
It then describes the representation of Swift modules in C++, and mentions the
builtin interoperability header shipped with the Swift compiler.
The remaining subsections describe the representation of top-level Swift functions
and primitive Swift types in the generated header.

### Unified header generation

The Swift compiler generates a header file with the C/C++/Objective-C[++] representation of
the exposed APIs in the Swift module when the `-emit-clang-header` or the `-emit-objc-header` flag is passed.

The header file can be generated from different inputs:
- Swift module source files
- Serialized Swift module file
- Textual Swift module interface file

The generated header file contains multiple sections. Each section provides representation for the exposed
APIs in a different language mode. The following sections can be present in the header:
- C section: Contains C function declarations with appropriate ABI annotations that represent exposed Swift functions. This section also contains the C function declarations for the `@_cdecl` Swift functions.
- Objective-C section: Provides representation for the `@objc` declarations. Its content is identical to the Objective-C content in the header that was emitted with the previously supported `-emit-objc-header` flag.
- C++ section: Provides representation for the exposed Swift declarations that are representable in C++.
- Objective-C++ section: Provides additional glue code to connect declarations in the Objective-C and C++ sections, or additional APIs for emitted C++ types in Objective-C++.
 
Each language-specific section is guarded with appropriate preprocessor guards to ensure
that the translation unit that includes this header will only see the declarations that are
supported in the language mode that's used to build it. This code snippet shows an example of how different sections are guarded in the header file:

```c++
// This section contains C declarations annotated with ABI attributes for native Swift functions.

#ifdef __OBJC__
  // This section contains exposed `@objc` declarations.
#endif

#ifdef __cplusplus
  // This section contains exposed Swift declarations that are representable in C++.

  #ifdef __OBJC__
    // This section contains any Objective-C++ specific glue code and API enhancements.
  #endif
#endif
```

#### Generated header name

Swift users can choose the file name of the generated
header using the `-emit-clang-header-path` flag. The recommended naming
scheme for such header file follows the currently recommended naming scheme
for Objective-C headers, i.e. `ModuleName-Swift.h"`. This recommended naming
scheme might change in the future.

### C++ namespace wraps APIs from one module

The generated C++ code that represents exposed Swift APIs in a module is placed
into a C++ `namespace` declaration. The name of the `namespace` declaration is
identical to the Swift module name. For example, a Swift module `Greeter`
maps to a C++ namespace with the same name, as represented by this sample snippet
from the generated header file:

```C++
#ifdef __cplusplus

namespace Greeter {
  // C++ declarations representing Swift APIs in the 'Greeter' module.
} // namespace Greeter

#endif // __cplusplus
```

The Swift standard library encapsulates its APIs using the `swift` namespace
in C++.

### Builtin Swift compiler interoperability header

The Swift compiler provides a builtin Swift-to-C++ interoperability header file
that contains some useful macro definitions that are used in the generated header.
This header also provides C++ type aliases that correspond to some fundamental Swift standard library types. Specifically, it provides definitions for the `Int` and `UInt` types in the `swift` C++ namespace:

```c++
namespace swift {

/// Represent Swift's `Int` type.
using Int  = ptrdiff_t;

/// Represents Swift's `UInt` type.
using UInt = size_t;

} // namespace swift
```

The builtin header Swift-to-C++ interoperability header is shipped with the compiler,
 **not** the standard library. This is done intentionally, as when the Clang
 compiler is included in the same toolchain as the Swift compiler, the Clang compiler
 can find this builtin header using a relative path that's relative to Clang's builtin
 headers. Other compilers or compilers in a separate toolchain will require the user
to specify an include path flag that points to Swift's `<toolchain>/lib/swift` path
when compiling their C/C++ sources, to ensure that this header can be found during their build.

### Exposing top-level functions to C/C++

The Swift compiler examines all public top-level functions in a Swift module
when determining which functions can be exposed to C/C++ in the generated header.
Each exposed function must satisfy the following constraints:
- it is synchronous, i.e. not `async`.
- it is not generic.
- it returns just one value or `Void`/`Never`.
- it does not `throw` any Swift errors.
- it is not annotated with `@backDeploy` or `@_alwaysEmitIntoClient`.
- the types used in its signature are covered by approved evolution proposals.

These constraints are going to be removed and/or relaxed in future Swift-to-C++
interoperability evolution proposals.

The function signature of each exposed Swift function is mapped to a
C and a C++ function signature. The C signature is needed when generating the C
declaration that represents the native Swift function. The C++ signature is
needed when generating the C++ inline thunk that invokes the native Swift function.
Each parameter and return type used for function's signature is mapped to its
corresponding C and C++ type. The next section describes how some fundamental Swift primitive
types that are defined in the standard library get represented in C and C++.

### Supported primitive Swift types

This section specifies the supported set of primitive Swift types that an
exposed function can use in its signature. Any type that is not
mentioned in this section is not yet supported and thus a function
with such type is not exposed to C and C++.

Swift's `Int` and `UInt` get mapped to different C and C++ types:

| Swift Type | C type | C++ type |
|---|---|---|
| `Int`        | `ptrdiff_t` | `swift::Int` |
| `UInt`       | `size_t` | `swift::UInt` |

The `swift::Int` and `swift::UInt` type aliases are defined in the builtin Swift-to-C++
interoperability header.

Several other primitive types are mapped to the same type in both C and C++:

| Swift Type | C / C++ type |
|---|---|
| `Void`       | `void` |
| `Float` / `CFloat`      | `float` |
| `Double` / `CDouble`    | `double` |
| `Bool` / `CBool` | `bool` |
| `CInt` | `int` |
| `CUnsignedInt` | `unsigned int` |
| `CShort` | `short` |
| `CUnsignedShort` | `unsigned short` |
| `CLong` | `long` |
| `CUnsignedLong` | `unsigned long` |
| `CLongLong` | `long long` |
| `CUnsignedLongLong` | `unsigned long long` |
| `CChar` | `char` |
| `CWideChar` | `wchar_t` |
| `CChar16` | `char16_t` |
| `CChar32` | `char32_t` |

Some Swift pointer types are mapped to raw C/C++ pointers with an appropriate
Clang nullability annotation:

| Swift Type      | C / C++ type |
|---|---|
| `OpaquePointer`     |  `void * _Nonnull` |
| `UnsafePointer<T>`  | `const T * _Nonnull` |
| `UnsafeMutablePointer<T>` | `T * _Nonnull` |

Such pointer types get mapped to `_Nullable` raw pointers if they're wrapped in
an `Optional` type on the Swift side:

| Swift Type      | C / C++ type |
|---|---|
| `Optional<OpaquePointer>>`     |  `void * _Nullable` |
| `Optional<UnsafePointer<T>>`  | `const T * _Nullable` |
| `Optional<UnsafeMutablePointer<T>>` | `T * _Nullable` |

### Top-level function declaration in C

Each exposed top-level function is mapped to a C function declaration that
matches Swift's native convention and ABI. The name of this C function is
the mangled name of the Swift function. The C function declaration is
placed in the unguarded section of the header, so that it can be accessed by
C, C++ and Objective-C[++] clients. In C++ mode, this C function declaration
is hidden in an `_impl` namespace in the C++ module namespace. This ensures
that this function is hidden from code-completion results for both top-level
code completions and completions that start with the module namespace qualifier.

The Swift snippet presented below demonstrates how a Swift function `sayHello`
from module `Greeter` is represented in C:

```swift
public func sayHello() {
  print("Hello world!")
}
```

The generated header for `Greeter` will then declare the C function `sayHello`
using the following C/C++ code:

```c++
#ifdef __cplusplus
namespace Greeter {
namespace _impl {
#endif // __cplusplus

SWIFT_EXTERN void $s7Greeter8sayHelloyyF(void) SWIFT_NOEXCEPT SWIFT_CALL;

#ifdef __cplusplus
} // namespace _impl
} // namespace Greeter
#endif // __cplusplus
```

The macros shown in the C/C++ code snippet above are defined in the builtin Swift-to-C++
interoperability header. They apply various attributes to the C function:

- `SWIFT_EXTERN`:   Applies `extern "C"` to the C function declaration in C++ mode only.
- `SWIFT_NOEXCEPT`: Applies `noexcept` to the C function declaration in C++ mode only.
- `SWIFT_CALL`:     Applies `__attribute__((swiftcall))` to the C function declaration in all language modes.

### Top-level function declaration in C++

Each exposed top-level function is mapped to a C++ inline function. It is considered
to be a thunk function as it has a body that invokes the underlying Swift function.
This thunk is responsible for multiple things:
- Validating the types passed to this function to ensure that the Swift function is called with correct types. This can be done statically using the C++ type signature.
- Obtaining Swift argument values from the given C++ argument values.
- Constructing C++ return values from the Swift return values.

Future proposals may augment the set of responsibilies shown above as needed.

The name of the C++ function is the base name of the Swift function, unless
there are other exposed functions that have the same base name, and the same
number of parameters. In such cases the name of the C++ function is augmented with
Swift argument labels in their order of appearance, until it is different than the other C++ names
of functions with the same base name and number of parameters.

For example, these three functions with the same `sayHi` basename:

```swift
func sayHi()
func sayHi(to: Int)
func sayHi(_: Int)
```

Will be represented using the following signatures in C++:

```c++
void sayHi();
void sayHiTo(swift::Int);
void sayHi(swift::Int);
```

Argument labels are incorporated into the name until a unique name is generated.
For example these two `clamp` functions:

```swift
func clamp(_ value: Int, before: Int, except: Int)
func clamp(_ value: Int, upTo: Int, except: Int)
```

will not use the `except` argument label in their C++ name:

```c++
void clampBefore(swift::Int, swift::Int, swift::Int);
void clampUpTo(swift::Int, swift::Int, swift::Int);
```

This proposal reserves the right to change how argument labels are incorporated
into the name of the C++ function in future proposals.

The C++ function declaration is placed in the C++ section of the header, inside
the module namespace declaration.

The `sayHello` function from the example shown in the prior section is represented
in the generated header in the following manner:

```c++
#ifdef __cplusplus
namespace Greeter {

SWIFT_INLINE_THUNK void sayHello() noexcept {
  $s7Greeter8sayHelloyyF();
}

} // namespace Greeter
#endif // __cplusplus
```

The `SWIFT_INLINE_THUNK` macro shown above applies `inline` specifier and the `always_inline`
attribute to the function. It also applies the `transparent` Clang attribute to the function (when
supported by the compiler). The `transparent` attribute allows the debugger to skip
over the thunk when the user wants to step into the Swift function from its C++
call site. This macro is defined in the builtin Swift-to-C++ interoperability header.

A documentation comment is attached to the generated C++ function declaration
if the original Swift function has one attached to it as well.

Appropriate availability annotations are attached to the generated C++ function
declaration if the original Swift functions has any `@available` attributes.
The generated header uses the `SWIFT_AVAILABILITY`, `SWIFT_UNAVAILABLE` and
`SWIFT_DEPRECATED` macros for availability annotations. These macros are already
defined in the generated header as they've been used for Objective-C declarations.
For example, if a function `sayHelloCursive` with an `@available` attribute
is added to the module `Greeter`:

```swift
/// Says hello using a new cursive system font.
@available(macos, introduced: 11.0)
public func sayHelloCursive() {
  ...
}
```

The generated header for `Greeter` will then declare the C++ `sayHelloCursive`
thunk using the following C++ code:

```c++
/// Says hello using a new cursive system font.
SWIFT_INLINE_THUNK void sayHelloCursive() noexcept SWIFT_AVAILABILITY(macos,introduced=11.0) {
  $s7Greeter15sayHelloCursiveyyF();
}
```

### Function return values

The C++ function thunk that returns a value whose type is one of the primitive
types that's listed in this proposal obtains the return value by calling
the C function that represents the native Swift function directly. For example,
given a new function in module `Greeter` that returns an `Int`:

```swift
public func helloLimit() -> Int {
  ...
}
```

The generated thunk for `helloLimit` will return the value returned by
Swift's `helloLimit` using the `return` statement:

```c++
SWIFT_INLINE_THUNK swift::Int helloLimit() noexcept SWIFT_WARN_UNUSED_RESULT {
  return $s7Greeter15sayHelloCursiveyyF(); // TODO: fix the mangled name.
}
```

A C/C++ function that represents a Swift function that returns a value
is annotated with `SWIFT_WARN_UNUSED_RESULT` in the generated header, unless
the Swift function is annotated with the `@discardableResult` attribute.
`SWIFT_WARN_UNUSED_RESULT` expands to the `warn_unused_result` attribute, if
the compiler supports it. This attribute instructs the C++ compiler to warn when
the result is unused.

Swift functions that returns `Never` return `void` but receive
an additional `SWIFT_NORETURN` annotation in the generated header. The `SWIFT_NORETURN`
expands to the `noreturn` attribute, if the compiler supports it.

### Function parameters

The parameters of both the C and the C++ function correspond to the parameters
of the exposed Swift function. The name of the C/C++ parameter is the name of
the Swift parameter, or a number preceded by `_` if the parameter is unnamed.
Default parameter values are ignored.

Parameters with primitive types are passed to the native Swift function directly.
In-out parameters and variadic parameters are not yet supported,
and functions with such parameters are not exposed to C++.

The following example demonstrates how parameters are incorporated into the
C++ function signature and its body. Given the following function in module
`Greeter`:

```swift
public func sayHellos(numberOfHellos: Int = 1) {
  ...
}
```

The generated C++ thunk for `sayHellos` will look like this:

```c++
SWIFT_INLINE_THUNK void sayHellos(swift::Int numberOfHellos) noexcept {
  return $s7Greeter15sayHellosyyF(numberOfHellos); // TODO: fix the mangled name.
}
```

### Unexposed public functions are unavailable in C++

Top level public functions that are not exposed due to constraints mentioned
in the "Exposing top-level functions to C/C++" section are represented as an unavailable
C++ function declaration in the generated header, unless the module's C++ namespace
already contains a C++ function with the same name as the unexposed function.

For example, this Swift function:

```swift
public func printAnyValue<T>(_: T)
```

will be represented as the following unavailable function in the generated
header, giving a user a clear error as to why it can't be called from C++:

```c++
void printAnyValue() SWIFT_UNAVAILABLE_MSG("generic function not yet exposed to C++");
```

However, this one exposed and one unexposed function with the same name:

```swift
public func printValue(_ val: Int) { }
public func printValue<T>(_ val: T) { }
```

Will only be represented as one inline thunk:

```c++
SWIFT_INLINE_THUNK void printValue(swift::Int) noexcept { ... }
```

## Source compatibility

This is an additive feature that does not impact on Swift's source compatibility.

## Generated header source compatibility

This is an additive feature to the existing generated header that contains
Objective-C declarations that represent Swift's `@objc` declarations. It does
not impact existing Objective-C clients. Existing Objective-C++ clients could
be impacted if they have namespaces that conflict with the name of the generated
namespace, or the `swift` namespace.

### Generated C++ source compatibility

The design for C++ namespace declarations that represent the Swift module are
finalized in this proposal. Their uses in C++ name qualifiers or
`using namespace` declarations is going to be compatible with future versions of
the compiler Swift that will add things to the generated header.

The C++ representation of the Swift top-level functions in the generated header
is finalized in this proposal, except that the rules that determine the name
of the C++ function might change. However, such changes are expected to be minor.
Moreover, these changes will try their best to preserve source compatibility for some time
by preserving the C++ function thunks with the old names and just marking them
as deprecated.

## Effect on ABI stability

This is an additive change that has no direct impact that compromises ABI stability.

### ABI stability for C++ users

C++ users are guaranteed ABI stability when calling exposed
top-level functions from Swift modules that opt in into library evolution.

## Effect on API resilience

This is an additive change that has no impact on API resilience.

## Alternatives considered

TODO:
- why header?
- why namespace?
- why function name argument labels?

## Future directions

TODO

## Acknowledgments

TODO: Thanks to John, Becca, Zoe.

