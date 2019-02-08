---
title: How to abuse unique_ptr
categories: coding
tags: cpp
---

After all those years there is still no [`unique_resource`](https://wg21.link/P0052) for exception-safe and correct low-level code.
Abusing `unique_ptr` is a quick and dirty alternative, at least if the resource type satisfies *[NullablePointer](https://en.cppreference.com/w/cpp/named_req/NullablePointer)* like `FILE*` does.
Windows handles for example are in fact defined as pointers but the invalid value is -1 ([sort of](https://devblogs.microsoft.com/oldnewthing/20040302-00/?p=40443)).

Since C++20 introduced an interesting change related to that let's recap how it can be done in the different language versions (without macros).
Our running example will be the very common fopen/fclose scenario.

## C++11/14

The straightforward way is to just pass `fclose` as a custom deleter to `unique_ptr`.
A `make_unique`-like helper function with convenient type inference is easily written:

```cpp
template <typename T, typename D>
auto unique_resource(T* t, D&& d)
{
    return std::unique_ptr<T, D>(t, std::forward<D>(d));
}

auto f = unique_resource(fopen(file, "r"), fclose);
```

Job done... isn't it?

This code is "optimal" for lambdas/function objects but function pointers are saved into `f` even if they are known at compile-time.
As long as f is not passed around that's no big deal.
Otherwise this can be fixed by wrapping it in a lambda function, thus sort of encoding that information into the Deleter type:

```cpp
unique_resource(file, [](FILE* f) noexcept { fclose(f); })
```

The lambda object is still copied to the unique_ptr but since it's empty (stateless) it does not take up space due to tuple/pair compression behind the scenes.

Of course this is now quite some boilerplate code, even if you create a wrapper specialized for a particular resource type.

## C++17

So let's try lifting the function pointer to a compile-time parameter.
`auto` template parameters introduced a convenient way to do that.

```cpp
template <auto cleanup, typename T>
auto unique_resource(T* t)
{
    auto wrap = [](T* t) noexcept { cleanup(t); };
    return std::unique_ptr<T, decltype(wrap)>(t, wrap);
}

auto f = unique_resource<fclose>(cfile);
```

Now that lambda detail is nicely hidden. Only the API might be a bit confusing at first sight (unique_resource of type fclose?).
But actually we don't want to specify the cleanup function every time.

We'd like to define an alias, preferably with a concise and intuitive syntax, something like:

```cpp
using unique_file = unique_resource<fclose>;
```

One way (the only way?) to achieve that is a generic function wrapper:

```cpp
template <auto f>
struct FunctionWrapper
{
    template <typename... T>
    auto operator()(T&&... t)
    {
        return f(std::forward<T>(t)...);
    }
};
template <typename T, auto cleanup>
using unique_resource = std::unique_ptr<T, FunctionWrapper<cleanup>>;

using unique_file = unique_resource<FILE, fclose>;
```

Mission accomplished.

In theory the resource type parameter could even be deduced from the cleanup function signature with something like:

```cpp
template <typename T, typename R>
T getParamTypeHelper(R (*f)(T));

template <auto f>
using ResourceType = decltype(getParamTypeHelper(f));

template <auto cleanup, typename T = std::remove_pointer_t<ResourceType<cleanup>>>
using unique_resource = std::unique_ptr<T, FunctionWrapper<cleanup>>;
```

## C++20

C++20 removes some restrictions on lambdas, finally allowing to pass them to template parameters ([P0315](https://wg21.link/p0315)) and making them default constructible ([P0624](https://wg21.link/p0624)).

That obviates the need for the `FunctionWrapper` helper and we can do it the intuitive way in just 2 lines:

```cpp
template <typename T, auto cleanup>
using unique_resource = std::unique_ptr<T, decltype([](T* t) noexcept { cleanup(t); })>;
```

This is still far away from [D](https://dlang.org/spec/template.html#aliasparameters)'s elegance but rather pretty in C++ terms.
