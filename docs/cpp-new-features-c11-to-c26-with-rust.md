# C++ NEW FEATURES (C++11 to C++26)

**Complete Guide: Modern C++ Additions with Before/After Examples and Rust Equivalents**

---

## Table of Contents

1. Smart Pointers (unique_ptr, shared_ptr, weak_ptr)
2. Move Semantics and Rvalue References
3. Lambda Functions and Closures
4. Auto Type Deduction
5. Range-based For Loops
6. Variadic Templates
7. Constexpr and Compile-time Computation
8. Structured Bindings
9. Optional and Variant Types
10. Concepts and Constraints
11. Coroutines
12. Modules

---

## PART 1: SMART POINTERS (C++11)

### What Problem Did It Solve?

**Before C++11 (Manual Memory Management):**
```cpp
// Dangerous: Manual memory management
void process() {
    MyClass* obj = new MyClass();
    obj->doSomething();
    delete obj;  // Easy to forget or double-delete
    
    // If exception occurs, memory leaks!
    if (someCondition) {
        delete obj;  // May be deleted twice
        throw std::runtime_error("Error!");
    }
}
```

**Problems:**
- Memory leaks if you forget `delete`
- Memory leaks if exception thrown
- Use-after-free if you delete twice
- Confusing ownership semantics

### C++11 Solution: Smart Pointers

```cpp
// unique_ptr - Single ownership
std::unique_ptr<MyClass> obj(new MyClass());
obj->doSomething();
// Automatically freed when obj goes out of scope
// No manual delete needed!

// Make it safer with make_unique
auto obj2 = std::make_unique<MyClass>();

// Move ownership
std::unique_ptr<MyClass> obj3 = std::move(obj);
// obj is now nullptr, obj3 owns the memory

// shared_ptr - Shared ownership
std::shared_ptr<MyClass> shared1 = std::make_shared<MyClass>();
std::shared_ptr<MyClass> shared2 = shared1;
// Memory freed when LAST shared_ptr is destroyed
```

### Rust Equivalent

```rust
// Box - Single ownership (like unique_ptr)
let mut obj = Box::new(MyClass::new());
obj.do_something();
// Automatically freed when obj goes out of scope

// Move ownership
let obj2 = obj;  // obj is moved, no longer accessible
// obj is no longer valid - compiler prevents use-after-free!

// Shared ownership (like shared_ptr)
let shared1 = Rc::new(MyClass::new());
let shared2 = shared1.clone();
// Freed when all Rc are dropped

// Thread-safe shared ownership
let shared3 = Arc::new(MyClass::new());
let shared4 = shared3.clone();
// Arc = Atomic Reference Counting
```

### Comparison Table

| Feature | Manual (Before) | unique_ptr (C++11) | shared_ptr (C++11) | Rust Box | Rust Rc | Rust Arc |
|---------|---|---|---|---|---|---|
| **Ownership** | Manual | Single | Multiple | Single | Multiple (single-threaded) | Multiple (thread-safe) |
| **Safety** | ❌ Manual | ✅ Compiler enforced | ✅ Compiler enforced | ✅ Compiler enforced | ✅ Compiler enforced | ✅ Compiler enforced |
| **Overhead** | None | None | Reference count | None | Reference count | Atomic ref count |
| **Memory Leaks** | ❌ Possible | ✅ Prevented | ✅ Prevented | ✅ Prevented | ✅ Prevented | ✅ Prevented |
| **Use-after-free** | ❌ Possible | ✅ Prevented | ✅ Prevented | ✅ Prevented | ✅ Prevented | ✅ Prevented |

---

## PART 2: MOVE SEMANTICS AND RVALUE REFERENCES (C++11)

### What Problem Did It Solve?

**Before C++11 (Expensive Copies):**
```cpp
class Vector {
public:
    Vector() : data(nullptr), size(0) {}
    
    // Copy constructor - expensive!
    Vector(const Vector& other) {
        data = new int[other.size];
        std::copy(other.data, other.data + other.size, data);
        size = other.size;
    }
    
private:
    int* data;
    size_t size;
};

Vector create_vector() {
    Vector v;
    v.push_back(1);
    v.push_back(2);
    return v;  // COPY! Expensive!
}

Vector v = create_vector();  // Another COPY! Very expensive!
```

