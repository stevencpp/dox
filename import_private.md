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

Suppose we want to use struct `A1` in a struct `B1` in a module `B` whose ABI should remain stable even if the layout of `A1` changes. We could just `import A;` in `B` and use a pointer to `A1` but suppose we also want to completely isolate users of `B1` from the internals of `A1`, either because of Hyrum's law (even if we make the pointer a private member it may still be observable through reflection or other tricks), because those internals might need to remain secret, because doing so could provide build isolation for faster incremental builds (`B`'s BMI should not depend on `A`'s BMI), or because in this case `A` is slow to parse and we'd like to lower the rebuild latency for debug builds. We could use PIMPL but the problem is that forward declaring `A1` outside of module `A` is not allowed, and we can't convert `A` into a partition of `B` because we're not allowed to change the file that contains `A`. The proposed `import private` directive would allow the forward declaration while making it clear which module `A1` belongs to. At the same time the proposal would allow using `A1`'s definition inside non-inline functions in `B` so that we don't need to repeat the declarations of the members of `B1` in either an implementation unit or a `module :private` section:

```cpp
export module B;

import private { // only the following top-level names from A1 will be imported
    struct A1; // will be an incomplete type if implicitly exported
    struct A3; // ok while parsing, error during codegen - A does not export A3
} from A;

struct A1; // error: can't change which module A1 belongs to

export struct B1 {
    A1 * a; // ok
    A1 a_val; // error, this would require A1's definition to be available to the importer
    [[noinline]] B1() { // note: [[noinline]] is a placeholder for having some way to
    // ^ make class members not inline, whether it's via P1779 or some other method
        a = new A1(); // ok
    }
    [[noinline]] void foo() {
        A2{}; // error, A2 not listed in the import private block
        A1{}.foo(); // ok, note: we didn't list the member "foo", but it's still ok
    }
    [[noinline]] void foo(A1 * a) { a->foo(); } // ok
    A1 & get_a() { return *a; } // ok
    A1 make_a() { return A1{}; } // error
    void use_copy(A1 a) { } // error
    template<typename T> void bar(T t) {
        A1 * a1 = a; // ok
        foo(a1); // ok
        A1{}.foo(); // error, won't work as an incomplete type
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

In many cases parsing should be quite fast compared to codegen, but there will be some situations like module `A` where parsing might be slow as well. With the restrictions shown above on the use of `struct A1` within `B`, forward declaring `struct A1` in the `import private {..}` block should allow extracting `B`'s interface without knowing anything about the the imported module `A`, and so an implementation could choose to extract `B`'s interface in parallel with parsing `big_library` and `A`. The full implementation would no longer be available for inlining but this should further increase parallelism and reduce the rebuild latency for debug builds. In `B`'s BMI `struct A1` would only appear as an incomplete type. The bodies of any functions that use `struct A1`'s defintion would not be included in `B`'s BMI. An implemenation could choose not to include the body of any non-inline functions, making no attempt to parse their bodies, or it could attempt to parse them and not include the ones where that fails due to `struct A1` being odr-used, keeping everything else to improve performance.

There may still be some overhead in extracting the interface compared to just declaring all of the members and defining them in an implementation unit, but if the code grows large enough for that to be an issue, parts of it should probably be refactored into separate helper classes, which could then be placed in separate modules/paritions and imported privately, thus removing the need to parse most of the implementation while extracting the interface.

## Example 2 (circular references between modules):
```cpp
export module A;

import private {
    struct B;
} from B;

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

import private {
    struct A;
} from A;

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

import private {
    namespace AA {
        template<typename T> struct A1; // can only be instantiated in a non-inline context
    }
} from AB;

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

## Example 4 (shallow parsing for BMIs):

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
```

```cpp
export module C;

import private {
    struct A1;
    *
} from A;

import private B;

// everything in A and B can be used by non-inline functions and the module :private section/blocks
// struct A1 can also be used by the interface as an incomplete pointer

module :private { // optional - could've used something like the AB module from the previous example instead
    // things here can only be used by following non-inline functions and module :private section/blocks
    // so this block can be skipped entirely while extracting the interface
    // functions/classes whose interface requires the definitions of types in A/B go here:
    A1 make_a() { return A1{}; }
    struct B1 { A2::type var = 2; };
    // specializations and deduction guides for templates in A/B can only be added here:
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
    A1 * a1; // ok - this was named in the import private block
    void foo(A1 *) {} // same
    
    // the following are errors in both cases
    A1 a1; // error - undeclared identifier / A1's definition must not be accessible to importers
    A3<int> * a3i; // error - undeclared identifier / not named in the import private block
    
    [[noinline]] void bar() {
        // the following are ok while extracting the interface, but become errors when creating the object file:
        fooooo(); // error - typo, undeclared identifier
        foo2("asfd"); // error - wrong number of arguments
        B2<int>::type b2t; // error - not using the local redefinition which has a type member
        
        // the following would be errors while extracting the interface, but are ok when creating the object file
        A2::type a2t; // ok - not using the local redefinition which doesn't have a type member
        
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
// can use everything in A,B, and previous module :private blocks
```

An issue that comes up with privately importing entire modules (either `import private B;` or `import private { ...; * } from A;`) is that if there is a typo in e.g a function call then, without knowing the list of names from the imported module, the compiler cannot correctly diagnose the typo while extracting the interface. So in order to solve this, some typos could be ignored while extracting the interface, and then diagnosed only later while creating the object file. It may seem strange that the declaration for `D::bar` is successfully added to the BMI despite the errors in its body, but it is no different than declaring it in a header file then having errors in the implementation file.

This requires shallow parsing (not throwing an error upon encountering undeclared identifiers) all of the non-inline functions in `D` if there are any private imports of all the names from a module or the entire overload set of a function, but if this is too difficult to implement for some reason, having only private imports of explicitly specified sets of symbols would still be very useful. The implementation could still optimize by including the bodies of functions in the BMI where it does not encounter any undeclared identifiers. Support for omitting template parameters in the import private block also requires shallow parsing of the template arguments i.e accepting any number of template arguments, as long as all of the types/value in the list are known while extracting the interface. Shallow parsing can lead to possibly incorrect BMIs being emitted, e.g with class members having invalid template arguments, or classes/functions/specializations/deduction guides redefined in the interface but these errors will be caught while creating the object file, so the application won't link with invalid code.

When privately importing an entire module, this proposal doesn't allow the use of any types from the imported module (that were not forward declared in an `import private {...}` block) in the interface, not even as incomplete types. They can only be used inside non-inline functions. Doing otherwise would require shallow parsing not only the non-inline functions, but potentially the entire interface. It's hard to say whether that would work. One possible issue is that if the parser encounters an undeclared identifier `X`, it doesn't know if it's a type or a value, and the difference could actually change the interface https://godbolt.org/z/_b_VjG . On the other hand, many common uses of `X` would not be ambiguous, e.g `struct S { X * x; }` cannot be a multiplication, and in `struct S { std::unique_ptr<X> x; }` there is no overload of `unique_ptr` that accepts a value. So maybe this might be ok ? There's another problem if `import private :part1;` could import both exported and not exported names into the interface as incomplete pointers. If we have `export void foo(X *) {}` then we cannot correctly generate the mangled name of `foo` since `X` might have either module or external linkage. If some solutions could be found to these problems (and possibly more) then perhaps the restriction could be lifted.
