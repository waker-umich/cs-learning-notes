# Content

- [Chapter 1: Comparisons and Operator `<=>`](#chapter-1-comparisons-and-operator)
- [Chapter 2: Placeholder Types for Function Parameters](#chapter-2-placeholder-types-for-function-parameters)
- [Chapter 3~5: Concepts, Requirements and Constraints](#chapter-35-concepts-requirements-and-constraints)
- [Chapter 6~9: Ranges and Views](./ranges/ranges.md)
- [Chapter 10: Formatted Output](#chapter-10-formatted-output)
- [Chapter 12: `std::jthread` and Stop Tokens](#chapter-12-stdjthread-and-stop-tokens)

# Chapter 1: Comparisons and Operator `<=>`

- defining `operator==` enables use for all equality operators
    - compiler rewirtes `a != b` to `!(a == b)`
    - change the order when implicit conversion applies
- defining `operator<=>` enables uses for all relational operators
    - called "three-way comparison operator", "spaceship operator"
    - compiler rewrites `a < b` to `a <=> b < 0`
    - change the order when implicit conversion applies
    - defined as `default` to perform member-wise comparison
        - also implicitly generated `default` version of `operator==`
      ```cpp
      struct S {
        // enable use of all comparison operaotors
        [[nodiscard]] auto operator<=> (const S& rhs) const = default;
      };
      ```
- for `default` versions
    - `noexcept` if comparing the members `noexcept`
    - `constexpr` if possible

## Return Types of `operator<=>`

- returns __comparison category types__ instead of `bool`
- `std::strong_ordering` for the types that one of (`==`, `<`, `>`) must applies for any two values
    - contains static member constants `less`, `equal`/`equivalent`, `greater`
- `std::weak_ordering` for the types that __equivalent__ values might not be __equal__ (eg. case-insensitive strings)
    - contains static member constants `less`, `equivalent`, `greater`
- `std::partial_ordering` for the types that contains comparable value (e.g. `NaN` in floating-point type, any comparison yields `false`)
    - contains static member constants `less`, `equivalent`, `greater`, `unordered`
- notes
    - static member constants have the same type as the class
        - e.g., `std::strong_ordering::less` is of type `std::strong_ordering`
    - comparison category types are not integers, they contain an integral type (`signed char` in [gcc](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/libsupc%2B%2B/compare)) data member
        - can only be compared with `0` (or `nullptr`/`NULL`)
    - stronger ordering types can be implicitly converted to weaker ordering types
        ```cpp
        using namespace std;
        static_assert(
            is_same_v<
                common_comparison_category_t<strong_ordering,
                                             weak_ordering>,
                weak_ordering
            >);
        ```

# Chapter 2: Placeholder Types for Function Parameters

- function template syntax can be abbreviated
    ```cpp
    void foo(auto t);
    ```
- in the following code, `foo1` is equivalent to `foo2`
    ```cpp
    template <class T>
    void foo1(auto t1, T t2);

    template <class T2, class T1>
    void foo2(T1 t1, T2 t2);
    ```

# Chapter 3~5: Concepts, Requirements and Constraints

refer to [template note](../cppcon/template/template.md)

# Chapter 6~9: Ranges and Views
refer to [ranges note](./ranges/ranges.md)

# Chapter 10: Formatted Output

## format of format strings
- replacement fields specified `{}`
    - use `{{` to print `{`, use `}}` to print `}`
- in replacement fields: `{n:fmt}`
    - `n`: argument index
    - `fmt`: format specifier
    - both are optional and seperated by colon `:`
- format specifiers: `fill align sign # 0 width .prec L type`
    - all are optional, bulit-in types have default format specifiers
    - `align`
        - `<`(left-aligned), `>`(right-aligned), `^`(centered)
        - `fill`
            - must be used with `align`
            - the character to fill the field up to `width`
            - default: space
    - `width` specifies minimum field width
    - `.prec`
        - for floating-point types, specifies number of digits after dot(`.`)
        - for strings, specifies maximum number of characters

## usage

```cpp
constexpr const char* fmt = "key: {1:06.3}, val: {0:.3}";
std::format(fmt, "abcdefg", 3.4);
// "key: 0003.4, val: abc"
```
- format string must be a compile-time value
    - allows error checking in compile time, better performance
    - if not, use `std::vformat()`
- since C++23, `std::print()` equivalent to `std::cout << std::format()`

# Chapter 12: `std::jthread` and Stop Tokens

## Stop Sources and Stop Tokens

- code example [stop.cpp](./stop.cpp)
- a general-purpose mechanism to asynchronously request a stop
    - the asynchronous task that is requested to stop can have multiple ways to react to it
- `std::stop_source` is used to request a stop and `std::stop_token` is used to react to a stop request
- each `std::stop_source` and `std::stop_token` is associated with a shared stop state
    - stop state is created when default constructing `std::stop_source`
    - a stop state can only be requested stop once
    - stop state is destroyed only when the last of `std::stop_source`, `std::stop_token` and `std::stop_callback` is destroyed
- `std::stop_callback` can be used to register a callable when a stop is requested
    - it is constructed using a `std::stop_token` and a callable
    ```cpp
    void task(std::stop_token st) {
        // register temporary callback:
        std::stop_callback cb{st, []{
            std::cout << "stop requested\n";
        }};
    }   // unregisters callback
    ```
    - the callable cannot throw, otherwise `std::terminate` is called
    - register and unregister process are carefully synchronized that only one of __three__ cases can happen:
        - stop request before construction of `std::stop_callback`, then __callable__ will be called immediately where the callback is initialized
        - stop request after construction and before destruction of `std::stop_callback`, then __callable__ will be called immediately where `request_stop()` was called
            - `request_stop()` blocks until all registered callables have been called
            - order of calls not defined
        - stop request after destruction of `std::stop_callback`, then __callable__ will never be called


## `std::jthread`

- an enhanced version of `std::thread`
    - `RAII`: `join` is called in destructor (if `joinable`)
    - provides a _cooperative_ mechanism to signal that a thread should no longer run
        - it is _cooperative_ since the stop request does not kill the thread, which will leave the program in a corrupt state
        - uses [stop tokens](#stop-sources-and-stop-tokens)
        - the ctor that starts a thread creates a `std::stop_source` and stores it as a member of the thread object
        - if the called function's first parameter is `std::stop_token` and ctor does not receive a argument for it, then ctor will generate a `std::stop_token` to pass to the function

<p align="center">
    <img src="./jthread_op.png" width="700" align="center">
</p>

```cpp
{
    std::vector<std::jthread> threads;
    auto task = [](std::stop_token st) {
        while (!st.stop_requested()) {
            std::cout << "do something\n";
        }
    };
    for (int i = 0; i < numThreads; i++) {
        threads.push_back(std::jthread{task});
    }
    // request stops for all threads before we start to join
    for (auto& t : threads) t.request_stop();
}   // dtor joins all threads
```