### C++11 Solution: Move Semantics

```cpp
class Vector {
public:
    // Move constructor - cheap!
    Vector(Vector&& other) noexcept {
        data = other.data;      // Steal pointer
        size = other.size;      // Steal size
        other.data = nullptr;   // Leave other empty
        other.size = 0;
    }
    
    // Move assignment
    Vector& operator=(Vector&& other) noexcept {
        if (this != &other) {
            delete[] data;
            data = other.data;
            size = other.size;
            other.data = nullptr;
            other.size = 0;
        }
        return *this;
    }
    
private:
    int* data;
    size_t size;
};

Vector v = create_vector();  // MOVE! Cheap! Just steal data
```

### Rvalue References Explained

```cpp
// Lvalue reference - reference to existing object
int& lvalue_ref = x;

// Rvalue reference - reference to temporary
int&& rvalue_ref = 42;  // 42 is temporary

// Use in function
void process(const int& ref) {
    // Takes lvalue reference (const)
}

void process(int&& ref) {
    // Takes rvalue reference (temporary)
    // Can steal resources!
}

int x = 5;
process(x);     // Calls process(const int&)
process(10);    // Calls process(int&&) - can move!
```

### Rust Equivalent

```rust
// Move by default - no copying!
fn create_vector() -> Vec<i32> {
    let mut v = Vec::new();
    v.push(1);
    v.push(2);
    v  // Returns v
}

let v = create_vector();  // MOVE! Cheap!
// v now owns the vector

// Rust has built-in move semantics
fn take_ownership(vec: Vec<i32>) {
    // vec is now owned by this function
    println!("{:?}", vec);
    // Automatically dropped here
}

let v = vec![1, 2, 3];
take_ownership(v);  // v is MOVED
// println!("{:?}", v);  // ERROR! v no longer accessible

// Borrow instead of moving
fn borrow(vec: &Vec<i32>) {
    // vec is borrowed, not moved
    println!("{:?}", vec);
}

let v = vec![1, 2, 3];
borrow(&v);        // Borrow
println!("{:?}", v);  // Still accessible!
```

### Comparison

| Aspect | Before C++11 | C++11 Move | Rust Default |
|--------|---|---|---|
| **Default behavior** | Copy | Still copy (for compatibility) | Move |
| **Explicit move** | N/A | `std::move()` | Implicit/enforced |
| **Performance** | Slow (copies) | Fast (moves) | Fast (moves) |
| **Safety** | Manual | Better | Built-in |
| **Syntax clarity** | OK | ⚠️ Requires std::move | ✅ Clear |

---

## PART 3: LAMBDA FUNCTIONS (C++11)

### Before C++11 (Function Pointers and Functors)

```cpp
// Function pointer - no capture
void sort_by_value(std::vector<int>& v) {
    std::sort(v.begin(), v.end(), compare_func);
}

int compare_func(int a, int b) {
    return a < b;
}

// Functor - can capture, but verbose
class Comparator {
private:
    int threshold;
public:
    Comparator(int t) : threshold(t) {}
    bool operator()(int a, int b) const {
        return (a - threshold) < (b - threshold);
    }
};

std::vector<int> v = {1, 2, 3};
std::sort(v.begin(), v.end(), Comparator(10));  // Verbose!
```

### C++11 Solution: Lambda Functions

```cpp
// Simple lambda
auto simple = []() { return 42; };

// Lambda with parameters
auto add = [](int a, int b) { return a + b; };

// Lambda with capture (closure)
int threshold = 10;
auto compare = [threshold](int a, int b) {
    return (a - threshold) < (b - threshold);
};

// Use in algorithms
std::vector<int> v = {1, 2, 3};
std::sort(v.begin(), v.end(), [threshold](int a, int b) {
    return (a - threshold) < (b - threshold);
});

// Different capture modes
int x = 5;

[x]() { }              // Capture x by value
[&x]() { }             // Capture x by reference
[x, &y]() { }          // Capture x by value, y by reference
[=]() { }              // Capture all by value
[&]() { }              // Capture all by reference
[this]() { }           // Capture this pointer (for member functions)
```

