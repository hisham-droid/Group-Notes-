```table-of-contents
```
# Refactoring

- Refactoring = restructuring code so that its external behaviour does not change, but its internal structure improves
	- It is an effective way to remove code smells
![[file-Pasted image 20261008105129-20261008105129213.jpg|325]]

Six Goals of Refactoring: 
1. **Extensibility:** how easily new features can be added
    - Design patterns that help: Strategy, Observer
2. **Performance:** meeting timing/memory limits
    - e.g., choosing better data structures, removing unnecessary assignments and calls
    - Note: performance requirements must already be met before refactoring; refactoring can improve it further
3. **Maintainability:** how easily the system can be modified to fix faults or improve things
    - Some code from the 1970s and 1980s is still running in production today
4. **Readability:** how easily a human can understand the code
    - Code has two audiences: the computer and other humans
    - e.g., good variable names, simpler conditions
5. **Reliability:** the system keeps operating correctly over time (on the slides, skipped in the lecture)
6. **Scalability:** the system keeps performance up as demand grows
    - e.g., a prototype handles 100 users/week; refactor the architecture to handle 100 million

When to refactor: 
- Before adding new functionality: refactor first, then add the feature
- When you find code smells in a code review

Refactor is finished when all of these hold:
- The source code is improved (against one of the six goals)
- No new functionality was created
- All existing tests still pass (if a previously-passing test now fails, you have failed)
- It was done in small increments
- Big architectural changes are pushed to the next sprint, not done as a refactor

# Design Principles 

Why they help
- Design principles do not change what the program does at runtime. 
	- They change how the code is organised.
- Both compilers and humans have to "reason" about code: work out which parts affect which other parts.
	- If everything is tangled together, that reasoning problem is hard
- Good design keeps parts separate, so each part can be understood on its own
	- Makes reasoning easier for both compiler and human engineers
- Overall goal: keep simple things simple, isolate and minimise complexity, reduce coupling

## SOLID
- Set of object-oriented design rules that make software easier to maintain, extend and test
![[file-Pasted image 20261008112024-20261008112024946.jpg|525]]

### Single Responsibility Principle 

Definition:
- A class or method should have only one responsibility
- Equivalently: there should be only one reason for it to change
- Aim: modularisation (building the system from small, independent pieces)
Advantages
- Fewer test cases per class
- Looser coupling
- Greater reusability
Coupling and decoupling
- Coupling = how much one class depends on another
- High coupling = components are tightly tied together; changing one breaks others
- Decoupling = reducing those dependencies
- In a loosely coupled system, you can swap a component for another implementation with the same functionality

Example:
```java
public class InputHandler {           
// Responsibility: read data from a file
    public static List<Double> readData(String filename) { /* ... */ }
}

public class DataProcessor {          
// Responsibility: perform calculations
    public static double calculateAverage(List<Double> data) { /* ... */ }
}

public class OutputRenderer {         
// Responsibility: display the result
    public static void displayResults(double result) { /* ... */ }
}

public class Main {                   x// Ties everything together
    public static void main(String[] args) {
        String filename = "input.txt";
        List<Double> data = InputHandler.readData(filename);
        double result = DataProcessor.calculateAverage(data);
        OutputRenderer.displayResults(result);
    }
}
```
- Most obvious benefit here is that the rendering library only has tobe imported in OutputRenderer
- This is not exactly a DAO however
	- Main calls the classes directly no interface
	- Methods are static, so you cannot swap in a different implementation without editing main 

### Open/Closed Principle (OCP)

Definition: 
- Classes should be open for extension but closed for modification
- When adding functionality, add new classes instead of editing existing, proven ones

Why:
- Statistically, every time you edit existing code you tend to introduce new bugs
- Editing someone else's code is much more expensive than writing a new module

Examples:
Instead of editing a car to fly and sail:
![[file-Pasted image 20261008115100-20261008115100982.jpg|375]]
Use, open and close principle to create new classes instead of editiing the existing car:
```java
public class FlyingCar extends Car {
    private Wing wing;
    public void runAndFly() { /* ... */ super.run(); }
}

public class FlyingSailingCar extends FlyingCar {
    private Sail sail;
    public void runAndFlyAndSail() { /* ... */ }
}
```

