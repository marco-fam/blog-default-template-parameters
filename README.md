# Changing non template functions to template functions when static_assert fails for platform specific code. 
Recently I came across a problem using static_assert. 
The problem was that a piece of code was not allowed to compile on a specific target platform since double is not supported. 
However on all other platforms the code was intended to be used and compiled since double precision is needed for simulation 
real physical processes etc.

To give an idea of the initial problem a link to compiler explorer is given with a simple piece of code that shows that it is 
not possible for ordinary non template functions to use static_assert to limit platform compilation.

https://godbolt.org/z/99ht3H

For completeness the code is show below also.
```
// based on preprocessor magic we set the cpu target to nios2
constexpr bool cpu_nios2 = true;

// we want to prevent users from calling this version on cpu_nios2 targets
int square(double num) {
    static_assert(!cpu_nios2,"Prevent this from compiling on cpu_nios2 targets");
    return num * num;
}

int main() {
  // even  when we do not use the square function the static_assert can not be prevented.
}
```

The problem is that ordinary functions are compiled regardless if they are used or not.
In this case what we really want is for the compiler to ignore static_asserts for functions that are not used in the main program.

To work around this problem we can change the non template function to a template function and give it a default template parameter
to keep the calling conversion and prevent us from specifying another function arguments. If we do not have a default template parameter 
we would need a second argument for the compiler to deduce the type or call the function as square<T> which would be more verbose and change
existing code.
Additionally we sprinkle a little extra type traits and get the following implementation with the same function api.

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
  //tmp = square(3);      // fails when cpu_nios2 is true but compiles the cpu_nios2 is false. Same calling syntax as ordinary function.
  //tmp = square<int>(3); // fails to instantiate square for all other types than double.
  return tmp;
}
```

# Conclusion
If you have different target platforms that needs to restrict certain functions from compiling consider changing your
ordinary non template functions to template functions with default template parameters.

The benefit of this approach is that no client code needs to be changed as the function calling syntax stays the same.
Additionally the function can be limited to the exact type by using std::is_same<T,double>::value so that clients are
prohibited in creating e.g. a square\<int\> version thus keeping the binary size the same or smaller in case LTO is not use.