### C++14+ Features

```cpp
// C++14: Auto return type
auto lambda = [](auto x, auto y) { return x + y; };

// C++20: Template lambdas (simplified syntax)
auto generic = []<typename T>(T a, T b) {
    return a + b;
};
```

### Rust Equivalent

```rust
// Simple closure
let simple = || 42;

// Closure with parameters
let add = |a: i32, b: i32| a + b;

// Closure with capture
let threshold = 10;
let compare = |a: i32, b: i32| {
    (a - threshold) < (b - threshold)
};

// Use in iterators
let v = vec![1, 2, 3];
v.iter()
    .filter(|&x| x > &threshold)
    .map(|x| x * 2)
    .collect::<Vec<_>>();

// Three capture modes (automatic)
let x = 5;

let f = || x + 1;              // Fn - borrows immutably
let f = || x = x + 1;          // FnMut - borrows mutably
let f = || drop(x);            // FnOnce - takes ownership

// Traits instead of types
fn apply<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 {
    f(x)
}

apply(|x| x * 2, 5);
apply(|x| x + 1, 5);
```

### Comparison

| Feature | Function Pointer | Functor | C++ Lambda | Rust Closure |
|---------|---|---|---|---|
| **Capture** | ❌ No | ✅ Yes (verbose) | ✅ Yes (automatic) | ✅ Yes (automatic) |
| **Type** | Single type | Unique type | Unique type | Trait (Fn/FnMut/FnOnce) |
| **Syntax** | `void (*)()` | Class with `operator()` | `[]() {}` | `\|\| {}` |
| **Composition** | Limited | Awkward | Good | Excellent |
| **Trait system** | No | No | No | ✅ Yes |

---

## PART 4: AUTO TYPE DEDUCTION (C++11)

### Before C++11 (Verbose Type Names)

```cpp
// Tedious type names
std::map<std::string, std::vector<int>>::iterator it = m.begin();

// Iterator loops
std::vector<std::pair<int, std::string>> data;
for (std::vector<std::pair<int, std::string>>::iterator 
     it = data.begin(); 
     it != data.end(); 
     ++it) {
    // Use it
}
```

### C++11 Solution: Auto Type Deduction

```cpp
// Simple auto
auto x = 42;           // int
auto y = 3.14;         // double
auto z = "hello";      // const char*

// Complex types
auto it = m.begin();   // Iterator type deduced

// Range-based for (also C++11 feature)
for (auto& item : data) {
    // Type of item deduced
}

// Function return types (C++14)
auto add(int a, int b) {
    return a + b;  // Return type deduced
}

// Lambda return types (C++14)
auto lambda = [](int x) { return x * 2; };  // Return type deduced
```

### C++17+: Structured Bindings

```cpp
// C++17: Unpack data
std::pair<int, std::string> p = {1, "hello"};
auto [x, y] = p;  // x = 1, y = "hello"

std::tuple<int, double, std::string> t = {42, 3.14, "world"};
auto [a, b, c] = t;

// Works with structs
struct Point { int x; int y; };
Point p = {10, 20};
auto [px, py] = p;

// Ignore values
auto [first, second, ...] = t;  // ... is placeholder in some cases
```

### Rust Equivalent

```rust
// Type inference built-in
let x = 42;           // i32 inferred
let y = 3.14;         // f64 inferred

// Can be explicit
let x: i32 = 42;      // Type annotation

// Destructuring
let (a, b) = (1, "hello");
let (x, y) = (10, 20);

// Ignore with _
let (a, _) = (1, 2);

// In iterators
for item in &data {
    // Type inferred
}

// In functions (no auto, but inference works)
fn add(a: i32, b: i32) -> i32 {
    a + b  // Type must be specified in signature
}

// Closures (type inference)
let add = |a, b| a + b;  // Types inferred from context
```

### Comparison