![[file-Pasted image 20261008115336-20261008115336118.jpg|500]]

### Liskov Substitution Principle (LSP)

- A class can be replaced with any of its subclasses without disrupting the behaviour of the program
- Plain version: if your code works with a Vehicle, it must still work when handed any specific subclass of Vehicle, with no surprises

```java
public interface Vehicle {
    void turnOnEngine();
    void refuel();
}

public class FuelVehicle implements Vehicle { /* both methods work */ }

public class ElectricVehicle implements Vehicle {
    void turnOnEngine() { throw new Exception("I don't have an engine"); }
    void refuel()       { throw new Exception("I don't consume fuel!"); }
}
```
- LSP Violated here as swapping vehicle for subclass electricvehicle will throw exceptions, meaning program behavior has changed  

Bank Example: 
- We need new account: FixedTermDepositAccount which cannot withdraw
- If it extends Account, its withdraw() would have to throw an exception
	- LSP is then violated
- Subtypes must be substitutable for their supertype without ever having to modify the client code
	- So we should change the type hierarchy to make this compatible with LSP
![[file-Pasted image 20261008123148-20261008123148870.jpg|475]]
- Account no longer promises withdraw, new withdrawableAccount promises it
- With this LSP on FixedTermDepositAccount no longer violated 

LSP Rules:
- Require no more, promise no less

1. Contravariance of method arguments
	- An overriding method's argument types can be the **same or wider** (more general) than the parent
	- Accept at least everything the parent accepted
2. Covariance of return types
	- An overriding method's return type can be the same or narrower (more specific) 
	- e.g., parent returns Number, subclass returns Integer
	- Callers expecting a Number are still happy
3. Exceptions rule
	- the subclass method can throw fewer or narrower exceptions, but not additional or broader ones
	- The ElectricVehicle example break this rule

- Inputs: **get looser** (or stay the same)
- Outputs: **get tighter** (or stay the same)
- Exceptions: **get fewer/narrower**

LSP Property and Method Rules:
- **Precondition:** must be true **before** a method runs (what the method requires from the caller)
- **Postcondition:** guaranteed true **after** the method finishes (what the method promises)
- **Class invariant:** true before **and** after every method call (always true of a valid object)

1. **Preconditions:** a subtype can **weaken** (relax) but not strengthen them
    - Example: parent `withdraw(amount)` requires `amount > 0`. A subclass may accept `amount >= 0`, but may not require `amount > 100`.
2. **Postconditions:** a subtype can **strengthen** but not weaken them
    - Example: parent promises "balance decreases by amount". A subclass may also promise "and a receipt is logged", but may not drop the balance guarantee.
3. **Class invariants:** subtype methods must **maintain or strengthen** the parent's invariants
    - Strengthening means "A" becomes "A AND B" (allowed). Weakening means "A" becomes "A OR B" (not allowed).
    - Example: if `Account` guarantees `balance >= 0`, no subclass may allow a negative balance.
4. **History constraint:** subclass methods must not allow **state changes the parent did not allow**
    - Lecture example: if an integer field can never be incremented in the parent, it can never be incremented in the subclass

### Interface Segregation Principle (ISP)

Definition
- A class should only depend on the interfaces it needs
- Larger interfaces should be broken into smaller ones
- Clients should not be forced to depend on methods they do not use
- Warning sign: a class with a huge number of methods, where situation A uses half and situation B uses the other half. That is probably two classes.

```java
public abstract class Bird {
    abstract int layEggs();
    abstract void fly();
}
public class Pigeon extends Bird { /* lays 3 eggs, flies */ }
```
- However some birds such as a cassowary cannot fly. 
```java
public interface Oviparous { int layEggs(); }   // "oviparous" = egg-laying
public interface Flyable   { void fly(); }

public class Cassowary implements Oviparous { /* layEggs returns 5 */ }
public class Pigeon implements Oviparous, Flyable { /* both */ }
```
- This was also previously violating LSP. 
	- ISP fixes the problem by splitting interfaces; LSP is the rule the original design broke

### Dependency Inversion Principle (DIP)

Definition
- About **decoupling** modules
- When high-level modules (business logic, e.g., Program) depend on low-level modules (details, e.g., JSON/XML file handling)
	- they should depend on abstractions (interfaces) instead

