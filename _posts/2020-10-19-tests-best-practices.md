---
title: "Tests Best Practices"
date: 2020-10-19
categories:
  - blog
tags:
  - test
---

Tests are an integral part of any software project.
They are essential to a software project success and quality.

They are part of my daily routine when coding and reviewing code.

Tests, of all levels may support a project growth when done correctly but also
can slow down development when treated suboptimal.

In this post I will go over some basic tests definitions and best practices
I learned through my development path.

## Test Definitions

### A generic test is expected to include the following steps in its flow:
- Setup: Preparing any pre-requirements for the test (body) to operate.

- Test Body: The actual tested scenario which is examined.

 - Teardown: Cleanup/recover any resources & state that the setup or test body
   changed.

### Assertion:
Most tests, including their setup, include assertions that check expected
results from individual operations.

Asserts include convenient checks for various conditions and provide verbose
information on failures.

### Skipping:
Skips can be split into two categories: the ones that are hard-coded to skip
and those that will skip per a specific condition during run-time.

The first fits an expected test to fail scenario, where a test is added to
show a limitation or a bug in the system, expecting to be fixed in the future.

The second is used in various scenarios, most commonly to detect a platform or
a missing capability in which the test is not expected to pass, therefore set
to skip.

An (improved) alternative to the dynamic skip is to classify well the tests
(e.g. “linux-only”, “feature-foo”) and at test execution, use their
marking/labeling to filter them out from the run list.
This leaves the control to the test runner logic and avoids unintentional
skips to occur, introducing holes in the coverage.

### Focus:
It is common during development and troubleshooting to run specific tests and
not the whole suite.

It is expected that tests will correctly operate, running the needed setup and
teardown associated with them.
This emphasises that a test should be isolated with no dependencies on other
tests (as there is no guarantee on how it will be called).


## Test Best Practices
The following best practices are agnostic to languages and frameworks,
they outline patterns that may be controversial and sometimes not easy to
apply on some systems.

### Fail First
A test should fail before it passes otherwise it may end up always passing,
catching nothing.

### Readability and Structure
Test code that is easy to read, is easy to understand and maintain.

As with any code base, use well named tests, variables and
functions. Keep functions small and focused, split into directories and
files with a good and intuitive structure so it will be easy to track
tests/helpers and know where to add new ones (or avoid adding duplicate ones).

Note: Avoid generic names like "utils", "common" and alike. They usually start
small but quickly get filled with unrelated content, becoming a junkyard of
tests/helpers that do not fit anywhere else.

### Test body and Fixture distinction
Place the operations into the right step (setup, test body, teardown).

Indirect operations that are not in the main focus of the test, better be
placed in the setup or teardown fixtures and not in the test body.

As an example, testing if an entity can be successfully created fits a test
body.

```
setup:
    # empty
body:
    foo = CreateFoo()
    assert foo
teardown:
    assert DeleteFoo(foo)
```

However, testing if an entity can communicate with another is about
the communication check and not its creation or deletion, therefore, the
creation will better fit the setup (and deletion into the teardown) while the
communication check fits the test body.

```
setup:
    foo1 = CreateFoo()
    assert foo
    foo2 = CreateFoo()
    assert foo
body:
    assert foo1.ping(foo2)
teardown:
    err1 = DeleteFoo(foo1)
    err2 = DeleteFoo(foo2)
    assert err1
    assert err2
```

The distinction allows to:
- Differentiate between assumptions (setup) and uncertainty.
- Better identify and clarify the problem.
  With our last example, if the entity fails to get created (or destroyed),
  it is not a communication problem, i.e. not related to the test main check.
- Share fixtures with multiple tests.

### Isolation
Dependencies between tests should not exist.
Changes on one test should have no effect on other tests in the suite.

