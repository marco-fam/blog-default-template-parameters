# C++ static_assert: Changing functions to template functions. 
Recently I came across a problem where it was not possible to use static_assert in a non templated function.
The issue we tried to solve is that some functions are only allowed to be compiled for platforms that have HW support of double. 
On all other platforms double precision is allowed, since double precision is needed for simulation of real physical processes.

To give an idea of the initial problem a link to compiler explorer is given with a simple piece of code that shows that it is 
not possible for ordinary non template functions to use static_assert to limit platform compilation.

https://godbolt.org/z/99ht3H

For completeness the code is show below also. Note that the constexpr variable can be set via a platform specific define.
```
// based on preprocessor magic we set the cpu target to nios2
constexpr bool cpu_nios2 = true;

// we want to prevent users from calling this version on cpu_nios2 targets
constexpr double square(double num) noexcept {
    static_assert(!cpu_nios2,"Prevent this from compiling on cpu_nios2 targets");
    return num * num;
}

int main() {
  // even  when we do not use the square function the static_assert can not be prevented.
}
```

The problem is that ordinary non template functions are compiled regardless if they are used or not.
In this case, what we really want, is for the compiler to ignore static_asserts for functions that are not used in the main program.

To work around this problem we can change the ordinary non template function square to a template function and give it a default
template parameter.
This enables us to keep the calling conversion and prevent us from specifying another function argument in order for the compiler to
deduce the type automatically. Alternatively the function could be called as square\<T\>(...) which would be more verbose and change existing user code.
If we use another type trait we get the following implementation with the same function api of the non template square function.

A compiler explorer link for the final solution is given below.

https://godbolt.org/z/gG0Dma

```
#include <type_traits>

// based on preprocessor magic we set the cpu target to cpu_nios2 = {true, false}
constexpr bool cpu_nios2 = true;

// Non template version of square will not be able to compile for cpu_nios2=true
// see https://godbolt.org/z/99ht3H

// The template version of will make the code compile even when we do not use square in our program.
// The default template parameter allows us to use std::is_same without providing another 
// function argument to square for the compiler to detect T or calling square<T> explicitly. 
// This version has the same calling syntax as the non template version so no user code needs to be changed.  
template <typename T = double>
constexpr double square(double num) noexcept {
    static_assert(!cpu_nios2 && std::is_same<T,double>::value, "This function should only be used with double!");
    return num * num;
}

int main() {
  // It is now possible to compile the program even when square is not instantiated for cpu_nios2 targets. 
  // This is not possible with the non template square version.
  
  double tmp;
  //tmp = square(3);      // fails when cpu_nios2 is true but compiles when cpu_nios2 is false. Same calling syntax as ordinary function.
  //tmp = square<int>(3); // fails to instantiate square for all other types than double!
  return tmp;
}
```

# Conclusion
If you have different target platforms that needs to restrict certain functions from compiling consider changing your
ordinary non template functions to template functions with default template parameters.

The benefit of this approach is that no client code needs to be changed as the function calling syntax stays the same.
Additionally the function can be limited to the exact type by using std::is_same<T,double>::value so that clients are
prohibited in creating e.g. a square\<int\> version thus keeping the binary size the same or smaller. See Link Time Optimization for another way of keeping the binary size smaller.

