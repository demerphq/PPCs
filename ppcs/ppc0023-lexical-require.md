# Lexically require modules

## Preamble

    Author:  Ovid + Yves Orton
    Sponsor:
    ID:      23 
    Status:  Draft

## Abstract

When writing a module, the `use` and `require` statements make the modules
available globally. This leads to strange bugs where you can write
`my $object = Some::Class->new` and have it work, even if you didn't
require that module. This transitive dependency is fragile and will break
if the code requiring `Some::Class` decides to no longer require it.

This PPC proposes:

```perl
package Foo;
use feature 'lexical_require`;
use Some::Module;
```

Within a given lexical scope, **ß**, if the 'lexical_require' feature is used,
code inside of scope **ß** cannot call methods against class names that have
not been explicitly required within the current package, and doing so would
throw an exception. Methods would be allowed against any object (blessed
reference), but not against a class.

## Motivation

* While sometimes it is a deliberate design choice to use transitive
  dependencies to access code in packages that are not directly required
  by the code using them doing so accidentally is a form of action at a
  distance that may result in run time breakage if the code which does
  the require is changed.
* Transitive dependencies are even more fragile if the code is
  conditionally required:

```perl
if ($true) {
    require Some::Module;
}
```

In the above, the transitive dependency can fail if `$true` is never true.

The initial discussion is on [the P5P mailing
list](https://www.nntp.perl.org/group/perl.perl5.porters/2023/07/msg266678.html).

## Rationale

* The new syntax ensures that the module author can enable the feature in
a block of code and be sure that it does not exploit transitively required
modules.

## Specification

For the given lexical scope—block or file—`use feature 'lexical_require'`
will cause code using a class based method call to validate that the package
containing the call directly required that class. Any other use will cause an
exception.

```perl
package Foo {
    use feature 'lexical_require';
    use Hash::Ordered;
    my $hash_ordered = Hash::Ordered->new(); # this is ok
    my $other_thing = Other::Thing->new(); # not ok, throw an exception.
    no feature 'lexical_require';
    my $another_thing = Other::Thing->new(); # this is ok.
}
```

Note that, if possible, this should also apply to package variables and also
subroutine calls. For instance:

```perl
package Foo {
    use feature 'lexical_require';
    use Some::Package;
    my $whatever = Some::Package::whatever(); # this is ok
    my $result = Other::Thing::compute(); # not ok, throw an exception.
}
```

## Backwards Compatibility

There should be no backwards compatibility issues as the only code affected
by this proposal would have to explicitly use it.

## Security Implications

If anything, this might improve security by not allowing code to have an
accidental dependency on code it doesn't explicitly use.

## Implementation Difficulties

When code does:

    require Foo;

there is no guarantee that it actually populates the Foo package. For
instance it is not unusual for a package like `Foo::Heavy` to populate
the `Foo` package and for there not to be a `Foo::Heavy` package at all.
However for this proposal to work we would have to assume that the
package being required was the name being required. This is not
anticipated to be a problem as the feature is opt-in, if a dev wants to
use code patterns that could be problematic then they would simply not
use the feature.

## Performance Implications

We would have to test for the feature on every class based method call.
Whether we could localize those checks to compile time is an open
question. Probably not.

## Memory Implications

We would have to track which packages require which other packages,
which would implications on how much memory is used. Unfortunately this
would occur even for code that does not use the feature.

## Prototype Implementation

None.

## Future Scope

In the future, it might be nice to have `namespace` declarations.

```perl
namespace Las::Vegas;
package ::Casino; # package Las::Vegas::Casino

```

For the above, what happens in `Las::Vegas` stays in `Las::Vegas`.

The above allows you to declare a new namespace and everything within that
namespace is private to it. Only code that is officially exposed can be used.
Companies can have teams using separate namespaces and only official APIs can
be accessed. You can't "reach in" and override subroutines. This would require
developers to plan their designs more carefully, including a better
understanding of dependency injection and building flexible interfaces.

## Copyright

Copyright (C) 2023, Curtis "Ovid" Poe
Copyright (C) 2023, Yves Orton

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
