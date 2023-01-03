# Run Tests when Building with CMake

> This is a re-upload of a blog post I wrote back in 2020-02-06 with
> an anonymous account that I recently removed. I re-upload here in
> the hopes that someone will find it useful.
>
> 2023-01-03

## Purpose

Tests are good in many ways: they expose bugs, helps you to not
regress, makes you feel confident, etc. BUT one huge flaw with tests
is that you need to remember to run them. Would it not be nice to be
able to run tests as part of the build process, as the following
snippet outlines?

```text
Scanning dependencies of target alpha
[1%] Building CXX object alpha/CMakeFiles/alpha.dir/src/...
[10%] Linking CXX static library libalpha.a
[11%] Built target alpha
Scanning dependencies of target alphatest
[12%] Building CXX object alpha/CMakeFiles/alphatest.dir/tst/...
[20%] Linking CXX executable alphatest
[21%] Built target alphatest
[22%] Running target alphatest
===============================================================================
All tests passed (1337 assertions in 42 test cases)
[23%] Built target runalphatest
Scanning dependencies of target beta
[24%] Building CXX object beta/CMakeFiles/beta.dir/src/...
[30%] Linking CXX static library libbeta.a
[31%] Built target beta
Scanning dependencies of target betatest
[32%] Building CXX object beta/CMakeFiles/beta.dir/tst/...
[40%] Linking CXX executable betatest
[41%] Built target betatest
[42%] Running target betatest

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
betatest is a Catch vX.Y.Z host application.
Run with -? for options

-------------------------------------------------------------------------------
string view can find
  existing characters
-------------------------------------------------------------------------------
/home/shossjer/myproject/beta/tst/utility/string_view.cpp(9)
...............................................................................

/home/shossjer/myproject/beta/tst/utility/string_view.cpp(11): FAILED:
  CHECK( a.find('a') == 1 )
with expansion:
  0 == 1

===============================================================================
test cases:   91 |   90 passed | 1 failed
assertions: 1020 | 1019 passed | 1 failed
```

This makes sense to do, and for C++ it does not even need to impact
your build time noticeably because running C++ is way faster than
building it!

## CMake Setup

Assuming you have your favorite test environment up and running with
some sort of separation between regular code and test code, you might
want to modify your setup a bit to make this work as well as it
can. In order to run the tests as early as possible, it is probably
wise to split the regular code into not-too-large libraries and have
test code for each such library, something like in this snippet.

```cmake
cmake_minimum_required(VERSION 3.7)

project(myproject CXX)

option(RUN_TESTS "Run tests as part of the build process." ON)

add_library(alpha "")
target_sources(alpha PRIVATE ...)
target_include_directories(alpha PRIVATE ...)
target_link_libraries(alpha PRIVATE ...)

add_executable(alphatest "")
target_sources(alphatest PRIVATE ...)
target_include_directories(alphatest PRIVATE ...)
target_link_libraries(alphatest PRIVATE alpha)
set_target_properties(alphatest PROPERTIES VS_DEBUGGER_WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}")

add_library(beta "")
target_sources(beta PRIVATE ...)
target_include_directories(beta PRIVATE ...)
target_link_libraries(beta PRIVATE alpha ...)

add_executable(betatest "")
target_sources(betatest PRIVATE ...)
target_include_directories(betatest PRIVATE ...)
target_link_libraries(betatest PRIVATE alpha beta)
set_target_properties(betatest PROPERTIES VS_DEBUGGER_WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}")
```

## First attempt

CMake supports adding custom commands to targets, that is, extra
things to run as part of building the target. Interesting for us is
the ability to try and run the test executable after it has been
built, and we can use `add_custom_command` for this with the special
`POST_BUILD` argument, see below.

```cmake
add_library(alpha "")
# ...

add_executable(alphatest "")
# ...

if(RUN_TESTS)
    add_custom_command(
        TARGET alphatest
        POST_BUILD
        COMMAND $<TARGET_FILE:alphatest>
        WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}"
        )
endif()
```

This kind of works. As part of building the target `alphatest`, we
will now also run it. There is one problem however, what happens when
the test fails? Well, since running the tests are part of the same
target as building the tests, the target will fail and the build will
be thrown away. This means that we will never be able to debug a
failing test, ouch!

This is a pretty big problem since the whole idea of tests is for them
to break so you can fix them again. (If you are working in a company
that believes that test suits should never fail then you have my
sympathy.)

## Better attempt

We need to separate the running-part of the test from the
building-part. If we made a new target with a custom command to run
the executable then it would run every time we build our project. This
feels inefficient, since why should we rerun tests when nothing has
changed? Other than it will be annoying, it might be fine.

We could also try and fix that. Stack Overflow user 1178599 came up
with an easy solution to the problem (see [cmake add_custom_command
failure, target gets
deleted](https://stackoverflow.com/a/53673873)). The gist of it is
about adding an extra artifact that can keep track of the state of the
test, if it has passed since the executable was built. CMake loves
files, so this artifact is in fact just a file, see snippet below.

```cmake
function(run_tests name executable)
    set(_test_state "${PROJECT_BINARY_DIR}/run_tests/${executable}.passed")
    add_custom_command(
        OUTPUT ${_test_state}
        COMMAND $<TARGET_FILE:${executable}>
        COMMAND ${CMAKE_COMMAND} -E make_directory "${PROJECT_BINARY_DIR}/run_tests"
        COMMAND ${CMAKE_COMMAND} -E touch ${_test_state}
        DEPENDS ${executable}
        WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}"
        )
    add_custom_target(
        ${name}
        ALL
        DEPENDS ${_test_state}
        )
endfunction()
```

This function creates a new target, named `${name}`, and on line 14
decides that this target should depend on the artifact. It also sets
up a custom command that on line 4 is said to product the artifact and
that it depends on some given executable on line 8. (The actual
construction of the artifact happens on line 7.)

This means that whenever the executable is changed, it will trigger
the custom command to run and (maybe) output a new artifact. As long
as the artifact fails to be updated, the custom command will continue
to trigger when building. The purpose of the custom target is just to
have something that depends on the artifact. And here is how you would
use this function, see below.

```cmake
add_library(alpha "")
# ...

add_executable(alphatest "")
# ...

if(RUN_TESTS)
    run_tests(runalphatest alphatest)
endif()
```

Depending on your build generator, it might be the case that the test
targets will be run out of order and not immediately after the library
that should be tested has been built. You can add a fake dependency to
force tests to run before the next stage is built, see below.

```cmake
add_library(beta "")
# ...

add_executable(betatest "")
# ...

if(RUN_TESTS)
    add_dependencies(beta runalphatest)
    run_tests(runbetatest betatest)
endif()
```

Now the only thing that remains for this to be actually useful is for
you to convince your colleagues to START WRITING TESTS. Good luck!
