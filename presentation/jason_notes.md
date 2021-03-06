# Implementation Notes

* Earlier misunderstanding with compiler prevented use of `union` in a `constexpr` context. That was resolved. Getting initialization right can be "hard"
* Without `constexpr` support for lambdas in C++17, much of this would be much harder
* Many algorithms that are not currently `constexpr` could be made such. 
  - `find_if`
  - `mismatch`
  - `equal`
  - `copy`
  - `move` (not the cast operator) - interesting side note, `std::make_move_iterator` *is* `constexpr` in c++17
* `std::optional` 
  - Gets the hard part right: it's trivially destructable if the contained type is
  - However, gets everything else wrong - non-`constexpr` copy/move/assignment
  - Making it functionally useless in a `constexpr` context
* `std::variant`
  - See `std::optional`
* `map` and `vector` is easy to implement, even with iterator support. The only thing that makes it hard is deciding the size ahead of time. 
* I personally found the `static_string` class for a fixed-size wrapper around a `char *` literal to be very helpful. It is like a `string_view` but with more restrictions so you know you can store it (I think). I believe there's a proposal floating around similar to this.
 
# General Notes
 
* Debugging issues is hard
* Tests are surprisingly easy. There's no risk of something going wrong at runtime, if a test compiles, it worked!
* What about use cases for mixed-mode `constexpr` with fixed-size storage that optionally expands (is Eric Neibler around? He worked on this with his string class)
* Compile times are tricky to manage, but it could be worse. The hard part - which Ben will speak about - is building / flattening trees. Really this is not a `constexpr` issue, but a template issue in our cases.

# Allocators

* It should be possible to make a `constexpr` allocator, and containers that support both `constexpr` and not allocators

This comes with some difficulties, however.

* Placement `new` is not supported in `constexpr` (but this is something other people have been discussing)
* But this is largely irrelevant anyhow, as with `constexpr` data the contained thing must have already been allocated anyhow
* But this requires care to choose placement `new` when required, and not otherwise
* Growing the container is difficult because there really does not exist scratch allocator space since we are working with a very limited size storage that must be known at compile time
* This suggests some amount of awareness of the part of the container to allocate as much as it can up front, for a fixed-size allocator
* Cleaning up becomes almost impossible because a non-trivial destructor is impossible. This would mean that we need some kind of non-`constexpr` allocator wrapper that can clean up allocated memory on exit
* This should be possible, but adds some more complexity, but should work in the same way that `std::variant`'s `constexpr` destructor does
* What would be awesome is if an empty, but user defined, destructor could be considered "trivial" for the purposes of `constexpr`. This would allow us to `if constexpr` out the body of the destructor when possible

How to make a constexpr-safe destructor:

```cpp
#include <type_traits>
#include <memory>


template<typename Derived>
struct CleanUp
{
  ~CleanUp() {
    static_cast<Derived *>(this)->cleanup();
  }
};

template<typename Derived>
struct CleanUpTrivial

{
  // nothing to do!
};

template<typename T>
struct Container : std::conditional_t<
                                      std::is_trivially_destructible_v<T>, 
                                      CleanUpTrivial<Container<T>>, 
                                      CleanUp<Container<T>>
                                     >
{
  void cleanup() {
    // do cleanup stuff
  }
};

int main()
{
  static_assert(std::is_trivially_destructible_v<Container<int>>);
  static_assert(!std::is_trivially_destructible_v<Container<std::unique_ptr<int>>>);
}
```


# The Future

* What about overloading on `constexpr` usage?
* Can we remove the restriction on trivial destructors?
* What is the status on allowing dynamic allocations? 

