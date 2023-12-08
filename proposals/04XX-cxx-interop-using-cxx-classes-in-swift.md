# C++ Interoperability: Using C++ classes and structures in Swift

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: Alex Lorenz
* Review Manager: TBD
* Status: Partially implemented in Swift 5.9 and Swift's main branch
* Vision: [Using C++ from Swift](https://github.com/apple/swift-evolution/blob/main/visions/using-c%2B%2B-from-swift.md)

## Introduction

This is the first of many evolution proposals that describes how Swift interoperates
with C++. This proposal builds upon the high-level interoperability vision outlined in the
[Using C++ from Swift](https://github.com/apple/swift-evolution/blob/main/visions/using-c%2B%2B-from-swift.md)
vision document. It's advised that the reader of this proposal familiarizes themselves
with the vision document prior to reading this proposal.

This proposal introduces the ability for Swift to interoperate with C++.
This
is presented as opt-in interoperability setting that the user can enable via
a compiler flag, or via a target setting in a Swift Package Manager manifest.
The support for C++ interoperability is built on top of the existing
support for C and Objective-C interoperability, and as such it relies on
them for the set of baseline interoperability features.

This proposal introduces the ability for Swift to import
copyable and/or movable C++ records (a C++ `class` or `struct` type)
as Swift structures. Such Swift structures use the C++ special members
like copy constructors or destructors for corresponding value operations in Swift.
This proposal also introduces the ability for Swift to import
field members of these records with public access control as Swift properties.
Lastly, this proposal introduces the ability for Swift to import and call
top-level C++ functions whose signatures are composed of types supported by
existing C and Objective-C interoperability, or the imported C++ record types.

## Motivation

Swift has been able to interoperate with C and Objective-C since its inception.
Interoperability with C++ has been something that both the designers
and the users of Swift wanted to see as well, as it
would enable the incremental adoption of Swift in existing C++ codebases. The [vision document](https://github.com/apple/swift-evolution/blob/main/visions/using-c%2B%2B-from-swift.md)
provides a more general insight into the motivation for why Swift and C++
should interoperate with each other.

The existing ability of Swift to import and use C structures is one of its
cornerstone interoperability features, as it allows Swift to call C functions that
take or return such structures. Now that Swift is starting to support C++
interoperability, the proposed ability
of Swift to import and use C++ record types is the key first step that is
needed to build up the support for interoperability with C++, as it allows
Swift to call C++ functions that take or return such structures. This provides
users access to an initial set of C++ APIs that can be used from Swift.

## Proposed solution

This proposal introduces the ability for Swift to interoperate with C++,
by importing headers grouped into
Clang modules using the C++ or the Objective-C++ language mode. The user can choose to opt-in
into C++ interoperability by setting the language interoperability mode, either
using a compiler flag, or using a target setting in a Swift Package Manager manifest.

C++ interoperability builds upon the existing support for C and Objective-C
interoperability, and as such it supports all of the C-compatible types
and functions that already get imported into Swift. It also supports the
currently supported Objective-C types and methods.

This proposal posits that Swift should import C++ records as Swift structures,
just like Swift imports C structures today. C++ records defined in
C headers should be imported in a way that's compatible with existing C
interoperability support, to avoid breaking existing Swift code that uses
imported C structures and that wants to start using C++ interoperability.

A C++ class or structure becomes a Swift structure type. Its fields
with public access control become stored or computed Swift properties.
Such Swift structure also gets a synthesized default initializer (that
zero-initializes the structure), unless the C++ record has a default
constructor. Such Swift structure also gets a synthesized
elementwise initializer, unless the C++ record has a constructor that
initializes the value.

In addition to supporting C-compatible top-level C++ functions, this proposal
introduces support for calling top-level C++ functions whose signature includes
C++ records that can be imported by Swift.

## Detailed design

This section provides a detailed overview of how users can enable
C++ interoperability, and describes how the supported C++ records and functions behave
when used in Swift.

### Enabling C++ interoperability

Users can choose to enable C++ interoperability by passing in the
`-cxx-interoperability-mode=default` flag to Swift. This flag
instructs the Clang compiler embedded into Swift to build and/or import
Clang modules using the C++ or the Objective-C++ language mode.

#### C++ interoperability mode

The `-cxx-interoperability-mode` flag must be used with a version value for
it work correctly. This version value determines what C++ interoperability features
are supported by Swift. The use of the version value allows Swift to provide
source compatibility for C++ interoperability, whilst the support for
C++ interoperability is evolving.

The following values are supported by the
 `-cxx-interoperability-mode` flag:

- `off`: This flag value disables C++ interoperability.

- `default`: This flag value enables C++ interoperability, and uses the Swift
   language version as the C++ interoperability version. For instance,
   when the Swift 5 language mode is used, Swift supports C++ interoperability
   features supported in Swift 5.9 interoperability version, as that's the
   version first supported for Swift 5. On the other hand,
   when the upcoming Swift 6 language mode is used, the C++ interoperability
   version becomes Swift 6 as well, and more C++ interoperability features
   become supported.

- `swift-5.9`: This flag value enables C++ interoperability, and instructs
   Swift that C++ interoperability support should only should only support
   features that are supported in Swift 5.9.

- `upcoming-swift`: This mode enables C++ interoperability, and enables
   all the supported C++ interoperability features that will be included in
   a future release of Swift. This flag is primarily useful for testing upcoming
   features, before a release of Swift is put together, and a new C++ interoperability
   version is created for Swift.

#### Swift Package Manager support

Users can turn on C++ interoperability for a specific Swift Package
Manager target using the `.interoperabilityMode(.Cxx)` Swift target setting.
Such setting instructs the compiler that the `default` interoperability mode
should be used.

#### C++ interoperability is viral

A Swift module that is built with C++ interoperability enabled can only
be imported by clients that also enable C++ interoperability.

Clients don't need to enable C++ interoperability, and can import Swift module interface files
that get generated for Swift modules that use C++ interoperability only when that Swift
module uses library evolution.

### Using POD C++ records

Trivial C++ plain old data (POD) C++ records become Swift structures. Just like imported C structures,
imported C++ POD records get a synthesized default initializer (that
zero-initializes the structure), and a synthesized elementwise initializer.
The example below illustrates how the two structures `Point` and `Line` get imported
into Swift.

```c++
// C header.

struct Point {
  int x;
  int y;
};

struct Line {
  struct Point start;
  struct Point end;
  unsigned int brush : 4;
  unsigned int stroke : 3;
};
```

These two structures get mapped to Swift structures in the following manner:

```swift
// C header imported in Swift.

struct Point {
  var x: CInt { get set }
  var y: CInt { get set }
  
  // Default initializer that sets all properties to zero.
  init()

  // Elementwise initializer.
  init(x: CInt, y: CInt)
}

struct Line {
  var start: Point { get set }
  var end: Point { get set }
  var brush: CUnsignedInt { get set }
  var stroke: CUnsignedInt { get set }

  // Default initializer that sets all properties to zero.
  init()

  // Elementwise initializer.
  init(start: Point, end: Point, brush: CUnsignedInt, stroke: CUnsignedInt)
}
```

The existing Swift code that imports C structures should be valid when
C++ interoperability is enabled as such C structures get imported as POD
types, unless their C++ definition in the headers is different to their C
definition.

The field members of POD types become properties, just like
field members of C structures.

### Using copyable C++ records

Non-POD C++ records that are copyable (they have an implicit or a user defined copy constructor)
become Swift structures. Swift calls
the record's copy constructor when a copy of the structure is performed by Swift.
Swift calls the record's destructor when such structure is destroyed in Swift.
For example, the C++ `Person` structure becomes a `Person` structure in Swift:

```c++
// C++ header imported in Swift.

#include <string>

class Person {
  std::string firstName;
  std::string lastName;
};

Person createSystemUserIdentity();
```

Users of the `Person` type in Swift then obey the Swift value type rules
as expected, and Swift calls the special members like copy constructor or destructor
as needed:

```swift
let person = createSystemUserIdentity()
// Swift calls the C++ copy-constructor when copying `person` into `anotherPerson`
let anotherPerson = person
// At the end of the scope, Swift calls the C++ destructor that destroys `person` and `anotherPerson`.
```

### Using move-only C++ records

Non-copyable C++ records that are movable (they have an implicit or a user defined
move constructor) become non-copyable Swift structures. Swift calls the record's
move constructor when a consume of a value is performed by Swift. Swift calls
the record's destructor when such structure is destroyed in Swift.

Move-only C++ records are not imported in the Swift 5.9 C++ interoperability version.
They become available when users use the upcoming Swift 6 language mode, or choose
to use the `upcoming-swift` interoperability version. (NOTE to reviewers: this will be replaced by a concrete version of Swift once we know its value).

### Importing copyable C++ records as non-copyable

A copyable C++ record that has a move constructor becomes a copyable
Swift structure by default. However, users can elect to import such record
as a non-copyable Swift structure instead, by annotating it with the
`SWIFT_NONCOPYABLE` macro that's defined in the `swift/bridging` header that
is shipped alongside Swift. This macro expands to the
```__attribute__((swift_attr("~Copyable")))``` attribute. This attribute allows Swift
to recognize such type and to map it to a non-copyable structure instead.

### Using C++ records with Swift generics

Imported C++ records become Swift structures, and as such can be used in a
Swift generic context. Swift automatically generates a value witness table for such
record. The value witness table is used to provide a set of generic operations
for such type, making it possible to use in a generic Swift context. The generated
value witness table and its functions are visible only to the single translation unit
that required their emission,
but the value witness table and its functions can be merged accross translation
units by the linker at link time.

#### Value witness operations for C++ records

The following value witness functions are generated for a C++ record:

- `destroy`.
  This value witness function invokes the C++ destructor for this value.
- `initializeWithCopy`.
  This value witness invokes the C++ copy constructor to create a new copy of an existing value.
  This value witness function is generated only for copyable C++ records.
- `assignWithCopy`.
  This value witness destroys the original value, and then invokes the C++ copy constructor
  to create a new copy of an existing value, in place of the original value.
  This value witness function is generated only for copyable C++ records.
- `initializeWithTake`
  This value witness invokes the C++ move constructor to move an existing value to a new value.
  This value witness function is generated only when the C++ record has an accessible move constructor.
- `assignWithTake`
  This value witness destroys the original value, and then invokes the C++ move constructor
  to move an existing value to to a new value, in place of the original value.
  This value witness function is generated only when the C++ record has an accessible move constructor.

### C++ record type layout

Even though C++ records become Swift structures when imported into Swift,
Swift uses the original C++ type layout when generating code that operates on
such structures. The alignment of the C++ record is respected
by Swift, so a Swift variable or a stored property follows the alignment
rules specified for that specific C++ record. Swift allocates sufficient
storage for a variable or stored property of such type as well, and the amount
of allocated storage matches the size of the C++ record.

Since the type layout for imported C++ records matches the C++ type layout,
Swift can take unsafe pointers using standard APIs like `withUnsafePointer` to
construct an `UnsafePointer` / `UnsafeMutablePointer` value that can be passed
back to C++ when the user wants to call a C++ function that takes in a raw pointer
to the C++ record.

### C++ record lifetime

Imported C++ records follow Swift lifetime rules, which are different than the
C++ lifetime rules. For instance, Swift can destroy a variable right away after its last use, whereas
C++ has to wait until the end of the scope. This difference is illustrated below,
using the `LogFile` C++ structure:

```c++
struct LogFile {
  ~LogFile() {
    std::cout << "the log file is closed\n";
  }
};

LogFile openLog();
void recordEvent(const LogFile *log, const char *event);
```

In Swift, a `LogFile` variable could be destroyed after its last use, before
the `print` function is invoked:

```swift
func testLogging() {
  let log = openLog()
  recordEvent(&log, "event")
  // `log` can now be destroyed.
  print("done logging!")
}
```

The standard output in that case will show:

```
the log file is closed
done logging
```

However, in C++, an equivalent `LogFile` variable would be destroyed only
at the end of scope of the `testLogging` function:

```c++
void testLogging() {
  auto log = openLog();
  recordEvent(&log, "event");
  std::cout << "done logging!\n";
  // `log` is destroyed before the function returns.
}
```

### Accessing fields of imported records

Publicly accessible field members whose type is either a copyable imported C++ record, or
a type already supported by C and Objective-C interoperability, become Swift
properties that have a getter accessor, and a setter accessor unless they're constant fields.
The user is then able to obtain or change the field value in the record.

#### Working with non-copyable fields

Publicly accessible field members whose type is a non-copyable imported C++ record become Swift properties
that have an `unsafeAddress` accessor, and an `unsafeMutableAddress` accessor unless
they're constant fields. The user is able to borrow the field's value, using
the `borrowing` or `inout` parameter specifiers. For instance, a field whose
type is a C++ move-only `UniqueResource` structure is imported into Swift:

```c++
struct UniqueResource {
  UniqueResource(const UniqueResource &) = delete;
  UniqueResource(UniqueResource &&) { ... }

  int timeAquired;
  ...
};

struct Holder {
  UniqueResource resource; // this non-copyable field is imported into Swift.
};

Holder makeHolder();
```

Swift users can then borrow this field, for instance, by
passing it to a `borrowing` parameter:

```swift
func logInfo(resource: _ borrowing UniqueResource) {
  print("resource was aquired \(resource.timeAquired) seconds in")
}

let holder = makeHolder()
logInfo(holder.resource)
```

#### Inherited fields are not yet imported

This proposal does not allow Swift users to use publically accessible
fields in C++ records that are inherited from a derived C++ class.
Such fields will be covered by subsequent proposals that describe how
inherited members get imported into Swift.

### Calling top-level C++ functions from Swift

C interoperability allows Swift to call C functions, by mapping a C function
to a top-level Swift function. C++ interoperability
builds upon the existing support for C and Objective-C interoperability, by mapping users
to call top-level C++ functions whose signature is composed of types that
are supported by C and Objective-C interoperability. The existing Swift code that
calls such functions should still be valid when C++ interoperability is enabled.

In addition to supporting C-compatible top-level C++ functions, this proposal
introduces support for calling top-level functions whose signature includes
C++ records that can be imported by Swift. A C++ record parameter or return
type gets mapped to its corresponding Swift structure type when such C++ function is
mapped to a Swift function. A raw pointer to such type becomes `UnsafePointer` or
`UnsafeMutablePointer`. A raw pointer to a forward declared C++ record becomes
an `OpaquePointer` (thus matching the current C interoperability rules).

Top-level functions whose signature includes types that can't be imported by Swift
are not imported as well. Such types include unsupported record types
(e.g. a non-movable and non-copyable record), C++ references, and other
types are not yet supported by C++ interoperability. Such types and functions
will be covered by future proposals.

### C++ function ABI

A C++ function becomes a Swift function when imported into Swift, however,
Swift uses the underlying C++ ABI when it makes the call to C++. Such call
is made directly from Swift code to C++, without going through any kind of
indirection (unless mandated by the C++ ABI).
Swift uses the appropriately mangled function name when
invoking the C++ function as well. Swift uses the appropriate calling convention
for such calls as well.

Swift follows the proper C++ ABI rules when passing a parameter to a C++ function.
For instance, Swift may pass a POD C++ record that contains an `int` field in
directly a single register. On the other hand, a non-trivial C++ record is most
likely going to be passed indirectly, by passing a pointer to it instead.

Swift also follows the proper C++ ABI rules when obtaining the result
value returned by a call to a C++ function. For instance, a POD C++ record with two
`int` fields may be returned directly in a register wide enough to hold the entire
the record, and thus Swift will construct the resulting structure using the
data stored in the return register. On the other hand, a non-trivial C++ record
can be returned indirectly, by passing in an additional pointer parameter
to the function, and thus Swift will pass in a pointer to storage for the
returned value when making such call.

### Handling uncaught exceptions

C++ functions can throw exceptions. Swift does not supporting catching exceptions, so
Swift will terminate the execution of the program if an uncaught exception
reaches a Swift frame. Swift assumes that `noexcept` function do not throw
exceptions, and thus any exception thrown from such function will not
terminate the execution of the program, and will instead lead to undefined
behavior.

## Source compatibility

To preserve source compatibility for existing Swift code, C++ interoperability is an opt-in
feature. This ensures that Swift can build existing code,
even when a Swift module is importing headers that have different APIs or types in the C++ / Objective-C++ language mode,
as they still get imported in the C / Objective-C language mode.

Once users enable C++ interoperability, their existing Swift code should not break,
if the headers that they're importing maintain the same APIs and types when imported
in the C++ / Objective-C++ language mode. This makes it easier for users to transition
to C++ incrementally, especially if they have an existing C or Objective-C bridging
layer that they need to use while making the transition over to using the C++ APIs
directly from Swift.

C++ interoperability support itself is versioned using the version value
passed to the `-cxx-interoperability-mode`, as described in the detailed
design section above. This ensures that source compatibility can be preserved
when users upgrade to a new Swift compiler release without upgrading to a new
Swift language version. However, an upgrade to a new Swift language version can
introduce breaking changes to users code in places where C++ APIs are used in
Swift. However, users will be able to fallback to the old C++ interoperability
version if needed, by passing in an older interoperability version to the
`-cxx-interoperability-mode` flag.

## ABI compatibility

This proposal does not change Swift ABI and thus has no impact on ABI compatibility.

## Effect on ABI stability

The Swift compiler prohibits the use of imported C++ types at the ABI stable
Swift API boundary, so ABI stability for Swift should not be affected. Specifically,
Swift prohibits the use of C++ types in public API signatures or inlinable
bodies when a Swift module is being built with library evolution enabled.

## Effect on API resilience

Existing Swift APIs are unaffected by this proposal, even if they use
C types. C++ types in general should not be used for public APIs that want
to provide API resilience, as C++ types can not provide ABI resilience.

## Future directions

This proposal describes how to import a subset of C++ records. Other kinds
of records, like the ones listed below, will be covered by subsequent
proposals:

* Non-copyable __and__ non-movable C++ records.
* Class templates and class template specializations
* Records that use `trivial_abi` Clang attribute.

Future proposals will also provide describe how certain existing Swift language features
will interact with imported C++ records. For instance, this proposal doesn't
describe how retroactive protocol conformances can be applied to C++ records.
That will be covered by future proposals.

This proposal describes how C++ field members defined in the imported
C++ record are imported into Swift. Other members defined in the imported
C++ record, like constructors and member functions, or members inherited from base classes
will be covered by future proposals.

Future proposals will describe how other C++ language features like namespaces
and templates are imported into Swift.

## Acknowledgments

Zoe Carver contributed to the initial design and wrote an early draft of this proposal.
