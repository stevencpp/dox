# import private

## Example 1 (PIMPL)
```cpp
/* license: owned/maintained by someone else, cannot change this file */
export module A;

import big_library; // takes a while to build this module

export {
    struct A1 {
       void foo() { big_library::foo(); }
       static inline int value = 42;
    };
    struct A2 {};
}
```

```cpp
export module B;

import private { // only the following top-level names from A1 will be imported
    struct A1; // will be an incomplete type if implicitly exported
} from A;

export struct B1 {
    A1 * a_ptr; // ok, a_ptr is implicitly exported as an incomplete type
    A1 a_val; // error
    [[noinline]] B1() { // note: [[noinline]] is a placeholder for having some way to
    // ^ make class members not inline, whether it's via P1779 or some other method
        a_ptr = new A1(); // ok
    }
    [[noinline]] void foo() {
        A2{}; // error, A2 not listed in the import private block
        A1{}.foo(); // ok, note: we didn't list the member "foo", but it's still ok
    }
    [[noinline]] void foo(A1 * a) { a->foo(); } // ok
    A1 & get_a() { return *a_ptr; } // ok
    A1 make_a() { return A1{}; } // error
    void use_copy(A1 a) { } // error
    template<typename T> void bar(T t) {
        A1 * a1 = a_ptr; // ok
        foo(a1); // ok
        A1{}.foo(); // error, won't work as an incomplete type
        AA::baz(); // ok
    }
    auto frob() {
        return A1::value; // error - cannot determine the signature of frob without A1's definition
    }
};
```

```cpp
export module C;

import B;

void bar() {
    B1 b;
    b.a_ptr->foo(); // error, foo is inaccessible because A1 is an incomplete type
    auto a = b.get_a(); // ok, note: returns A1 &
    b.foo(&a); // ok
    using A1 = std::remove_reference_t<decltype(a)>;
    A1 aa; // error, can't construct a new A because it's an incomplete type
}
```

A simple implementation of this could choose to build `big_library` first, then `A`, then `B`, then `C`. That would reduce the parallelism and increase the latency in the build compared to only importing big_library from an implementation unit, but it would have the benefit of having the entire implementation available for inlining, which could potentially achieve better runtime performance than just LTO, e.g Clang/GCC can do heap elision https://godbolt.org/z/0zgjJF .

Alternatively, an implementation could choose to extract B's interface in parallel with building big_library and A. With the restrictions shown above on the use of `struct A1` within `B`, naming `struct A1` in the `import private {..}` block should allow extracting `B`'s interface without knowing anything about the the imported module `A`. In `B`'s BMI `struct A1` would only appear as an incomplete type. The bodies of any function that use `struct A1`'s defintion cannot be included in `B`'s BMI. An implemenation could choose not to include the body of any non-inline functions, making no attempt to parse their bodies, or it could attempt to parse them and not include the ones where that fails due to `struct A1` being odr-used, keeping everything else to improve performance.

There may still be some overhead in extracting the interface compared to just declaring all of the members and defining them in an implementation unit, but if the code grows large enough for that to be an issue, parts of it should probably be refactored into separate helper classes, which could then be placed in separate modules/paritions and imported privately, thus removing the need to parse most of the implementation while extracting the interface.

## Example 2 (circular references between modules):
```cpp
export module A;

import private {
    struct B;
} from B;

export struct A {
    B * b_ptr;
    [[noinline]] A() { b_ptr = new B(this); } // ok
    void foo() {}
    [[noinline]] void bar() { 
        b_ptr->foo(); // ok
        b_ptr->a_ptr->foo(); // ok
    }
};
```

```cpp
export module B;

import private {
    struct A;
} from A;

export struct B {
    A * a_ptr;
    B(A * a) : a_ptr(a) {}
    void foo() {}
    [[noinline]] void bar() { 
        a_ptr->foo(); // ok
        a_ptr->b_ptr->foo(); // ok
    }
};
```

```cpp
export module test1;

import A;

void test1() {
    auto a = new A(); // creates a new B as well
    auto b = a->b_ptr;
    a->bar(); // ok, calls b.foo then a.foo
    b->bar(); // error, B is an incomplete type here
    a->b_ptr->foo(); // error, same
    b->a_ptr->foo(); // error, same
}
```

```cpp
export module test2;

import A;
import B;

void test2() {
    auto a = new A(); // creates a new B as well
    auto b = a->b_ptr;
    a->bar(); // ok, calls b.foo then a.foo
    b->bar(); // ok because we've imported B as well, calls a.foo then b.foo 
    a->b_ptr->foo(); // ok because we've imported B as well
    b->a_ptr->foo(); // ok, same
}
```

