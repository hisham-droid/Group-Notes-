```table-of-contents
```
Introduction: 
- Writing unit tests reduces debugging time and reduces bugs
- Programmers job includes:
	- writing the code
	- writing the tests that show the code does the right thing
	- (ideally) documenting why it's correct
- Software testing contains a lot of things where the goal is to verify that software behaves as expected 
	- Does not prove that code is correct
	- Testing checks specific examples, and may miss cases
- Formal verification is harder task of producing mathematical proofs that an algorithm meets it's specifications 

# Validation and Verification 

Validation:
- "Building the right thing"
- Certifies that the system meets the customer's needs
- Checks whether the developer is building the correct product
- Test cases to test the requirement specifications
- E.g: functional testing, usability testing, performance testing

Verification: 
- "Building the thing right"
- Checks whether each function in the implementation works correctly and complies with the specification
- Checks the system against the design
- Certifies the quality of the system
- Example: reviews, code review, test coverage, defensive verification
- This course focuses on automatic verification 

# Testing Approaches 

## Black-Box Testing 

- Tests created purely on functional requirements of code 
- You don't know internals, input goes in, output goes out
- Three types of cases to be tested: 
	- Normal functioning
	- Boundary cases
	- Cases outside requirements and robustness (invalid input, empty input, huge numbers, negative withdrawals)

## White-Box Testing

- You know the code and build tests based on it
- This lets you target test cases at specific if conditions, loops etc
- Normally aims for good coverage, meaning every part of the code gets executed by some test
	- E.g: if the code has if (total >= 40) … else …, pick inputs that hit both branches

# Unit Tests and JUnit

Unit Testing:
- Unit: smallest testable part of software
	- In OOP this is often called a method
	- Some inputs, and single output
- Unit testing: looking for errors in a subsystem in isolation 
- In Java, we can use JUnit to execute automated unit tests
- Unit testing is often white-box, because you test the code you just wrote.

- We use JUnit 4 in this course
- No public static void main in a test class
	- JUnit's runner has it's own main, which scans for methods with @Test and runs them

Basic Framework with JUnit: 
- For a class Foo, create a test class FooTest containing the test-case methods 
- Each method checks particular results and passes or fails 
- Use assertions to check things you expect to be true
	- If assertion fails, test fails 

```java
import org.junit.Test;

public class SomethingTest {
    @Test                       // flags this method as a test case
    public void testSomething() {
        ...
    }
}
```
- All @Test methods run when JUnit runs the class

## Assert Methods

![[file-Pasted image 20261006014730-20261006014730551.jpg|550]]
- Each assertion can also take a message as the first argument to show on failure 
	- `assertEquals(String message, expected, actual)`

### assertEquals vs assertSame
- assertEquals asserts that two objects are equal
	- Calls equals()
	- .equals uses whatever logic the class defines
	- If .equals is not overridden for the objects type, it still compares references
- assertSame asserts that two objects refer to the same object
	- Uses `==` which compares refereces (is it the same object in memory?)
	- == on objects means the same object. Two new objects with identical fields are still !=.
- Example
	- ![[file-Pasted image 20261006015210-20261006015210095.jpg|475]]
		- Identical literal strings share one object in Java, new string forces a new object
		- Therefore, never compare strings with `==.` Use .equals().

###  Floating Points
- Some values can't be represented exactly in IEEE floating point, when they are representations of infinite values 
- This creates a little errors, which need a tolerance delta
```java
assertEquals(expected, actual, delta)   // doubles or floats

assertEquals(0.3333333, 1.0/3.0, 0.0);        // FAIL
assertEquals(0.3333333, 1.0/3.0, 0.0000001);  // PASS
```

### Example of using JUnit
Assume we create class:
```java
public class MyMath {
    int add(int a, int b) { return a + b; }
}
```
We can create a test class for it:
```java
import static org.junit.Assert.*;
import org.junit.Test;

class MyMathTest {
    private MyMath math;

    @Test
    public void testAdd() {
        math = new MyMath();
        assertEquals(4, math.add(2, 2));
    }
}
```

## Resource Reuse/Release

- Method tagged with @Before runs before each @Test method
	- Uses: initialisation, like creating objects or arrays
	- Like a constructor for each test
- method tagged with @After runs after each @Test method
	- Uses: cleanup, setting objects to null, freeing resources

Example: 
![[file-Pasted image 20261006020308-20261006020308771.jpg|525]]