| Feature | C++ Before | C++ auto | C++17 Structured | Rust |
|---------|---|---|---|---|
| **Type inference** | ❌ Manual | ✅ Limited | ✅ Good | ✅ Excellent |
| **Verbosity** | High | Medium | Low | Low |
| **Clarity** | Clear | Good | Excellent | Excellent |
| **Function returns** | Manual | C++14+ | N/A | Type annotations required |

---

## PART 5: RANGE-BASED FOR LOOPS (C++11)

### Before C++11

```cpp
// Traditional for loop
std::vector<int> v = {1, 2, 3, 4, 5};
for (int i = 0; i < v.size(); ++i) {
    std::cout << v[i] << "\n";
}

// Iterator loop
for (std::vector<int>::iterator it = v.begin(); it != v.end(); ++it) {
    std::cout << *it << "\n";
}

// Works with containers but tedious
```

### C++11 Solution: Range-Based For Loop

```cpp
// Clean syntax
std::vector<int> v = {1, 2, 3, 4, 5};
for (int x : v) {
    std::cout << x << "\n";
}

// Reference to avoid copy
for (const auto& x : v) {
    std::cout << x << "\n";
}

// Mutable reference
for (auto& x : v) {
    x *= 2;
}

// Works with any container
std::map<std::string, int> m = {{"a", 1}, {"b", 2}};
for (const auto& [key, value] : m) {  // C++17 structured binding
    std::cout << key << ": " << value << "\n";
}
```

### Rust Equivalent

```rust
// Iterator (preferred)
let v = vec![1, 2, 3, 4, 5];
for x in &v {
    println!("{}", x);
}

// Mutable iterator
for x in &mut v {
    *x *= 2;
}

// Consuming iterator
for x in v {
    println!("{}", x);  // v is moved/consumed
}

// With destructuring
let map = vec![("a", 1), ("b", 2)];
for (key, value) in &map {
    println!("{}: {}", key, value);
}

// Iterator chain (Rust style)
v.iter()
    .map(|x| x * 2)
    .filter(|x| x > &5)
    .for_each(|x| println!("{}", x));
```

### Comparison

| Feature | C++ Before | C++11 Range-based | C++17+ Structured | Rust Iterators |
|---------|---|---|---|---|
| **Syntax clarity** | Poor | Excellent | Perfect | Perfect |
| **Composable** | ❌ No | Limited | Limited | ✅ Excellent |
| **Lazy evaluation** | N/A | No | No | ✅ Yes |
| **Idiomatic** | No | Yes | Yes | Yes |

---

## PART 6: VARIADIC TEMPLATES (C++11)

### What It Enables

**Before C++11 (Function Overloading):**
```cpp
// Multiple versions for different argument counts
void print() { }
void print(int a) { std::cout << a << "\n"; }
void print(int a, int b) { std::cout << a << ", " << b << "\n"; }
void print(int a, int b, int c) { std::cout << a << ", " << b << ", " << c << "\n"; }
// Scales poorly!
```

### C++11 Solution: Variadic Templates

```cpp
// Base case
template<>
void print() { }

// Recursive case
template<typename T, typename... Args>
void print(T first, Args... rest) {
    std::cout << first << ", ";
    print(rest...);
}

print(1, 2, 3, 4, 5);  // Works for any number of arguments!

// Fold expressions (C++17)
template<typename... Args>
int sum(Args... args) {
    return (args + ... + 0);  // Fold left
}

sum(1, 2, 3, 4, 5);  // 15
```

### Rust Equivalent

```rust
// Rust doesn't have variadic templates like C++
// Instead uses macros or trait objects

// Macro approach
macro_rules! print_all {
    () => {};
    ($first:expr) => {
        println!("{}", $first);
    };
    ($first:expr, $($rest:expr),+) => {
        print!("{}, ", $first);
        print_all!($($rest),+);
    };
}

print_all!(1, 2, 3, 4, 5);

// Generic function (works with iterators)
fn sum<I: Iterator<Item = i32>>(iter: I) -> i32 {
    iter.sum()
}

sum(vec![1, 2, 3, 4, 5].into_iter());

// Modern: Variadic generics (RFC accepted, not stable yet)
// Will allow similar syntax to C++ in future
```

