---
title: Using std::visit in Modern C++ code 
author: Andrei Gorbulin
date: 2023-03-13
styles:
    style: gruvbox-dark
---

# Visitor pattern

**Visitor** is a behavioral design pattern that allows adding new behaviors to existing class hierarchy without altering any existing code.

Object hierarchy:

```
   ShellBase          VisitorBase 
     ↑   ↑                 ↑
ShellA   ShellB       PrintVisitor
```

## Classic Visitor pattern implementation

```cpp
class ShellA;
class ShellB;

class VisitorBase {
public:
    virtual void visitA(ShellA &shell) = 0;
    virtual void visitB(ShellB &shell) = 0;
};

class ShellBase {
public:
    virtual ~ShellBase() {};
    virtual void accept(VisitorBase *visitor) = 0;
};
```

# Visitor pattern (cont)

```cpp
class ShellA : public ShellBase {
public:
    void printHello() {
        std::cout << "Shell A visited.\n";
    }
    void accept(VisitorBase *visitor) override{
        visitor->visitA(*this);
    };
};
class ShellB : public ShellBase {
public:
    void printHello() {
        std::cout << "Shell B visited.\n";
    }
    void accept(VisitorBase *visitor) override{
        visitor->visitB(*this);
    };
};

class PrintVisitor : public VisitorBase {
public:
    void visitA(ShellA &shell) override{
        // Do stuff specific to ShellA.
        shell.printHello();
    };
    void visitB(ShellB &shell) override{
        // Do stuff specific to ShellB.
        shell.printHello();
    };
};
```

# Visitor pattern (cont)

```cpp
int main() {
    std::unique_ptr<ShellBase> shell = std::make_unique<ShellA>();
    auto visitor = PrintVisitor{};

    shell->accept(&visitor);

    return 0;
}
```

Downsides:
1. We only have 2 shells, but need to implement 5 classes;
1. A lot of boilerplate code;
1. Very intrusive, forces Shells to implement `accept()` methods that are not a part of regular Shell interface;
1. Inheritance and virtual functions.

There's got to be a better way...

# Introduction to std::variant
`std::variant` is a type-safe union: it allows us to hold a value from a set of predefined types.
1. Is a C++17 feature;
1. Available to us in Kanzi through `boost::variant2` (very close to `std::variant` implementation);
1. `std::variant` will always be the size of its biggest element;
1. If you want to represent an empty `std::variant`, use `std::monostate`.

## Examples

* Any of the variant types can be assigned to a variant.

```cpp
std::variant<ShellA, ShellB> shellVariant = ShellA{};

// In order to retrieve the value, we need to specify the type.
auto shell = std::get<ShellA>(shellVariant);
```
* In order to avoid copying, we can construct the object in place by using in place constructor. This will call `ShellA(nativeHandle)` constructor under the hood.

```cpp
auto shellVariant = std::variant<ShellA, ShellB>{std::in_place_type_t<ShellA>, nativeHandle};
```

* Default constructible if used with monostate.

```cpp
auto shellVariant = std::variant<std::monostate, ShellA, ShellB>{};

// Here's how to check if variant currently holds specific type.
std::holds_alternative<std::monostate>(shellVariant); // true
```

# std::visit

`std::visit` allows us to perform operations on a `std::variant` without knowing the underlying type.

```cpp
class ShellA {
public:
    void printHello() {
        std::cout << "Shell A visited.\n";
    }
};

class ShellB {
public:
    void printHello() {
        std::cout << "Shell B visited.\n";
    }
};
```

Notice there's no inheritance involved. We are free to implement `ShellA` and `ShellB` however we want.

Also, there's no need to implement `accept()` methods in our Shell classes like we would have to if we'd be using classic Visitor pattern.

# std::visit (cont)

`std::visit` accepts any callable object, so we can create a `Visitor` class and overload `operator()` for each class we would like to use.

```cpp
class Visitor {
public:
    void operator()(const ShellA &shell){
        // Do stuff specific for ShellA.
        shell.printHello();
    };
    void operator()(const ShellB &shell){
        // Do stuff specific for ShellB.
        shell.printHello();
    };
};

int main() {
    std::variant<ShellA, ShellB> shellVariant = ShellB{};

    std::visit(Visitor{}, shellVariant);

    return 0; 
}
```
Compiler will actually force us to implement overloads for each type our variant holds.

# Overload pattern

Same as Visitor object, but using lambdas instead.

```cpp
template<class... Ts> struct overload : Ts... {
    using Ts::operator()...;
};
template<class... Ts> overload(Ts...) -> overload<Ts...>; // Line not needed in C++20...

int main() {
    const auto shellAHandler = [](const ShellA &shell){
        shell.printHello();
    };
    const auto shellBHandler = [](const ShellB &shell){
        shell.printHello();
    };

    ShellVariant shellVariant = ShellB{};

    const auto visitor = overload {shellAHandler, shellBHandler};

    std::visit(visitor, shellVariant);

    return 0; 
}
```

# Using generic lambda

Since we have similar code for each overload, we can leverage it by using generic lambda, which will simplify our code significantly:

```cpp
class ShellA {
public:
    void printHello() {
        std::cout << "Shell A visited.\n";
    }
};

class ShellB {
public:
    void printHello() {
        std::cout << "Shell B visited.\n";
    }
};

int main() {
    std::variant<ShellA, ShellB> shellVariant = ShellB{};

    const auto visitor = [](auto& shell){
        // Works for both ShellA and ShellB.
        shell.printHello();
    };

    std::visit(visitor, shellVariant);

    return 0; 
}
```

However, this is only possible if code for each variant type is exactly the same.

