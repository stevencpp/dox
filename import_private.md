# import private

## Example 1 (PIMPL)
```cpp
/* license: owned/maintained by someone else, cannot change this file */
export module A;

import big_library; // takes a while until the interface of big_library is available
// e.g might be the result of a long chain of imports, might export a ton of templates

export {
    struct A1 {
       void foo() { big_library::foo(); }
       static inline int secret_value = 42;
    };
    struct A2 {
        // even just extracting the interface of A could be slow:
        using T = std::conditional_t< slow_constexpr_func() == 1, int, char>;
        T frob() { return '2'; }
    };
}
struct A3 {};
```

Suppose we want to use struct `A1` in a struct `B1` in a module `B` whose ABI should remain stable even if the layout of `A1` changes. We could just `import A;` in `B` and use a pointer to `A1` but suppose we also want to completely isolate users of `B1` from the internals of `A1`, either because of Hyrum's law (even if we make the pointer a private member it may still be observable through reflection or other tricks), because those internals might need to remain secret (e.g closed source libraries), because doing so could provide build isolation for faster incremental builds (`B`'s BMI should not depend on `A`'s BMI), or because in this case `A` is slow to parse and we'd like to lower the rebuild latency for debug builds. We could use PIMPL but the problem is that forward declaring `A1` outside of module `A` is not allowed, and we can't convert `A` into a partition of `B` because we're not allowed to change the file that contains `A`. A proposed `using module A { struct A1; }` statement would allow the forward declaration while making it clear which module `A1` belongs to, and a proposed `import private A;` statement would allow using `A1` and everything else in `A` inside non-inline functions in `B` so that we don't need to repeat the declarations of the members of `B1` in either an implementation unit or a `module :private` section:

```cpp
export module B;

import private A; // allow using everything from A in non-inline functions

using module A { // forward declare the following symbols from A to use them in the interface as well
    struct A1; // will be an incomplete type if implicitly exported
    struct A3; // ok while parsing, error during codegen - A does not export A3
}

export struct B1 {
    A1 * a; // ok
    A1 a_val; // error, this would require A1's definition to be available to the importer
    A2 * a2; // error, A2 wasn't forward declared
    [[noinline]] B1() { // note: [[noinline]] is a placeholder for having some way to
    // ^ make class members not inline, whether it's via P1779 or some other method
        a = new A1(); // ok
    }
    [[noinline]] void foo() {
        A1{}.foo(); // ok
        A2{}.frob(); // ok - wasn't forward declared, still ok
    }
    [[noinline]] void foo(A1 * a) { a->foo(); } // ok
    A1 & get_a() { return *a; } // ok
    A1 make_a() { return A1{}; } // error
    void use_copy(A1 a) { } // error
    template<typename T> void bar(T t) {
        A1 * a1 = a; // ok
        foo(a1); // ok
        a1->foo(); // error, won't work as an incomplete type
        A2 * a2 = nullptr; // error, A2 wasn't forward declared
    }
    auto frob() {
        return A1::secret_value; // error - cannot determine the signature of frob without A1's definition
    }
};
```

```cpp
export module C;

import B;

void bar() {
    B1 b;
    b.a->foo(); // error, foo is inaccessible because A1 is an incomplete type
    auto a = b.get_a(); // ok, note: returns A1 &
    b.foo(&a); // ok
    using A1 = std::remove_reference_t<decltype(a)>;
    A1 aa; // error, can't construct a new A1 because it's an incomplete type
    int v = A1::secret_value; // error, A1 is an incomplete type
}
```

A simple implementation of this could choose to compile `big_library` first, then `A`, then `B`, then `C`. A more efficient solution could use two-step compilation where it would e.g extract the interface of `A` in parallel with generating the object file for `big_library`, do the parsing for `B` in parallel with the codegen for `A` etc. Since there are no separate implementation units, in either case for release builds the implementation could choose to include everything in the BMIs for each module, making the entire implementation available for inlining, which could potentially achieve better runtime performance than just LTO, e.g Clang/GCC can do heap elision https://godbolt.org/z/0zgjJF to remove the allocation and indirection overhead of PIMPL.