---

## PART 7: CONSTEXPR (C++11, Enhanced in C++17, C++20)

### What It Enables

**Before C++11 (Runtime Computation):**
```cpp
// Must run at runtime
int factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}

int result = factorial(5);  // Computed at runtime, then stored
```

### C++11+ Solution: constexpr

```cpp
// C++11: Compile-time computation
constexpr int factorial(int n) {
    return (n <= 1) ? 1 : n * factorial(n - 1);
}

constexpr int result = factorial(5);  // Computed at COMPILE TIME!
// result is known at compile time, no runtime cost

// C++17: if constexpr
template<typename T>
void process(T value) {
    if constexpr (std::is_integral_v<T>) {
        // This code only instantiated for integral types
        std::cout << "Integer: " << value << "\n";
    } else {
        std::cout << "Other type\n";
    }
}

// C++20: consteval - MUST be compile-time
consteval int compile_time_only(int n) {
    return n * 2;
}

// compile_time_only(5);  // OK - computed at compile time
// int x = 5;
// compile_time_only(x);  // ERROR - x not compile-time constant
```

### Rust Equivalent

```rust
// const - compile-time constant
const FIVE: i32 = 5;
const FACTORIAL_5: i32 = 120;  // Must be literal or const function

// const fn - can be used in const context
const fn factorial(n: i32) -> i32 {
    if n <= 1 { 1 } else { n * factorial(n - 1) }
}

const RESULT: i32 = factorial(5);  // Computed at compile time

// const generics (unstable, nightly only)
#[cfg(feature = "nightly")]
struct Array<const N: usize> {
    data: [i32; N]  // Size known at compile time
}

// Can use in match expressions
match value {
    x if x == const_fn() => { }  // Compile-time evaluated
    _ => { }
}
```

---

## PART 8: OPTIONAL TYPES (C++17 onwards)

### Before C++17

```cpp
// Returning "no value" - Ugly patterns
struct MaybeInt {
    bool has_value;
    int value;
};

MaybeInt find(const std::vector<int>& v, int target) {
    for (const auto& x : v) {
        if (x == target) {
            return {true, x};
        }
    }
    return {false, 0};  // Confusing: 0 doesn't mean anything
}

auto result = find(v, 42);
if (result.has_value) {
    std::cout << result.value << "\n";
}
```

### C++17 Solution: std::optional

```cpp
std::optional<int> find(const std::vector<int>& v, int target) {
    for (const auto& x : v) {
        if (x == target) {
            return x;  // Implicit conversion
        }
    }
    return std::nullopt;  // Explicit "no value"
}

auto result = find(v, 42);

// Check if has value
if (result.has_value()) {
    std::cout << result.value() << "\n";
}

// Or use operator*
if (result) {
    std::cout << *result << "\n";
}

// C++20: Use in if-init
if (auto result = find(v, 42)) {
    std::cout << *result << "\n";
}
```

### C++17+: std::variant (Tagged Union)

```cpp
std::variant<int, double, std::string> value = 42;

// Match on type
std::visit([](const auto& v) {
    std::cout << v << "\n";
}, value);

// Or check type
if (std::holds_alternative<int>(value)) {
    std::cout << std::get<int>(value) << "\n";
}
```

### Rust Equivalent

```rust
// Option type (like Optional)
fn find(v: &[i32], target: i32) -> Option<i32> {
    for &x in v {
        if x == target {
            return Some(x);
        }
    }
    None
}

let result = find(&v, 42);

// Check with if-let
if let Some(value) = result {
    println!("{}", value);
}

// Or match
match result {
    Some(v) => println!("{}", v),
    None => println!("Not found"),
}

// Result type (for errors)
fn parse(s: &str) -> Result<i32, ParseError> {
    s.parse().map_err(|_| ParseError)
}

// Similar patterns work

// Enum variant (like variant)
enum Value {
    Integer(i32),
    Float(f64),
    String(String),
}

let v = Value::Integer(42);
match v {
    Value::Integer(i) => println!("{}", i),
    Value::Float(f) => println!("{}", f),
    Value::String(s) => println!("{}", s),
}
```