- Similarly, @BeforeClass / @AfterClass run once per test class
	- Runs once before (or after) all the tests in the class.
	- Must be static. It is like a static initialiser.
	- Before use: initization code, establish db connection
	- After use: cleanup code, freeing resources, close connection

## Additional  Tags

### Testing for Exceptions 

```java
@Test(expected = ArithmeticException.class)
public void exceptionTesting() {
    math.divide(1, 0);
}
```
- This test passes if that exception is thrown, fails if it isn't 
	- If an error of class ArithmeticException is thrown, only then will this test pass
- This is used to test expected errors 

### Testing with Timeout 

```java
@Test(timeout = 1000)    // in milliseconds
public void timeoutNotExceeded() {
    assertEquals(5, math.add(2, 3));
}
```
- Fails if the test doesn't finish within 1 second.
- Used to guarantee performance or catch infinite loops
- JUnit 4 has no default timeout, so you must set value yourself

### Parameterized Tests

- Used when we need to use the same input across different test methods
- Method to implement:
	- Annotate the class with `@RunWith(Parameterized.class)`
	- Write a **`static`** method annotated `@Parameters` that returns a collection of parameter arrays.
	- Map each array position to a public field with `@Parameter(index)`

```java
@RunWith(Parameterized.class)
public class MyMathParameterizedTest {

    @Parameters
    public static Collection<Object[]> data() {
        return Arrays.asList(new Object[][] { 
        {0.7f, 0.3f, 1}, 
        {0.4f, 0.2f, 0} 
        });
    }

    @Parameter(0) public float a;        // 1st entry of each array
    @Parameter(1) public float b;        // 2nd entry
    @Parameter(2) public int expected;   // 3rd entry

    @Test
    public void test() {
        MyMath math = new MyMath();
        assertEquals(expected, math.sumAndFloor(a, b));
    }
}
```
- The test runs once per row:
	- (0.7, 0.3) → expects 1
	- (0.4, 0.2) → expects 0
- For the parameters the method name (data) doesn't matter as long as it has a tag
	- Public static is required, as JUnit calls data before any test object exists and a static method can be called on the class itself 
	- `Collection<Object[]> data`
		- Collection is the list of test runs, each element is one run
		- `Object[]` is one row of arguments for that run 
		- Both of these together describe the shape of the returned data. 
	- `Arrays.asList(new Object[][]{});`
		- Creates literal 2d array where each inner {...} is one row
		- Arrays.asList wraps it into a `List<Object[]>`, matching return type is a list is a collection 
- By putting an f behind the number, it stores it a float
	- If we didn't do this, it would be made a double, which would give an error

### Test Suites 

- One class that runs several test classes together
	- This can be useful for
		- Grouping tests by feature
		- run a quick subset (for example as a CI pipeline before commit)
		- Share expensive setups

```java
import org.junit.runner.RunWith;
import org.junit.runners.Suite;

@RunWith(Suite.class)
@Suite.SuiteClasses({ 
	MyMathTest.class, 
	TreeTest.class, 
	UserSessionTest.class 
}) 
public class FeatureTestSuite { 
// the class remains empty 
}
```

- Just like Parameterized tests, we add RunWith to stop using the default JUnit runner
- JUnit then instead of looking for @Test reads the list of classes in @Suite.SuiteClasses and runs them one by one in order
- The class body should be empty and is put there to comply with java syntax

## Testing Tips

General Tips:
- You cannot test every possible input, so choose limited set of tests that are likely to find bugs.
- Think about boundary cases:
    - positive, zero and negative numbers
    - very large values
    - just below, at, and just above a limit
- Think about empty and error cases:
    - `0`, `-1`, `null`
    - an empty list or array
- Test behaviour in combination:
    - `add` may work on its own but fail after calling `subtract`
    - Make multiple calls: `size` might only fail on the second call

Unit Test Guidelines:
- Test one thing per test method
    - 10 small tests are better than 1 test that is 10× as large.
- Have few asserts per test (ideally 1)
    - The first failing assert stops the test, so you won't know whether later asserts would also fail.
- Avoid logic in tests
    - Minimise `if`/`else`, loops and `switch`. Avoid `try`/`catch`.
    - If you need logic, you're probably testing too much. Split it up.
- Torture (stress) tests are fine, but only in addition to simple tests.

JUnit Tips:
- Tests need failure atomicity, meaning you can tell exactly what failed:
	- Give each test a clear, long, descriptive name
	- Give assertions clear messages
	- Write many small tests, not one big one