Here any implemenation needs to separately extract both `A` and `B`'s' interfaces first (those  can be done in parallel) and only then attempt to create object files for them. Of course the bodies of `A::bar` and `B::bar` will not appear in the BMIs for `A` and `B`. 

## Example 3 (functions, templates, specializations, constexpr, CTAD, ADL)

```cpp
export module A;

namespace AA { export {
    template<typename T> struct A1 { A1(T) {} };
    template<> struct A1<char> { };
    template<typename T> A1(T*) -> A1<T>;
    template<typename T> struct A2 {};
    void baz() {}
    void baz(int) {}
    template<typename T> void baz(A1<T>) {}
    template<typename T> void baz(const char *) {}
    constexpr int fortytwo() { return 42; }
}}
```

```cpp
export module B;

import private {
    namespace AA {
        template<typename T> struct A1; // note: char specialization not listed
        constexpr int fortytwo(); // ok but this cannot be used at compile time while extracting the interface
    }
    void AA::baz(); // only these overloads of baz
    template<typename T> void AA::baz(char *) {}
} from A;

namespace AA {
    void baz(int) { } // ok while extracting the interface, but it shadows the baz(int) in A
    // ^ so it'll be an error when creating the object file
    template<> struct A1<double> { }; // error - without shallow parsing
    // ^ we don't know if this specialization already exists in A
    template<typename T> A1(T**) -> A1<T>; // error - without shallow parsing
    // ^ we don't know if this deduction guide already exists in A
}

export {
    void frob() {
        AA::A1<int> a;
        baz(a); // error - can't find the overload with ADL because it was not listed in the import block
    }
    template<int N> struct helper {};
    using namespace AA;
    template<typename T> struct B1 {
        A1<float> * a1_ptr_f; // ok
        A1<char> * a1_ptr_c; // ok, the specialization doesn't need to be listed in import private
        A1<T> * a1_ptr_t; // ok - regardless of T, it'll still be an incomplete type
        A2<float> * a2_ptr_f; // error - not listed in the import private block
        helper< fortytwo() > h; // error - fortytwo not avaible while extracting the interface
        [[noinline]] void foo() { // note: this can be [[noinline]] because it does not depend on T
            baz<T>("fgh"); // error, can't refer to T in a noinline function
            baz(); // ok
            baz<int>("asd"); // ok
            baz(2); // ok, but calls the AA::baz defined in B, not the one defined in A
            // ^ todo: how would name mangling work for this, is it an ODR violation ?
            A1 a11 {3}; // ok, uses CTAD
            A1 a12 {(int*)nullptr}; ok, the deduction guide deduces A1<int>
            helper< fortytwo() > h; // ok - fortytwo available while creating the object file
        }
        void bar() {
            baz<T>("fgh"); // ok, refers to an externally defined function
            A1<int> a1 {3}; // error - A1 must be exported as an incomplete type but
            // ^ bar depends on B1's template parameter so it has to be in the BMI
            helper< fortytwo() > h; // error - fortytwo not available when adding this to the BMI
        }
        
    };
}
```

## Example 4 (shallow parsing for BMIs):

```cpp
export module A;

export {
    struct A1 { void foo() {} };
    struct A2 { void foo() {} };
    void foo_a1() {}
    void foo_a2() {}
}
```

```cpp
export module B;

export {
    struct B1 { void foo() {} };
    template<typename T> struct B2 { };
    void foo_b1() {}
}
```

```cpp
export module C;

namespace CC { export {
    struct C0 {};
    void blarg() {}
    void frob() {}
    void frob(int) {}
    void frob(C0) {}
}}

export {
    template<typename T, typename U> struct C1 {};
    template<typename T> struct C2 { using type = T; };
    template<> struct C2<double> {};
    template<typename T, typename U> C2(T*) -> C2<T, U>;
}
```