Wall Socket Analogy: 
- The socket is an **interface**
- You only need to know how to plug in your lamp; the electrician only needs to know how to supply the socket
- There is no direct dependency between you and the electrician
- At runtime electricity still flows (the dependency exists while running), but in the **design** the two sides only know about the socket
- Key point: DIP is about **how code is organised**, not about runtime control flow

Example:
- Before two versions of Program, each hard wired to one format
```java
public class Program {
    private final JSON json;
    public void read()  { json.read(); }
    public void write() { json.write(); }
}
```
- After, depends on abstraction via constructor: 
```java
public interface DAO { void read(); void write(); }
public class JSON implements DAO { /* ... */ }
public class XML  implements DAO { /* ... */ }

public class Program {
    private final DAO dao;
    public Program(DAO dao) { this.dao = dao; }
    public void read()  { this.dao.read(); }
    public void write() { this.dao.write(); }
}
```
- `Program` never mentions JSON or XML
- Passing the dependency through the constructor is called dependency injection

## DRY: Don't Repeat Yourself

- Never copy and paste code within a system. If you need to, the design is wrong.
- Extract common logic into one function or class: "write once, use many times"
- Benefits: reusability, maintainability, testability
- Common example: **utility classes**, e.g., Java's `String` and `Math` classes group common operations

## KISS: Keep It Small and Simple

- (Engineers often say "Keep It Simple, Stupid")
- Designs work best when kept simple
- Keep classes and methods small
- Benefits: readability, maintainability, easier testing
- Lecturer's test: if you are scared to open a piece of ordinary business-logic code, the design has failed

# Code Smells

- Code smells are structures in code that indicate violations of fundamental design principles and hurt design quality
- They are not bugs. They are weaknesses that may slow development or increase the risk of bugs later.
- In code review: if a pull request has code smells, reject it and refactor

## Common Smells 

Mysterious names:
- Names that don't tell you what something is or does
- Bad: `a`, `b`, `aa`, `yorn`, `torf`, `lucky1`, `luckyFunction`
- Good practice:
    - Meaningful, unique, descriptive names
    - Complete words rather than acronyms
    - Methods named with verbs (`run()`, `write()`)
    - Follow the language's naming conventions (Oracle's Java naming conventions)

Hardcoding:
- Embedding data directly in source code
	- Instead of reading it from outside (config files, command-line arguments, Android XML resources) or generating it at runtime
- Acceptable to hardcode: 
	- unchanging values like physical/mathematical constants (pi, e)
	- version numbers, static text
- Never hardcode:
	- passwords and other secrets, file paths, arbitrary mappings like "Monday = 7"

Duplicated code:
- Usually happens when several programmers work on different parts at once 
	- e.g., `A.readFileX()` and `B.readFileY()` doing the same thing
- Merge it: shorter, simpler, easier to maintain and test

Long class and long method:
- Too many lines of code
- Happens because people prefer adding to existing classes over creating new ones
- Fix: decompose into smaller classes/methods (usually a sign SRP is broken)

Long parameter list:
- e.g., `method(param1, param2, param3, param4, param5, param6, ...)`
- Fix: group related parameters into objects, 
	- e.g., `method(Object1, Object2)`

Comments:
- Comments should be concise and help readers understand the code
- **The best comment is a good name** (self-documenting code)
- Don't over-comment (e.g., getters and setters)
- Never commit **commented-out code** to production (the lecturer called it a security problem)

Primitive obsession:
- Using primitives (`int`, `float`, `String`) for things that should have their own type
	- `public static final int MONDAY = 1; public static final int TUESDAY = 2;` vs `public enum Day { MONDAY, TUESDAY, WEDNESDAY }`
	- Never use integers for categories; use enums or a class hierarchy

Mutable Objects: 
- A **mutable** object can be changed after it is created
- Risk: you pass an object somewhere, and someone else changes it under you (e.g., an object stored in a `HashSet` whose fields change, so its hash is now stale and it is in the wrong bucket)
- Fix: make **defensive copies** when needed
```java
  public void method(List<Person> persons) {
      List<Person> localPersons = new ArrayList<>(persons);
      // work on localPersons
  }
```

Deadcode:
- Delete unused code, unused files and unneeded parameters
- Every line in production should actually be executed in production
- Improves readability and reduces size