- Test expected errors and exceptions
- Add a timeout to every test, e.g. @Test(timeout=1000)
- Choose a descriptive assert method, not always assertTrue
- Choose representative test cases from equivalent input classes
- Avoid complex logic in tests

### Test Driven Development
- TDD: write the tests first, then write code until they pass
- Example: add a log method to MyMath, writing tests derived from the Math.log Javadoc before implementing it.

```java
// Case a < 0 or a is NaN → NaN
assertEquals(Double.NaN, math.log(-1), 0.000001);
assertEquals(Double.NaN, math.log(Double.NaN), 0.000001);
// Case a is +infinity → +infinity
assertEquals(Double.POSITIVE_INFINITY, math.log(Double.POSITIVE_INFINITY), 0.000001);
// Case a = 0.0 or -0.0 → -infinity
assertEquals(Double.NEGATIVE_INFINITY, math.log(0.0), 0.000001);
assertEquals(Double.NEGATIVE_INFINITY, math.log(-0.0), 0.000001);
// Otherwise → natural log
assertEquals(1, math.log(Math.E), 0.000001);
```

# Integration Testing

- Testing to check if multiple units work correctly when interacting with each other
	- Individual units are combined and tested as a group
	- Interactions between units are clearly defined
	- Both black-box or white-box can be used
- Done after unit testing
	- Not done at the same time, as you won't be tell if it's a bug inside unit or with how the units interact
- A unit-testing framework (e.g. JUnit) can still be used.

Mocking:
- While building, some modules may not exist yet
- Common practice to replace those with mock modules that return hard-coded expected results
- Replaced when real modules arrive

Two approaches to doing integration testing:
![[file-Pasted image 20261006032914-20261006032914776.jpg|425]]
- Bottom-up
	- Test lowest-level components first, use them to test higher-level components
	- Repeat up to the top of hierarchy
	- Only works when most modules at a level are ready
- Top-down 
	- Test top integrated modules first
	- Then test each branch, step by step to the end of related module 

Example of testing the session, state transitions and DAO/Database working together:
```java
@Before
public void before() {
    UserActivityDao.getInstance().deleteAll();   // reset shared DB state
}

@Test(timeout = 1000)
public void testUserStateLoginActionFail() {
    UserSession userSession = new UserSession();
    boolean login = userSession.login("admin", "1233");   // wrong password
    assertFalse(login);
    assertNull(userSession.username);
    // Every action should be rejected while logged out:
    UserActivity userActivity = userSession.createPost("abc");
    boolean likePost = userSession.likePost(1);
    List<ActionCount> actionCountList = userSession.generateActionCountReport();
    List<UserActivity> allPosts = userSession.findAllPosts();
    boolean logout = userSession.logout();
    assertNull(userActivity);
    assertFalse(likePost);
    assertFalse(logout);
    assertNull(actionCountList);
    assertNull(allPosts);
}
```

Difference:
![[file-Pasted image 20261006035334-20261006035334813.jpg|475]]

- We won't write integration tests in this course

# System Testing

- Checks that the entire system works within the environment it will be deployed in
- Checks that all hardware and software parts work with external components under normal loads
- Ideally done on a duplicate of the real hardware/software. When that isn't feasible, it is done "live"
- Examples:
    - Does the app work on an old iPhone?
    - Does it work on a different Android version?
    - Does it work on a tiny screen?
    - Big companies keep racks of real phones and test every release on them.

# Code Coverage 

- Coverage measures what percentage of the code has been executed by the test suite
	- This can help you find new test cases
- Coverage should ideally be 100%

Note:
- Coverage tells you that code ran, not that behaviour was tested, and number and quality of tests matters more than the coverage percentage
- However coverage below 100% is still a useful signal to find missing test cases

## Statement Complete

- Tests are statement complete when every statement is executed at least once by the tests
- Hard to reach 100%; about 80% is typical. Some code only runs in rare conditions.
- It may not test what it really should, because it only cares about running statements. You still have to analyse what's being covered.
- IntelliJ's coverage tool reports lines, not statements, because it works from bytecode, which has no clear statement information.