```cpp
export module D;

import private {
    struct A1;
    void foo_a1(); // foo_a1 can be used anywhere
    * // all other types/functions can be used as well inside non-inline functions
    // but the other types cannot be implicitly exported
} from A;

import private B; // import everything from B to be used for the implementation
// but only in non-inline functions and the types will not be implicitly exportable

import private {
    struct CC::C0;
    CC::frob; // the entire overload set of CC::frob, but not other functions,
    // CC::frob can only be used within non-inline functions
    template struct C1; // no need to specify the template parameters
    template struct C2;
} from C;

struct B1 {}; // ok while extracting the interface, error while creating the object file
// ^- this struct is already defined in B
template<typename T> struct B2 { using type = float; }; // same

namespace CC {
    void frob(char *); // ok, adds a local overload
    void frob() {} // ok while extracting the interface, error while creating the object file
    // ^- this overload is already defined in C
}
template<> struct C2<char *>{ using T = C2<float>::type; }; // ok - initially the main definition of
// ^ C2 is not available, so the parser needs to completely ignore the body of this until later
template<> struct C2<double> { using type = double; }; // ok while extracting the interface,
// ^ error while creating the object file - this specialization is already defined in C
template<> struct C2<double, float> {}; // same - wrong number of template parameters
template<typename T, typename U> C2(T*) -> C2<T, U>; // same - this deduction guide is already defined in C

export struct D {
    A1 * a1_ptr; // ok, because this was named in the import private block
    A2 * a2_ptr; // error, A2 is undeclared for because it was not
    B1 * b1_ptr; // ok while extracting the interface because B1 was redefined locally
    // ^ error while creating the object file
    B2<int>::type b2t; // same
    C1<int, float> * c1_if_ptr; // ok even though we didn't specify template params in the import private block
    C1< C2<int>::type, char > * c1_ic_ptr; // error - C2's definition not available while extracting the BMI.
    // ^ The template parameter could be ignored, but then how would the importer know it's the same type as 
    // ^ C1<int, char>, to be able to check when passing to e.g void foo(C1<int, char> * c1) ?
    C1<int> * c1_ptr; // ok while extracting the interface, error when creating the object file
    C2<double>::type dbl_val; // error, although there's a local specialization available, we still
    // ^ can't use it while extracting the interface because we don't know yet whether it's a dublicate
    void foo(A1 *) {} // ok
    void foo(A2 *) {} // error, A2 undeclared
    void foo(B1 *) {} // ok while extracting the interface (B1 redefined), error while creating the object file
    void bar() {
        // the following are ok while extracting the interface, but become errors when creating the object file:
        fooooo(); // error - typo, undeclared identifier
        B2<int>::type b2t; // error - not using the local redefinition which has a type member
        CC::frob(2, 3); // error - too many arguments
        CC::blarg(); // error - blarg was not named in the private import block
        // the following would be errors while extracting the interface, but are ok when creating the object file
        B1{}.foo(); // ok - not using the local redefinition which doesn't have a foo member
        // the following are ok in both cases:
        A1{}.foo(); // ok
        foo_a1(); // ok
        A2{}.foo(); // ok
        foo_a2() ; // ok
        foo_b1(); // ok
        CC::frob(); CC::frob(2); // ok
        CC::frob("a"); // ok
        c1_ptr = new C1<int, float> {2}; // ok
        C2<char*> c2cs; // ok, uses local specialization
        C2<double> c2d; // ok, uses specialization in C
        frob(CC::C0{}); // ok, finds void CC::frob(CC::C0) using ADL
    }
};
```

The issue that comes up with privately importing entire modules is that if there is a typo in e.g a function call then, without knowing the list of names from the imported module, the compiler cannot correctly diagnose the typo while extracting the interface. So in order to solve this, some typos could be ignored while extracting the interface, and then diagnosed only later while creating the object file.

A similar issue comes up with importing the overload set of a function without declaring each overload. While extracting the interface the compiler doesn't know if a function call with invalid arguments may become valid if a required overload exists in the implementation module which may not be created/loaded until later.
It may seem strange that the declaration for `D::bar` is successfully added to the BMI despite the errors in its body, but it is no different than declaring it in a header file then having errors in the implementation file.

This requires shallow parsing (not throwing an error upon encountering undeclared identifiers) all of the non-inline functions in `D` if there are any private imports of all the names from a module or the entire overload set of a function, but if this is too difficult to implement for some reason, having only private imports of explicitly specified sets of symbols would still be very useful. The implementation could still optimize by including the bodies of functions in the BMI where it does not encounter any undeclared identifiers. Support for omitting template parameters in the import private block also requires shallow parsing of the template arguments i.e accepting any number of template arguments, as long as all of the types/value in the list are known while extracting the interface. Shallow parsing can lead to possibly incorrect BMIs being emitted, e.g with class members having invalid template arguments, or classes/functions/specializations/deduction guides redefined in the interface but these errors will be caught while creating the object file, so the application won't link with invalid code.