It is a good practice to leave the target
([SUT](https://glossary.istqb.org/en/search/SUT)) in a (clean) state such that
no leftovers will leak to the next test.

Leaks between tests are known to create flakiness and unintended
dependencies which can manifest as false positives or false negatives.

### Keep assertions in tests
Using assertions in helper libraries are better avoided.

It is preferred to keep the control of the flow in the test body or its
fixtures and not in the helper libraries it calls.

Embedding assertions in helper libraries does not allow tests to control
how to react to errors or specific responses, limiting the usage of such
helpers by tests and the information that can be provided (data may have
been collected on the way back of the stack which could be relevant for
debugging).

Note: Custom assertion wrappers are considered assertions. These are focused
on assertion and named accordingly.

### Traceability on Failure
Upon assert failure, leave enough information that explains what failed.
This is commonly supported by the test name, the asserted objects and their
content.

The reported failure should point back to the source code (e.g. line number,
traceback) to ease debugging.

### Shared Resources
Shared resources between tests have the potential to leak state changes
between tests, violating the test isolation.

Therefore, whenever possible it would be always preferable to recreate any
resource from scratch for each and every test.

That said, one cannot ignore that some tests (e.g. e2e) pay a heavy cost
when working with resources (reflected by time and/or memory).

In such scenarios, a more pragmatic approach is in order.
Sharing resources across a group of tests with an immutable restriction many
times worth the management effort and results with faster and less costly
tests.

### Continue On Failure
Expect tests to fail, stopping on the first failure may not be informative
enough.
That said, leave behind enough information to debug and troubleshoot the
failure.

### Avoid Dead Tests
Production code has `*tests`. Tests have only themselves.
Do not leave unexecuted code, it rots.
Keep only running tests and helper functions.

Note: Tests that are tracking failing scenarios (bugs, missing features) are
better executed with an expected failure over skipped.


## Parallel Tests

Running tests in parallel can decrease substantially the tests runtime if
the tests are mostly I/O bound.

When tests are running in parallel against a single SUT, several points are
in practice added to the tests:
- Checking how well a test is isolated, as it may now randomly be executed with
  other tests in parallel. The mode in which the tests are executed in parallel
  affect the risk of this point (e.g. multi threaded or multi process).
- Checking how well the SUT is behaving while multiple clients (i.e. the tests)
  challenge its API in parallel.
- In both points above, the randomization is an important factor.
- System logs may be now difficult to examine when debugging and
  troubleshooting failures.

Therefore, an amount of uncertainty is introduced, something that tests try to
avoid in order to be reproducible and consistent in their reported results.

Projects should balance between efficient runtime, covering concurrent access
to the SUT and the risk of uncertainty and debugging effort.


## Workarounds
This section is about patterns that attempt to solve needs in a questionable
manner. I have seen them applied on working projects due to restraints or needs
that solve one or more problems, but introduce many others in parallel.
I would consider most as anti-patterns, but some may argue that any other way
would have been too costly to implement.

### Stop On Failure
When a test fails, stop the execution of the whole suite, clean nothing and
collect all available information from the system. In some cases, leave it
as is for a developer to debug the state in which it is.

The advantage of this approach is that it is very simple to implement. Log and
other data collection is simplified to just capturing a current state after
the tests stopped completely.
It also allows a developer to access the system while all resources and objects
have not been cleaned.

On the other hand, it is very limiting to see the big picture of the test suite
and understand if the problem is specific to a scenario or affects others.

### Clean on Test Setup
The test pattern recognizes that there is a need for a test to start on a
fresh clean setup. However, it suggests not to clean at a test teardown stage
but at a test early setup.

The pattern fits only when the cleanup is generic, i.e. cleanup occurs on
everything that is associated with the tests in general.

Its advantage is also in its simplicity. Individual resources do not need
carrying by the test writer, as an overall cleanup will be issued anyway before
each test is running.

The problem with this approach is that it needs to know in advance what are
the resources the tests will use and it may mask issues which are related to
the resources deletion.

It is worth noting that test isolation is violated by this pattern because
the leftovers of one test is passed to the responsibility of the next one.
Failure to cleanup is harder to troubleshoot and test failure will be
attributed to the wrong test.
