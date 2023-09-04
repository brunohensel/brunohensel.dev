---
title: "The Downsides of Excessive Mocks and Stubs in Unit Testing"
date: 2023-08-19T08:09:57+02:00
draft: true
---

## The Downsides of Excessive Mocks and Stubs in Unit Testing

As commonly known, unit testing is a crucial part of software development. The use of mocks and stubs has become standard practice to isolate components and ensure reliable tests (London School). However, it's imperative to recognize the fragility and potential pitfalls associated with excessive mock and stub usage.

## Clarifying Mocks and Stubs:

Before delving into the drawbacks, let's clarify the terms "mock" and "stub." A mock is typically created with the assistance of a mocking framework (e.g., Mockito) and helps emulate and examine interactions between the System Under Test (SUT) and its dependencies. On the other hand, stubs are constructs that assist with interactions between the SUT and its dependencies to provide specific data.

## The Fragility of Excessive Mocks and Stubs:

At many workplaces (Vinted is definitely not an exception), mocking libraries are used in virtually every unit test, leading to repetitive setup code. This can hinder scalability and create maintenance overhead as developers keep repeating the same configurations.

Moreover, the ease of constructing the SUT using mocking libraries might tempt developers to overlook the importance of a good design. For instance violating the Interface Segregation Principle makes very painfull to create a Fake (another type of test doubles, I will cover it in next posts) of it, on the other hand, it's a breeze to do that with a mocking library.

## Complex Test Setup and Review Challenges:

Test files often end up with extensive setup code, making it challenging for reviewers to grasp the test's intent if they are unfamiliar with the production code. The back-and-forth between test and production files can be time-consuming and inefficient. With that much setup in the way, it's hard to be as declarative as possible.

## Fragile Interaction Testing:

Interaction testing, which verifies whether specific dependencies were invoked, can couple tests tightly to implementation details. This results in brittle tests that require frequent updates whenever implementation details change, undermining the value of automation.

_If a set of tests needs to be manually tweaked by engineers for each change, calling it an "automated test suite" is a bit of a stretch!_ (Software Engineering at Google, p.223)

## Incomplete Test Coverage:

Over-reliance on interaction testing can lead to incomplete test coverage, as it focuses solely on verifying dependency interactions while potentially neglecting the behavior of the SUT itself.

## Risks of Stubbing External Functions:

Stubbing functions from external sources that are not owned or fully understood can lead to a mismatch between the stubbed behavior and the actual implementation. This poses a risk of breaking present or future preconditions, invariants, or postconditions in the external function.

Example Illustration:

```kotlin
// Example of Stubbing a Function - MyClassTest

class Calculator {
    fun sum(a: Int, b: Int): Int {
        return abs(a + b) // always returns a positive integer
    }
}

class MyClassTest {
    private val calc: Calculator = mock()
    private val sut: MyClass = MyClass(calc)

    @Test
    fun testAdd() {
        whenever(calc.sum(1, -3)).doReturn(2) // Stubbing the sum function

        assertEquals(sut.getNewValue(1, -3), 2)
    }
}
```

The test passes. But by stubbing the function `sum` we are forced to kind of duplicate the details of the contract, and there is no way to guarantee that it has or will have fidelity to the real implementation. Just by reading the signature of the `sum` method, there is no guaratee that this function always returns positive integers. More about depending on [implicit interface behavior](https://www.hyrumslaw.com/)

Times went by and the owner of the `Calc#sum` method decides to change the postcondition of always returning positive integers to now also returns negative values. The owner updates their own test suite and runs the entire test suite of the project (assuming that all code belongs to the same repo). The worst happened, `MyClassTest#test_add` still passes!, giving a false feeling of safety. If always have a positive number is expected but not explicitly promessed by the contract, you should've written a test for it (The Beyoncé Rule).

## Conclusion:

Excessive use of mocks and stubs in unit testing can introduce fragility, hinder maintainability, and lead to incomplete test coverage. Being aware of these downsides is crucial for fostering a robust and reliable testing strategy. In the next blog post, we will explore guidelines and best practices for using test doubles more effectively, ensuring high-quality test suites that stand the test of time.

[if everything is mocked, are we really testing the production code?](https://github.com/mockito/mockito/wiki/How-to-write-good-tests#dont-mock-everything-its-an-anti-pattern)

## Sources

- Software Engineering at Google ([currently available to read online](https://abseil.io/resources/swe-book))
- Efective Software Testing
- Unit Testing (Principles, Practices, and Patterns)