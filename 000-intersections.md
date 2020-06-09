# PHP RFC: Intersection types v2
  * Version: 0.1
  * Date: 2020-06-09
  * Author: Gert de Pagter, backendtea@gmail.com
  * Status: Draft
  * First Published at: https://github.com/BackEndTea/intersections
  
## Previous work

The original intersection types RFC was proposed by Levi Morrison here: https://wiki.php.net/rfc/intersection_types.
This RFC expands on that, heavily inspired by https://wiki.php.net/rfc/union_types_v2, by  Nikita Popov.
This RFC largely follows the same format as the union types RFC


## Introduction

An intersection type is like a union type, except instead of requiring the type to be either, it has to be both. 
Intersection types allow for smaller interfaces, and better interopability between userland packages and frameworks.

Currently there is no way to hint a type like this, meaning it is only possible through phpdoc annotations.

## Proposal

Intersection types are specifed using the syntax T1&T2&... and can be used in all positions where types are currently accepted:

```php
class ArrayLike {
    private iterable&ArrayAccess $arrayLike;
 
    public function setArrayLike(iterable&ArrayAccess $number): void {
        $this->arrayLike = $arrayLike;
    }
 
    public function getArrayLike(): iterable&ArrayAccess {
        return $this->arrayLike;
    }
}
```

### Caveats

#### Combination with union types

There are a few way the following type could be interpeted:
```php
function foo(): array|iterable&ArrayAccess {}

// 1
function foo(): (array|iterable)&ArrayAccess {}

// 2
function foo(): array|(iterable&ArrayAccess) {}
```

To reduce the chance of bugs, when a combination of union and intersection types is used, parethesis are required. 

#### Scalar types

A scalar type, or an array, can not be part of an intersection type.
As there is no possible type that could result from that, however a union between a scalar and an intersection is still allowed
```php
function foo(): int&A {} // Disallowed

function foo(): int|(A&B) {} // Allowed

function foo(): int(A&string) {} // Disallowed
```

### Special types

The `void` type can not be part of an intersection type.

The `false` and `null` types are like scalar types, in that they can never produce a valid intersection type,
and are therefore also not allowed.

The following types are allowed, as these types are part of the union, and not the intersection:

```php
function foo(): false|(A&B) {}

function foo(): null|(A&B) {}
```

#### Impossible types

It is possible to create impossible types with intersections, for example with two unrelated concrete classes.
This is not possible to determine until runtime, where executing the code will always trigger a type error.

```php

class Foo {}

class Bar {}

// There is no valid type for $input
function check(A&B $input) {}

check(new Foo); // TypeError
```


### Variance

Intersection types follow the existing variance rules:

* Return types are covariant (child must be subtype).
* Parameter types are contravariant (child must be supertype).
* Property types are invariant (child must be subtype and supertype).

Generally, addition and removal of types can be seen as the inverse of union types.


#### Property types

 Property types are invariant, which means that types must stay the same during inheritance. However, the “same” type may be expressed in different ways.

Intersection types expand the possibilities in this area:

```php
class A {}
class B extends A {}
 
class Test {
    public B $prop;
}
class Test2 extends Test {
    public A&B $prop;
}

class Other {
    public A&B $prop;
}

class Other2 {
    public B $prop;
}
```

In this example, the intersection A&B actually represents the same type as just B, and this inheritance is legal, despite the type not being syntactically the same.

Formally, we arrive at this result as follows: First, B is a subtype of A&B, because it is a subtype of B. Second, B is a subtype of A&B, because B is a subtype of B and B is a subtype of A.


#### Adding and removing intersection types

It is legal to add intersection types in return position and remove intersection types in parameter position:

```php
interface T1{}

interface T2{}

class Test {
    public function param1(T1 $param) {}
    public function param2(T1&T2 $param) {}
 
    public function return1(): T1&T2 {}
    public function return2(): T1 {}
}
 
class Test2 extends Test {
    public function param1(T1&T2 $param) {} // FORBIDDEN: Restricting the param type
    public function param2(T1 $param) {}    // Allowed: Widening param type
 
    public function return1(): T1 {}        // FORBIDDEN: Widening the return type
    public function return2(): T1&T2 {}     // Allowed: Restricting the return type type
}
```

#### Variance of individual union members

Similarly, it is possible to restrict a union member in return position, or widen a union member in parameter position:

```php
interface T1{}
interface T2 extends T1{}
class A {}

 
class Test {
    public function param1(A&T1 $param) {}
    public function param2(A&T2 $param) {}
 
    public function return1(): A&T2 {}
    public function return2(): A&T§ {}
}
 
class Test2 extends Test {
    public function param1(A&T2 $param) {} // Forbidden: Restricting intersection member T1 -> T2
    public function param2(A&T1 $param) {} // Allowed: Widening intersection member T2 -> T1
 
    public function return1(): A&T1 {}     // FORBIDDEN: Widening intersection member T2 -> T1
    public function return2(): A&T2 {}     // Allowed: Restricting intersection member T1 -> T2
}
```

#### Changing between Union and Intersection types

```php
interface T1{}
interface T2{}

class Test {
    public function param1(T1|T2 $param) {}
    public function param2(T1&T2 $param) {}
 
    public function return1(): T1&T2{}
    public function return2(): T1|T2 {}
}
 
class Test2 extends Test {
    public function param1(T1&T2 $param) {} // FORBIDDEN: Restricting the param type
    public function param2(T1|T2 $param) {} // Allowed: Widening param type
 
    public function return1(): T1|T2 {}     // FORBIDDEN: Widening the return type
    public function return2(): T1&T2 {}     // Allowed: Restricting the return type type
}
```

#### Typing mode

As a scalar can not be part of the intersection type, there is no change to type conversion.
Intersection behaviour is not affected by `strict_types`.


#### Reflection

To support intersection types, a new class `ReflectionIntersectionType` is added.
A new `ReflectionCompoundType` interface is added as well, which is also added to the `ReflectionUnionType` class.

```php
interface ReflectionCompoundType {
   /** @return ReflectionType[] */
    public function getTypes(); 
}

class ReflectionIntersectionType extends ReflectionType implements ReflectionCompoundType {
    /* Implemented from ReflectionType */
    /** @return ReflectionType[] */
    public function getTypes();

 
    /* Inherited from ReflectionType */
    /** @return string */
    public function __toString();
}
```

The `getTypes()` method returns an array of ReflectionTypes that are part of the intersection. The types may be returned in an arbitrary order that does not match the original type declaration. A retrieved types may also be `ReflectionCompoundType`.


##### Examples

Examples
```php
// This is one possible output, getTypes() and __toString() could
// also provide the types in the reverse order instead.
function test(): A&B {}
$rt = (new ReflectionFunction('test'))->getReturnType();
var_dump(get_class($rt));    // "ReflectionIntersectionType"
var_dump($rt->getTypes());   // [ReflectionType("A"), ReflectionType("B")]
var_dump((string) $rt);      // "A&B"
 
function test2(): int|(B&C) {}
$rt = (new ReflectionFunction('test2'))->getReturnType();
var_dump(get_class($rt));    // "ReflectionUnionType"
var_dump($rt->allowsNull()); // true
var_dump($rt->getTypes());   // [ReflectionType("int"), ReflectionIntersectionType("B&C")]
var_dump($rt->getTypes()[1]->getTypes) // [ReflectionType("B"), ReflectionType("B")]
var_dump((string) $rt); // "int|(B&C)"
```

## Backward Incompatible Changes
This RFC does not contain any backwards incompatible changes. However, existing ReflectionType based code will have to be adjusted in order to support processing of code that uses intersection types. 

## Proposed PHP Version(s)
This feature is proposed for the next PHP 8.x.

## RFC Impact 

### To SAPIs 
Describe the impact to CLI, Development web server, embedded PHP etc.

### To Existing Extensions
Will existing extensions be affected?

### To Opcache
It is necessary to develop RFC's with opcache in mind, since opcache is a core extension distributed with PHP.

Please explain how you have verified your RFC's compatibility with opcache.



### Open Issues
None atm

## Unaffected PHP Functionality

The `instanceof` operator is not affected by this RFC, and functionality like `$a instanceof Foo&Bar` is not added.

## Future Scope
This section details areas where the feature might be improved in future, but that are not currently proposed in this RFC.

## Proposed Voting Choices
This feature will have a simple Yes/No vote requiring two-thirds in the affirmative. 

## Patches and Tests
Links to any external patches and tests go here.

If there is no patch, make it clear who will create a patch, or whether a volunteer to help with implementation is needed.

Make it clear if the patch is intended to be the final patch, or is just a prototype.

For changes affecting the core language, you should also provide a patch for the language specification.

## Implementation
After the project is implemented, this section should contain 
  - the version(s) it was merged into
  - a link to the git commit(s)
  - a link to the PHP manual entry for the feature
  - a link to the language specification section (if any)

## References 
Links to external references, discussions or RFCs

## Rejected Features
Keep this updated with features that were discussed on the mail lists.