In many cases parsing should be quite fast compared to codegen, but there will be some situations like module `A` where parsing might be slow as well. With the restrictions shown above on the use of `struct A1` within `B`, forward declaring `struct A1` in the `using module A {..}` block should allow extracting `B`'s interface without knowing anything about the imported module `A`, and so an implementation could choose to extract `B`'s interface in parallel with parsing `big_library` and `A`. The full implementation would no longer be available for inlining but this should further increase parallelism and reduce the rebuild latency for debug builds. In `B`'s BMI `struct A1` would only appear as an incomplete type. The bodies of any functions that use `struct A1`'s defintion would not be included in `B`'s BMI. An implementation could choose not to include the body of any non-inline functions, making no attempt to parse their bodies, or it could attempt to parse them and not include the ones where that fails due to `struct A1` being odr-used, keeping everything else to improve performance.

There may still be some overhead in extracting the interface compared to just declaring all of the members and defining them in an implementation unit, but if the code grows large enough for that to be an issue, parts of it should probably be refactored into separate helper classes, which could then be placed in separate modules/paritions and imported privately, thus removing the need to parse most of the implementation while extracting the interface.

## Example 2 (circular references between modules):
```cpp
export module forwards;

export using module A { struct A; }
export using module B { string B; }
```

```cpp
export module A;

import forwards;
import private B;

export struct A {
    B * b;
    [[noinline]] A() { b = new B(this); } // ok
    void foo() {}
    [[noinline]] void bar() {
        b->foo(); // ok
        b->a->foo(); // ok
    }
};
```

```cpp
export module B;

import forwards;
import private A;

export struct B {
    A * a;
    B(A * a) : a(a) {}
    void foo() {}
    [[noinline]] void bar() {
        a->foo(); // ok
        a->b->foo(); // ok
    }
};
```

```cpp
export module test1;

import A;

void test1() {
    auto a = new A(); // creates a new B as well
    auto b = a->b;
    a->bar(); // ok, calls b.foo then a.foo
    b->bar(); // error, B is an incomplete type here
    a->b->foo(); // error, same
    b->a->foo(); // error, same
}
```

```cpp
export module test2;

import A;
import B;

void test2() {
    auto a = new A(); // creates a new B as well
    auto b = a->b;
    a->bar(); // ok, calls b.foo then a.foo
    b->bar(); // ok because we've imported B as well, calls a.foo then b.foo 
    a->b->foo(); // ok because we've imported B as well
    b->a->foo(); // ok, same
}
```

Here any implemenation needs to separately extract both `A` and `B`'s' interfaces first (those  can be done in parallel) and only then attempt to create object files for them. Of course the bodies of `A::bar` and `B::bar` will not appear in the BMIs for `A` and `B`. Note that in order to support this use case, the compiler and the build system is required to be able to do two-step compilation. Scanners would need to separately report the possibly different sets of imports for the interface and implementation of any given source file.

## Example 3 (class templates)

```cpp
export module A;

namespace AA { export {
    template<typename T> struct A1 { using type = T; };
}}
```

```cpp
export module AB;

export import A;

using namespace AA;
export { // adding specializations and deduction guides to A1, to be used by B's impl
    template<> struct A1<int> {};
    template<typename T> A1(T*) -> A1<T>;
}
```

```cpp
export module B;

import private AB;

using module AB {
    namespace AA {
        template<typename T> struct A1; // can only be instantiated in a non-inline context
    }
}

// cannot add any further specializations / deduction guides here for A1

using namespace AA;

export {
    template<typename T>
    struct B1 {
        A1<float> * a1_f; // ok
        A1<T> * a1_t; // ok - regardless of T, it'll still be an incomplete type
        A1<float>::type * a1ft; // error - cannot instantiate A1<float> here
        
        [[noinline]] void foo() { // note: this can be [[noinline]] because it does not depend on T
            a1_f = new A1<float>(); // ok
        }
        void bar(T*) { // note: this function must be inline because it depends on T
            a1_t = new A1<T>(); // error: A1<T> is an incomplete type
        }
    };
}
```

## Example 4 (shallow parsing for non-inline functions):

```cpp
export module A;

export {
    struct A1 { void foo() {} };
    struct A2 { using type = int; };
    template<typename T> A3 { A3(T) {} };
    void foo() {};
    void foo2() {};
}
```