## 7. B-Trees

### 7.1 Context: balance in trees you already know 

The lecturer quizzed the class on this and said it will probably be tested.

- **Perfectly balanced** = every path from the root to a leaf has the same length (about log N)
- **Red-black trees:** not perfectly balanced. The longest root-to-leaf path is at most **twice** the shortest, so height is at most about **2 log2(n)**.
    - Reason: every path has the same number of black nodes, and red nodes can't be adjacent, so at worst every other node on a path is red
- **AVL trees:** not perfectly balanced either. Subtree heights differ by at most 1, giving height at most about **1.44 log2(n)**. They are more tightly balanced than red-black trees.
- **When to use which:**
    - Lots of insertions/deletions: red-black (fewer rotations)
    - Mostly lookups: AVL (shorter paths)
- **B-trees:** the only **perfectly** balanced tree in this course. All leaves are on the same level.

### 7.2 What a B-tree is and why it exists

- A **generalisation of a binary search tree**: nodes can have many children, not just 2
- **Problem with BSTs on disk:** each node is read individually, and reading from secondary memory (hard disk/SSD) is slow
- **B-tree solution:** designed for **block-oriented devices**
    - A disk reads a whole block (page) at a time, so reading 1 key or 100 keys costs about the same
    - So put many keys in each node, and **map each node to one disk block (page)**
    - Conceptually, it is like taking a chunk of a binary search tree (e.g., a 7-node subtree) and squashing it into one node with many children
- Result: very short, wide trees, so very few disk reads

#### Where you see them

- Databases and file systems when data is bigger than RAM
- MySQL uses **B+ trees** (variant where all the actual keys/data are in the leaves and internal nodes only guide the search)
- Lecturer note: if an AI generates Rust code using B-tree maps for in-memory work, that may not be ideal; in memory, red-black or AVL trees are often better choices

### 7.3 Terminology

- **Order m:** the **maximum number of children** a node can have
- **Keys:** the values stored and sorted in a node
- A node with k children holds k - 1 keys, so a node holds **at most m - 1 keys**
- **Page:** one node, stored as one disk block

### 7.4 Structure of a node (general idea)

For m = 5, a full node looks like:

```
        [ k1 | k2 | k3 | k4 ]
       /    |    |    |     \
     p1    p2   p3   p4     p5
```

- p1: keys **less than** k1
- p2: keys strictly **between** k1 and k2
- p3: between k2 and k3
- p4: between k3 and k4
- p5: keys **greater than** k4
- Keys within a node are kept sorted
- Inequalities are **strict**, so **duplicate keys are not allowed** (like a database primary key)

### 7.5 Formal definition (Knuth, 1997) - learn this exactly

A B-tree of order m satisfies:

1. Every node has **at most m children**
2. Every node **except the root and leaves** has **at least ceil(m/2) children**
3. The **root has at least 2 children** (unless it is a leaf, i.e., the whole tree is one node)
4. A non-leaf node with **k children contains k - 1 keys**
5. **All leaves appear on the same level** (this is "perfect balance")

Derived key counts (useful for problems):

|Node type|Min keys|Max keys|
|---|---|---|
|Root|1|m - 1|
|Any other node (internal or leaf)|ceil(m/2) - 1|m - 1|

Examples:

- m = 3: non-root nodes hold 1 to 2 keys (a "2-3 tree")
- m = 4: non-root nodes hold 1 to 3 keys (a "2-3-4 tree")
- m = 5: non-root nodes hold 2 to 4 keys

Note: the lecture's code treats leaves the same as internal nodes for minimum fill (ceil(m/2) - 1 keys). Knuth draws the "leaves" as the null pointers below the bottom row, which is why the formal rule says "except leaves".

### 7.6 Capacity formulas

Convention: **height h** counts edges from root to the bottom level, so a single-node tree has h = 0 and has h + 1 levels.

#### Maximum number of keys

Every node is full (m - 1 keys) and has m children:

- Level 0: 1 node, level 1: m nodes, level 2: m^2 nodes, ..., level h: m^h nodes

```
N_max = (m - 1)(1 + m + m^2 + ... + m^h)
      = (m - 1) * (1 - m^(h+1)) / (1 - m)      [geometric series]
      = m^(h+1) - 1
```

