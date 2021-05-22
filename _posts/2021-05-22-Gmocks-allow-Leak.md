---
layout: post
title: "gMock's AllowLeak()"
---
#### ::testing::Mock::AllowLeak( )
During a discussion with a coworker, I've found out the existence of this method.

(you can check also this [video](https://www.youtube.com/watch?v=-3GdfsPf1OY) if you want to know more about it)

Let's take a look into it's source code:

```cpp
// Tells Google Mock to ignore mock_obj when checking for leaked mock
// objects.
void Mock::AllowLeak(const void* mock_obj)
    GTEST_LOCK_EXCLUDED_(internal::g_gmock_mutex) {
  internal::MutexLock l(&internal::g_gmock_mutex);
  g_mock_object_registry.states()[mock_obj].leakable = true;
}
```

Ok, so it seems ``gMock`` checks for leaked mock objs. 

Cool!

Let's see how it works in this [example](https://godbolt.org/z/WjoEj8YdW):

```cpp
#include <iostream>
#include <stdint.h>
#include "gtest/gtest.h"
#include "gmock/gmock.h"

(...)

struct Mck {
    MOCK_METHOD(void, foo, ());
};

TEST(TestSuite, TestCase) {
    auto* m = new Mck;
    EXPECT_CALL(*m, foo);
}
```

And we got this output:

```cpp
Running main() from /opt/compiler-explorer/libs/googletest/release-1.10.0/googletest/src/gtest_main.cc
[==========] Running 1 test from 1 test suite.
[----------] Global test environment set-up.
[----------] 1 test from TestSuite
[ RUN      ] TestSuite.TestCase
[       OK ] TestSuite.TestCase (0 ms)
[----------] 1 test from TestSuite (0 ms total)

[----------] Global test environment tear-down
[==========] 1 test from 1 test suite ran. (0 ms total)
[  PASSED  ] 1 test.

/app/example.cpp:22: ERROR: this mock object (used in test TestSuite.TestCase) should be deleted but never is. Its address is @0x1adc8e0.
ERROR: 1 leaked mock object found at program exit. Expectations on a mock object is verified when the object is destructed. Leaking a mock means that its expectations aren't verified, which is usually a test bug. If you really intend to leak a mock, you can suppress this error using testing::Mock::AllowLeak(mock_object), or you may use a fake or stub instead of a mock.
```

So great, ``gMock`` give us an error, saying that the mock was leaked,
 and it mentions the possible usage of ``testing::Mock::AllowLeak(mock_object)`` to prevent this error.

Let's try to do [that](https://godbolt.org/z/EbPd7Td7b):
```cpp
TEST(TestSuite, TestCase) {
    auto* m = new Mck;
    EXPECT_CALL(*m, foo);
    ::testing::Mock::AllowLeak(m);
}
````

Now our test passes:

```cpp
Running main() from /opt/compiler-explorer/libs/googletest/release-1.10.0/googletest/src/gtest_main.cc
[==========] Running 1 test from 1 test suite.
[----------] Global test environment set-up.
[----------] 1 test from TestSuite
[ RUN      ] TestSuite.TestCase
[       OK ] TestSuite.TestCase (0 ms)
[----------] 1 test from TestSuite (0 ms total)

[----------] Global test environment tear-down
[==========] 1 test from 1 test suite ran. (0 ms total)
[  PASSED  ] 1 test.
```

Wow...wait a minute...We had an expect call, and it wasn't satisfied, so what's the story there?

Why did our test pass?

``gMock`` will verify the expectations on a mock object when it is **destructed**.

But in our test, the ``Mck`` obj was leaked, it was never destroyed so all the expectations on it, were never verified.

And because of that the test passes.

Using the ``::testing::Mock::AllowLeak(m)`` basically turned our mock into a fake kinda obj.

If one cannot fix his code, or its using global/static mocks that don't get destructed at the end of each test,
one can manually verify these expectation and mimic the ``gMock's`` behaviour at destruction time.


For that one can use (taken from [gMock cheat sheet](https://google.github.io/googletest/gmock_cheat_sheet.html)):

```cpp
using ::testing::Mock;
...
// Verifies and removes the expectations on mock_obj;
// returns true if and only if successful.
Mock::VerifyAndClearExpectations(&mock_obj);
...
// Verifies and removes the expectations on mock_obj;
// also removes the default actions set by ON_CALL();
// returns true if and only if successful.
Mock::VerifyAndClear(&mock_obj);
```

After verifying and clearing a ``mock`` one should never set new expectations on it.