```cpp
export module B;

import private A; // everything in A can be used by non-inline functions and the module :private section/blocks

using module A {
    struct A1; // this can also be used by the interface as an incomplete pointer
}

module :private { // optional - could've used something like the AB module from the previous example instead
    // things here can only be used by following non-inline functions and module :private section/blocks
    // so this block can be skipped entirely while extracting the interface
    // functions/classes whose interface requires the definitions of types in A go here:
    A1 make_a() { return A1{}; }
    struct B1 { A2::type var = 2; };
    // specializations and deduction guides for templates in A can only be added here:
    ...
}

// the following are ok while extracting the interface, error while creating the object file
void foo() {} // error - already defined in A
struct A2 {}; // error - already defined in A
struct B1 {}; // error - already defined in the previous module :private block

export struct B2 {
    // the following are ok while extracting the interface, but become errors when creating the object file:
    B1 * b1; // defined in the interface, but also defined in the module :private block
    B1 b1; // same
    void foo(B1 *) {} // same
    A2 * a2; // same - also defined in A
    
    // the following are ok in both cases
    A1 * a1; // ok - this was forward declared in the using module block
    void foo(A1 *) {} // same
    
    // the following are errors in both cases
    A1 a1; // error - undeclared identifier / A1's definition must not be accessible to importers
    A3<int> * a3i; // error - undeclared identifier / not named in the using module block
    
    [[noinline]] void bar() {
        // the following are ok while extracting the interface, but become errors when creating the object file:
        fooooo(); // error - typo, undeclared identifier
        foo2("asfd"); // error - wrong number of arguments
        B2<int>::type b2t; // error - not using the local redefinition which has a type member
        
        // the following would be errors while extracting the interface, but are ok when creating the object file
        A2::type a2t; // ok - not using the local redefinition which doesn't have a type member
        int var = B1{}.var; // ok - using B1 from the module :private block
        
        // the following are ok in both cases:
        make_a().foo(); // ok
        foo2(); // ok
        a3i = new A3{2}; // ok
        A2{}; // ok - both definitions just happen to have a default constructor
        
        // the following are errors in both cases
        B1{}.foo(); // error - neither version of B1 has a foo member
        A2:::type x; // random syntax errors
    }
};

module :private;
// can use everything in A and previous module :private blocks
```

An issue that comes up with `import private A;` is that if there is a typo in e.g a function call then, without knowing the list of names from the imported module, the compiler cannot correctly diagnose the typo while extracting the interface. So in order to solve this, some typos need to be ignored while extracting the interface, and then diagnosed only later while creating the object file. This requires shallow parsing (not throwing an error upon encountering undeclared identifiers) all of the non-inline functions in `B`. If the parser finds an undeclared identifier in a non-inline function it could just skip the rest of the function body. It may seem strange that the declaration for `B2::bar` may be successfully added to the BMI despite the errors in its body, but it is no different than declaring it in a header file then having errors in the implementation file. Shallow parsing can lead to possibly incorrect BMIs being emitted, e.g with classes/functions/specializations/deduction guides redefined in the interface but these errors will be caught while creating the object file, so the application won't link with invalid code.

# Example 5 (shallow parsing for incomplete types)

```cpp
export module A;

import private B, C; // still allows using everything from from B and C in non-inline functions, but here it
// ^ also allows using types from B and C in the interface as incomplete types without forward declarations

export struct X {
    Y * y; Z * z; // ok
    Y y; Z z; // error
    [[noinline]] void foo(Y *, Z *) { // ok
        y = new Y; z = new Z; // ok
    }
};
```

```cpp
export module B;

import private A, C;

export struct Y {
    X * x; Z * z; // ok
    X x; Z z; // error
    [[noinline]] void foo(X *, Z *) { // ok
        x = new X; z = new Z; // ok
    }
};
```

```cpp
export module C;

import private A, B;

export struct Z {
    X * x; Y * y; // ok
    X x; Y y; // error
    [[noinline]] void foo(X *, Y*) { // ok
        x = new X; y = new Y; // ok
    }
};
```

In order to support circular references between modules without requiring forward declarations, a kind of shallow parsing is required not only in non-inline methods, but in the interface as well. E.g while extracting the interface of module `A`, when the parser encounters the undeclared identifier `Y`, privately importing variables/concepts/namespaces/aliases into `A`'s interface (i.e outside of non-inline functions) is not allowed, so there are only two possibilities. Either the code is correct and there exists a type `Y` in one of the modules, or it's a typo, some kind of user error. The parser can proceed assuming it is not a typo, as this assumption will be checked later while creating the object file for `A`. The BMI for `A` will record that `A` exports a method `void X::foo(Y*,Z*)` but the shallow parsing cannot tell which modules `Y` and `Z` are attached to, so it is not possible to generate the full mangled name of `X::foo` using the information in `A`'s BMI alone. But after the BMIs are extracted for `B` and `C` as well, when creating the object file for `A` the sets of symbols that `B` and `C` export will be available to the compiler, so at that point it should be able to find that `B` exports `Y` and `C` exports `Z`, allowing it to write the correct full mangled name for `X::foo` into the object file for `A`.

// TODO: needs more work