---

## PART 9: CONCEPTS AND CONSTRAINTS (C++20)

### Before C++20 (SFINAE)

```cpp
// Complex detection with SFINAE
template <typename T, typename = void>
struct is_iterable : std::false_type {};

template <typename T>
struct is_iterable<T,
    std::void_t<decltype(std::begin(std::declval<T>()))>>
    : std::true_type {};

template <typename T>
typename std::enable_if<is_iterable<T>::value>::type
process(T& container) {
    for (auto& item : container) {
        // ...
    }
}
```

### C++20 Solution: Concepts

```cpp
#include <concepts>

template <typename T>
concept Iterable = requires(T t) {
    std::begin(t);
    std::end(t);
};

template <Iterable T>
void process(T& container) {
    for (auto& item : container) {
        // ...
    }
}

// Named concepts
template <typename T>
concept Arithmetic = requires(T a, T b) {
    { a + b } -> std::convertible_to<T>;
    { a - b } -> std::convertible_to<T>;
    { a * b } -> std::convertible_to<T>;
};

template <Arithmetic T>
T sum(T a, T b, T c) {
    return a + b + c;
}
```

### Rust Equivalent

```rust
// Traits are Rust's concept system
use std::ops::{Add, Sub, Mul};

trait Arithmetic: Add<Output = Self> + Sub<Output = Self> + Mul<Output = Self> {
}

fn sum<T: Arithmetic>(a: T, b: T, c: T) -> T {
    a + b + c
}

// Iterator trait (Iterable concept)
fn process<T: IntoIterator>(container: T) {
    for item in container {
        // ...
    }
}

// Trait bounds
fn max<T: std::cmp::Ord>(a: T, b: T) -> T {
    if a > b { a } else { b }
}

// Multiple bounds
fn process<T: Clone + Debug>(item: T) {
    println!("{:?}", item);
}
```

---

## PART 10: COROUTINES (C++20)

### What It Enables

**Before C++20 (State Machines):**
```cpp
// Awkward state machine for generator-like behavior
struct Generator {
    int current = 0;
    bool done = false;
    
    int next() {
        if (current >= 5) done = true;
        return current++;
    }
};

Generator gen;
while (!gen.done) {
    std::cout << gen.next() << "\n";
}
```

### C++20 Solution: Coroutines

```cpp
#include <coroutine>

Generator<int> generate() {
    for (int i = 0; i < 5; ++i) {
        co_yield i;  // Suspend and yield value
    }
}

// Use like generator
for (int value : generate()) {
    std::cout << value << "\n";
}

// Async with co_await
std::future<int> async_work() {
    auto data = co_await fetch_data();  // Wait for async operation
    return process(data);
}
```

### Rust Equivalent

```rust
// Rust uses async/await
async fn async_work() -> i32 {
    let data = fetch_data().await;  // Wait for async
    process(data)
}

// Generators are available on nightly
#![feature(generators, generator_trait)]

fn generate() {
    yield 0;
    yield 1;
    yield 2;
}

// Iterators (stable equivalent)
fn generate() -> impl Iterator<Item = i32> {
    (0..5).into_iter()
}

for value in generate() {
    println!("{}", value);
}
```

---

## PART 11: MODULES (C++20)

### Before C++20 (Header/Source Separation)

```cpp
// myclass.h
#ifndef MYCLASS_H
#define MYCLASS_H

class MyClass {
public:
    void doSomething();
private:
    int value;
};

#endif

// myclass.cpp
#include "myclass.h"

void MyClass::doSomething() {
    // Implementation
}

// main.cpp
#include "myclass.h"

int main() {
    MyClass obj;
    obj.doSomething();
}

// Problems:
// - Include guards are error-prone
// - Separate compilation model
// - Slow compilation
// - No module interface
```

### C++20 Solution: Modules