![[file-Pasted image 20261006131102-20261006131102866.jpg|500]]
- There are two statements: 
	- someMethod(true, true, true) -> runs if(a), X, if(c), W
	- ![[file-Pasted image 20261006131946-20261006131946575.jpg|425]]
	- someMethod(false, true, true) -> runs `if(a)`, Y, `if(b)`, Z, `if(c)`, W
	- ![[file-Pasted image 20261006132002-20261006132002817.jpg|425]]
- These two test cases together cover all 7 statements and are therefore enough to be statement complete

This example can be turned into a Control Flow Graph
- CFG is a graph of every way execution can move through a method
- Coverage tools build a CFG and tick off which nodes and edges your tests visit.
![[file-Pasted image 20261006131720-20261006131720073.jpg|500]]

## Branch Complete 

- Definition: every branch outcome (both true and false of every if for example) is taken at least once 
	- This includes "invisible" false branches that run no statements
- Formula: `Branch coverage = 100 × (number of executed branches) / (total number of branches)`
- Minimal Branch complete set:
	- ![[file-Pasted image 20261006132607-20261006132607245.jpg]]
- Usually needs more tests than statement coverage, so it is more expensive
- Branch coverage subsumes statement coverage
	- 100% branch implies 100% statement, because taking every branch reaches every statement
- Still doesn't guarantee bug-free code, but can catch defects statement coverage misses

## Condition Complete 

- Applies when a decision has several clauses
	- e.g. if (A || B), where A is clause 1 and B is clause 2
- Each individual clause has been evaluated to both true and false
- Formula: `Condition coverage = 100 × (clauses evaluated to both T and F) / (total number of clauses)`

![[file-Pasted image 20261006133335-20261006133335474.jpg|475]]
- We have achieved 100% condition coverage with two tests:
	- A = TRUE, B = FALSE
	- A = FALSE, B = TRUE
- However, both tests make A || B true, therefore stmt Y is never tested
	- Hence, condition coverage doesn't subsume branch coverage
- Ideally one should write tests so each condition that makes up a decision has a true and a false outcome at least once, so that branch coverage can be achieved as well
- Java short circuits:
	- A || B: if A is true, B is never evaluated.
	- `A && B`: if A is false, B is never evaluated
	- This complicates short circuiting, as some tests may never really be evaluated 

## Multiple Condition Complete

- Considers how the compiler actually evaluates multiple conditions (short-circuiting). Each condition, and each branch, is exercised.
- Formula: `Multiple condition coverage = 100 × (conditions and branches evaluated) / (total conditions and branches)`
![[file-Pasted image 20261006133938-20261006133938262.jpg|525]]
- 100% multiple condition coverage implies 100% decision (branch) AND 100% condition coverage
	- More thorough than other, but does not guarantee path coverage
	- "Branch" and "multiple condition" are often used interchangeably but mean different things, use this course's definition 

## Path Complete 
- Every possible path through the CFG is executed 
- Formula: `Path coverage = 100 × (number of executed paths) / (total number of paths)`
![[file-Pasted image 20261006134236-20261006134236551.jpg|500]]
- Branch complete was only 3 tests, so 100% branch coverage does not imply path coverage. 
- Therefore it also subsumes path coverage 
- Every combination counts as a path, even when there's no else
- Very expensive. Covering all paths in medium-sized software is often infeasible.
	- Usually reserved for critical parts of a system 

## Conclusion

![[file-Pasted image 20261006134744-20261006134744385.jpg|500]]

Code that doesn't usually get covered:
- Usually code that only runs in exceptional circumstances:
	- low memory, full disk
	- lost connections, bad registry data
- This can be solved by using fault injection tools, that introduce faults at compile time and runtime
	- Compile-time examples:
		- Mutation: change code, e.g. i=i+1 becomes i=i-1, and check that your tests catch it
		- Code insertion: add code that produces faults.
	- Introduce runtime examples:
		- Memory corruption: RAM, I/O map
		- Network faults: lost or reordered packets

Instrumentation:
- Code instrumentation: extra code is added to monitor the program's execution
- Used to observe:
	- functional aspects, like function calls and return values
	- data aspects, like variable changes and memory allocation
- The collected information is used to measure test coverage
- Example:
	- ![[file-Pasted image 20261006135020-20261006135020393.jpg|625]]

Covering while/for loops in testing:
- How do we fully cover a loop
	- run it zero times
	- run it exactly once
	- run it n times, where n is a small, typical value
	- run it m times, where m is the maximum possible
	- (optional) run it m − 1 and m + 1 times
![[file-Pasted image 20261006135151-20261006135151443.jpg|500]]
