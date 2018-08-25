# Default-template-parameters prevents problems with static_assert. 
Recently I came across a problem using static_assert. The problem was that a piece of code was not allowed to compile on the target
platform as it has no support for double. However on x86 platforms the code was intended to compile as it is used for simuling real 
physical systems where high precision is needed. 

Example of functions templates that benefit of default parameter due to static_assert issue