```cpp
// MyClass.cppm (Module implementation)
export module MyClass;

export class MyClass {
public:
    void doSomething();
private:
    int value;
};

void MyClass::doSomething() {
    // Implementation
}

// main.cpp
import MyClass;

int main() {
    MyClass obj;
    obj.doSomething();
}

// Benefits:
// - No include guards needed
// - Faster compilation
// - Clear interface
// - No macro pollution
```

### Rust Equivalent

```rust
// lib.rs (library module)
pub mod myclass {
    pub struct MyClass {
        value: i32,
    }
    
    impl MyClass {
        pub fn do_something(&self) {
            // Implementation
        }
    }
}

// Or in separate file: myclass.rs
pub struct MyClass {
    value: i32,
}

impl MyClass {
    pub fn do_something(&self) {
        // Implementation
    }
}

// main.rs
use myclass::MyClass;

fn main() {
    let obj = MyClass { value: 0 };
    obj.do_something();
}

// Rust modules:
// - Always available (no preprocessor)
// - Clear visibility (pub keyword)
// - Integrated with package system
// - Fast compilation
```

---

## PART 12: SUMMARY TABLE - ALL NEW FEATURES

| Feature | C++ Version | Purpose | Rust Equivalent |
|---------|---|---|---|
| **Smart Pointers** | C++11 | Automatic memory management | Box, Rc, Arc |
| **Move Semantics** | C++11 | Efficient resource transfer | Ownership system |
| **Lambda Functions** | C++11 | Inline functions | Closures with Fn traits |
| **Auto Type Deduction** | C++11 | Reduce verbosity | Type inference |
| **Range-based For** | C++11 | Iterator loops | for-in loops |
| **Variadic Templates** | C++11 | Variable argument templates | Macros, generics |
| **Constexpr** | C++11+ | Compile-time computation | const fn |
| **Optional** | C++17 | Represent missing values | Option<T> |
| **Variant** | C++17 | Tagged unions | Enums |
| **Structured Bindings** | C++17 | Destructure data | Pattern matching |
| **Concepts** | C++20 | Template constraints | Traits |
| **Coroutines** | C++20 | Async operations | async/await |
| **Modules** | C++20 | No header files | Module system |

---

## PART 13: EVOLUTION TIMELINE

```
C++11 (2011):
├─ Smart pointers (unique_ptr, shared_ptr)
├─ Move semantics (&&)
├─ Lambda functions
├─ Auto type deduction
├─ Range-based for loops
└─ Variadic templates

C++14 (2014):
├─ Generic lambdas (auto parameters)
├─ std::make_unique
└─ std::get for tuples

C++17 (2017):
├─ std::optional
├─ std::variant
├─ Structured bindings
├─ if constexpr
└─ C++20 preparation (concepts drafted)

C++20 (2020):
├─ Concepts and constraints
├─ Coroutines (co_await, co_yield)
├─ Ranges
├─ Modules (std::)
└─ Three-way comparison (<=>)

C++23 (2023):
├─ deduced this
├─ std::expected
├─ Stacktrace
└─ Improved module support

C++26 (In Development):
├─ Further module improvements
├─ Reflection (proposed)
├─ Pattern matching (proposed)
└─ More constexpr capabilities
```

---

## CONCLUSION

### How C++ and Rust Compare

**C++ Approach:**
- Started with basic features
- Added safety features gradually (smart pointers, optional, variants)
- Still maintaining backward compatibility
- Learning curve for each new version

**Rust Approach:**
- Designed with safety from day 1
- Ownership and borrowing built-in
- Optional/Result as core types
- Traits as core abstraction
- Async/await from early on

### Key Takeaway

Modern C++ (C++20+) has adopted many patterns pioneered by Rust and other languages:
- Smart memory management (like Rust's ownership)
- Type constraints (like Rust's traits)
- Algebraic data types (like Rust's enums)
- Async support (like Rust's async/await)

However, C++ maintains backward compatibility, making it more complex. Rust had the luxury of starting fresh with these patterns built in from day 1.

---

**Document Version:** 1.0  
**Last Updated:** 2024  
**C++ Standards Covered:** C++11 through C++26 (proposed)  
**Code Examples:** 70+ samples