Geometric series used: a + ar + ar^2 + ... + ar^(n-1) = a(1 - r^n)/(1 - r), with a = 1, r = m, n = h + 1. The (m - 1) on top cancels with the (1 - m) on the bottom (after flipping signs).

#### Minimum number of keys

Every node is as empty as allowed:

- Root: 1 key, 2 children
- Level 1: 2 nodes; level 2: 2 * ceil(m/2) nodes; level i: 2 * ceil(m/2)^(i-1) nodes
- Each non-root node has ceil(m/2) - 1 keys

```
k_min = 1 + (ceil(m/2) - 1) * 2 * (1 + ceil(m/2) + ... + ceil(m/2)^(h-1))
      = 1 + (ceil(m/2) - 1) * 2 * (ceil(m/2)^h - 1) / (ceil(m/2) - 1)
      = 1 + 2(ceil(m/2)^h - 1)
      = 2 * ceil(m/2)^h - 1
```

### 7.7 Search

To find key x, starting at the root:

1. Search among the keys in the current page (if m is large, use binary search within the page)
2. If found, done
3. If not found, pick the child to descend into:
    - x < k1: go to the first child (p0)
    - ki < x < k(i+1): go to child pi
    - x > the last key: go to the last child
4. If there is no child (you are at a leaf), **the key is not in the tree**

### 7.8 Insertion

Insertion always happens at a **leaf**. Let p be the leaf where x belongs (found by searching).

- **Case 1: p has fewer than m - 1 keys** (not full)
    - Insert x in sorted position. Done.
- **Case 2: p is full** (would now have m keys = overflow)
    1. Allocate a new page
    2. Split the m keys (including x):
        - The **ceil(m/2) - 1 smallest** keys stay in the old page
        - The **m - ceil(m/2) largest** keys go to the new page
        - The **median** (the key at position ceil(m/2)) is **pushed up** into the parent
    3. If the parent now overflows, **split it too** (repeat upward)
    4. If the **root** splits, create a **new root** containing just the median

#### Slide example (m = 5, so max 4 keys per node)

- Insert 23: goes to leaf [22, 24], which has room, giving [22, 23, 24]
- Insert 46: leaf [41, 42, 45, 47] is full. With 46: [41, 42, 45, 46, 47]
    - Median 45 goes up to the parent: [30, 40, 45]
    - Left keeps [41, 42], new right page gets [46, 47]

#### Worked sequence: insert 10, 30, 50, 70, 90, 20, 40, 60, 80, 100 (m = 5)

```
Insert 10, 30, 50, 70:   [10 30 50 70]                (full, 4 keys)

Insert 90: overflow [10 30 50 70 90], median 50 goes up

                [50]
              /      \
        [10 30]      [70 90]

Insert 20, 40 (left), 60, 80 (right):

                [50]
              /      \
   [10 20 30 40]    [60 70 80 90]

Insert 100: right leaf overflows [60 70 80 90 100], median 80 goes up

                [50 80]
              /    |    \
   [10 20 30 40] [60 70] [90 100]
```


### 7.9 Practice exercise from the slides (with answers) 

Insert: 20, 11, 15, 3, 5, 7, 12, 15, 16, 19, 25, 30, 33, 37, 22, 23, 26, 31, 42, 35, 6, 47 for m = 3 and m = 5. What is the height?

The second 15 is a duplicate, so it is ignored (21 distinct keys).

**m = 5: height 2 (3 levels)**

```
                         [25]
                 /                \
            [11 16]              [31 37]
           /   |    \           /   |   \
  [3 5 6 7] [12 15] [19 20 22 23] [26 30] [33 35] [42 47]
```

Key splits along the way: 5 overflows the first leaf (11 goes up), 19 causes 16 to go up, 33 causes 25 to go up, 31 causes 31 to go up, and finally 47 overflows [33 35 37 42 47] so 37 goes up, which overflows the root [11 16 25 31 37], so 25 becomes the new root.

**m = 3: height 3 (4 levels)**

```
Level 0:                        [19]
Level 1:              [11]                  [25 33]
Level 2:        [5]        [15]       [22]   [30]   [37]
Level 3:   [3] [6 7]   [12] [16]   [20] [23] [26] [31] [35] [42 47]
```

