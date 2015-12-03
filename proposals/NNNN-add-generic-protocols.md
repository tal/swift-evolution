# Generic Protocols

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/proposals/NNNN-add-generic-protocols.md)
* Author(s): [Tal Atlas](https://github.com/tal)
* Status: **Review**
* Review manager: TBD

## Introduction

This feature would allow for the definition of a type which has internal types
specified. This allows for APIs to be protocol based for protocols which require
internal types.

## Motivation

Swift allows for use of protocols as types. This allows for interactions with
any object type that conforms, creating a flexible interface that doesn't rely
on inheritance. Unfortunately for protocols with internal types (where the
consumer cares about the internal type) you cannot define a type which specifies
what the internal type should be.

For example, using the following protocol:
```swift
protocol Validator {
  typealias TypeToValidate

  var value: TypeToValidate { get set }
  var valueIfValid: TypeToValidate? { get }
}
```

It would be impossible to define a type which describes a `Validator` which
validates only a specific type. This sort of description is possible for extensions
or generic functions

```swift
extension Validator where TypeToValidate == String {
  //... internally assumes interal type is string
}

func handleValidator<T: Validator where T.TypeToValidate == String>(validator: T) -> T {
  //... internally assumes interal type is string
}
```

Allowing a type which fully describes the useable API of a specifically defined
protocol is consistent with what's available within other parts of the language.

The following example does not compile – this makes sense because `V` is a single
type, meaning we can’t return multiple different types from the function. But
Swift currently lacks a way for a function’s signature to specify that it can
return _any_ type that conforms to `Validator where Validator.ValidatedType == String`

```swift
protocol Validator {
    typealias ValidatedType

    func isValid(object: ValidatedType) -> Bool
}
struct NonEmptyValidator: Validator {
    func isValid(object: String) -> Bool {
        return object.isEmpty
    }
}
func stringValidator<V: Validator where V.ValidatedType == String>() -> V {
    if someCondition {
      return NonEmptyValidator()
    }
    else {
      return SomeDifferentStringValidator()
    }
}
```

## Proposed solution

The simplest solution would be to allow for the definition of a the above
`Validator` with an internal type to be done as such:

```swift
protocol Validator<TypeToValidate> {
  var value: TypeToValidate { get set }
  var valueIfValid: TypeToValidate? { get }
}

struct FooStringValidator: Validator<String> {
  //... implementation
}

let stringValidator: Validator<String>
```

## Detailed design

The protocol generic semantics would be identical to that of generics in the
rest of the language. This includes the ability in the definition of the protocol
to limit the types accepted using inheritance and the `where` clause.

To adapt an example from the swift documentation:

```swift
protocol SequenceComparitor<FooType: SequenceType, BarType: SequenceType where FooType.Generator.Element: Equatable, FooType.Generator.Element == BarType.Generator.Element> {
  var foo: FooType { get }
}

extension SequenceComparitor {
  func anyCommonElements(bar: BarType) -> Bool {
    for fooItem in foo {
        for barItem in bar {
            if fooItem == barItem {
                return true
            }
        }
    }

    return false
  }
}
```

Using this example an object of type `SequenceComparitor<[Int], [Int]>` would
have the following accessible properties and methods:

*Instance Properties*:
- `foo: [Int]`

*Instance Methods*:
- `anyCommonElements(bar: [Int]) -> Bool`

Extension behaviors would also be familiar to those existing.

```swift
protocol SequenceFirstGettable<T: SequenceType> {
  var first: T.Generator.Element { get }
}

extension SequenceFirstGettable where T.Generator.Element == String {
  func firstItemEquals(string: String) -> Bool {
    return first === string
  }
}
```

## Impact on existing code

The primary negative to this approach would be that all existing protocols with
internal types would have to change their syntax to be generic.

For example, this:

```swift
protocol Foo {
  typealias Bar
}
```

would have to become:

```swift
protocol Foo<Bar> {}
```

This change should be easily automatable.

## Alternatives considered

The primary alternate approach I considered was to make protocol definitions
unchanged but allow for type definitions to be something like:

```swift
let var: Type where Type.Internal == OtherType
```

The problem with this is that it breaks the encapsulation of the type with an
abundance of unenclosed spaces. It also does not allow for limitations to be
defined by the protocol definition and is a departure from the way generics
are implemented elsewhere in the language.

## Related reading:

- [Swift Associated Types by Russ Bishop](http://www.russbishop.net/swift-associated-types)