Sanity check using the formulas: m = 5, h = 2 allows 17 to 124 keys, and m = 3, h = 3 allows 15 to 80 keys. 21 keys fits both. Smaller order means a taller tree.

Visualiser to check your own work: [https://www.cs.usfca.edu/~galles/visualization/BTree.html](https://www.cs.usfca.edu/~galles/visualization/BTree.html) (note: it allows duplicates and may split even orders differently).

### 7.10 Removal

#### Step 1: Make sure you are removing from a leaf

- Removal is always performed at a **leaf**
- If the key is in an **internal** node, replace it with either:
    - its **predecessor** (largest key in its left subtree), or
    - its **successor** (smallest key in its right subtree)
- Then delete that predecessor/successor from its leaf instead

Slide example (successor): remove 10 from root [10, 20] with middle child [13, 14, 15]: 13 moves up, giving root [13, 20] and child [14, 15].

#### Step 2: Remove from the leaf

- If the leaf still has at least the minimum number of keys (ceil(m/2) - 1), done
    - Slide example (m = 5): remove 5 from [2, 5, 7, 8], giving [2, 7, 8]. Still has 3 keys, fine.
- Otherwise the leaf is **under capacity** ("underflow") and you must **merge** or **rebalance**

#### Step 3a: Merge

- Use when: the underflowing node + its adjacent sibling + the **parent key between them** all **fit in one node** (at most m - 1 keys)
- Do: combine them into one node; the separating parent key moves **down** into the merged node
- The parent loses a key, so it may now underflow: **repeat upward** as needed
- If the root loses its last key, its single child becomes the new root and the **tree shrinks by one level** (the only way height decreases)

Slide example (m = 5): tree root [10, 20], leaves [2, 5], [13, 14], [22, 24]. Remove 5:

```
Leaf [2] underflows (min is 2 keys for m = 5).
[2] + separator 10 + [13 14] = 4 keys, fits in one node: merge.

         [20]
        /     \
[2 10 13 14]   [22 24]
```

#### Step 3b: Rebalance (borrow / rotate)

- Use when: merging would **not** fit (the sibling has spare keys)
- Do (a rotation through the parent):
    1. Move the **parent separator key down** into the underflowing node
    2. Move the **nearest key from the sibling up** into the parent
        - Sibling on the left: take its **largest** key ("borrow from left")
        - Sibling on the right: take its **smallest** key ("borrow from right")
- The parent keeps the same number of keys, so **no propagation upward**

Slide example (m = 5): root [20], leaves [2, 10, 13, 14] and [22, 24]. Remove 22:

```
[24] underflows. [2 10 13 14] + 20 + [24] = 6 keys > 4: can't merge, so rebalance.
20 comes down to the right, 14 (largest on the left) goes up.

        [14]
       /    \
[2 10 13]   [20 24]
```

#### Lecture worked examples (m = 5)

Remove 70 from:

```
           [50 80]
         /    |    \
[10 20 30] [60 70] [90 100]
```

- [60] underflows. Left option: [10 20 30] + 50 + [60] = 5 keys, too many. Right option: [60] + 80 + [90 100] = 4 keys, fits. **Merge with the right.**

```
           [50]
          /    \
 [10 20 30]   [60 80 90 100]
```

Then remove 20 (tree is now [10 20] | [60 80 90 100] in the slide version):

- [10] underflows. [10] + 50 + [60 80 90 100] = 6 keys, can't merge. **Rebalance (borrow from right):** 50 comes down, 60 goes up.

```
          [60]
         /    \
    [10 50]   [80 90 100]
```

#### Choosing between merge and rebalance

- Lecture's convention: **merge if it fits, otherwise borrow**
- If both a left and right sibling exist, pick one by a fixed rule (always try left first, or always right first)
    - Being **deterministic** matters: random choices make code impossible to test, debug or benchmark

**[Watch out]** Slide 78 says merge when the two pages "together have **less than m** keys". Taken literally that can overflow. Example with m = 5: underflowing page has 1 key, sibling has 3. Together that is 4 (< 5), but adding the parent key gives 5 keys, more than the 4 allowed. The safe rule (and what the lecturer said aloud) is: **merge only if both pages plus the parent separator fit in one node (at most m - 1 keys); otherwise rebalance.**