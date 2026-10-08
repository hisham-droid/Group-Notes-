```table-of-contents
```
# Data Structures
- It doesn't matter how efficent the language is, if the chosen data structure is not appropriate  
- Data structures are universal across languages and facilitate data management and retrieval 
- Most common structures are availaible in the Java Collections framework
	- Rarely will we implement from scratch, but we should understand how they work

Two main types:
![[file-Pasted image 20260918153305-20260918153305565.jpg]]
- Primitive are predefined and stored directly as a value on the stack
- Non-primitive are the actual data structures
	- Whole object gets allocated on the heap, and you hold a pointer to it 

# Linear Data Structures 

- Nodes are arranged in a sequence, one after the other

Arrays:
- Needs one predefined block of memory
- Must decide the size up front
- Upside: It's always sitting there, instant random access, This is O(1)
- Downside: Space is fixed, if not full you have wasted memory 

Linked List
- Does not need a predefined block
- Each node stores a value plus a pointer to the next node.
- Upside: Can grow dynamically, allocate new node when you need it
- Downside: to reach an k item, you must hop through k nodes. O(n)

- Always trade-off between time and space, you must choose which is more important

# Non-Linear Data Structures

- Trees and Graphs link node to 2+ other nodes
- There is no single sequence 
- Harder to implement than linear
- Memory grows dynamically via pointers, therefore use of memory is more effective

Say you have 5 elements
- In a list reaching the last can take 4 hops. Worst case, O(N)
- In a balanced tree, deepest node only 2 levels down
- A tree gives you the memory efficiency of a linked list (only allocate what you use) without having to walk past every element to find something

![[file-Pasted image 20260918154443-20260918154443753.jpg]]

Big O hieracry (best to worst):
`O(1)` < `O(log n)` < `O(n)` < `O(n log n)` < `O(n²)` < exponential

# Tree Data Sturctures 

![[file-Pasted image 20260918161024-20260918161024109.jpg|500]]
- Node stores a tuple of attributes and pointers to child nodes 
- Edge is a pointer from one node to another, they are directed and one way
![[file-Pasted image 20260918161148-20260918161148513.jpg|525]]

## BST
- BST: tree with at most two children for each node
	- Left child node is smaller than parent node
	- Right child node is greater than parent node
- Nodes of (key,value) pairs
	- Keys must be sortable (numbers or strings)
	- Edges relate the keys 
- Supports dynamic set ops: Search, Minimum, Maximum, Predecessor, Successor, Insert, Delete.

Node structure:
![[file-Pasted image 20260919144957-20260919144957216.jpg|575]]
- Stored keys must always follow the greater=right, smaller=left property

### Traversing the tree: 
- Three recursive ways to process whole tree
- In Order Traversal
	- ![[file-Pasted image 20260919145233-20260919145233576.jpg|475]]
	  - This would print: **12, 18, 24, 26, 27, 28, 56, 190, 200, 213**
	  - This gives you the ascending order
- Pre-order — Root, Left, Right
	- ![[file-Pasted image 20260919145533-20260919145533770.jpg|500]]
	- **56, 26, 18, 12, 24, 28, 27, 200, 190, 213**
- Post-order — Left, Right, Root
	- ```
	  Postorder-Tree-Walk(x) 
	  1. if x ≠ null 
	  2. then Postorder-Tree-Walk(left[x]) 
	  3. Postorder-Tree-Walk(right[x]) 
	  4. print key[x]
	  ```
	  - **12, 24, 18, 27, 28, 26, 190, 213, 200, 56**

Side note:
- Breadth-first (level by level) uses a queue: 
	- Visit a node, push its children, dequeue the next.
- Depth-first (dive deep first) uses a stack.
- For graphs you'd also need a visited set to avoid cycles
	- But trees can't revisit a node, so no visited set needed.

### Insertion 
- Ensure the binary-search-tree property holds after insertion
- A new key is always inserted at the leaf node
![[file-Pasted image 20260919150056-20260919150056531.jpg|500]]
- y is used to store the parent, for when we need to create node
- z is the key being inserted
- x finds the position of the new node
### Predecessor and Successor 
- Successor of node x, is node y such that key of y is the smallest key greater than key of x
	- Successor of largest key is null
- Predecessor of x, is the node with the largest key smaller than key of x

Successor
- Has two cases 
	- x has a right subtree: minimum of that right subtree (go right once and then as far left as possible)
	- x has no right subtree: walk up tree (successor is first ancestor you reach by going up left, lowest ancestor whose left child is also an ancestor of x): e.g for 28 -> 56
![[file-Pasted image 20260924235611-20260924235611996.jpg|475]]

Predecessor
- Symmetric to Successor, just swap right with left, and minimum with maximum 
![[file-Pasted image 20260924235859-20260924235900005.jpg|500]]

### Deletion
- Tricky because you have repair tree after deletion. Has three cases 
	- Case 0 — no children (a leaf): just remove it. Nothing to repair.
		- Simply remove x
	- Case 1 — one child: "pull the child up" to take the node's place. Whole subtree remains the same and therefore correct
		- Make `p[x.child] point to p[x]`
	- Case 2 — two children: you can't just pull one child up
		- Instead, find the node's successor (or predecessor)
		- copy its value into the node being deleted
		- then delete that successor from the subtree
		- and the successor always has 0 or 1 child, so it reduces to case 0 or case 1.
![[file-Pasted image 20260925000734-20260925000734993.jpg|500]]
![[03-bst-delete-case0.jpg|500]]
![[03-bst-delete-case1.jpg|500]]
![[03-bst-delete-case2.jpg|500]]

## Red-Black Tree

- If we insert `6, 5, 4, 3, 2, 1` into a plain BST, in that order, every node becomes a left-child and tree depth grows linearly, making it a linked list/unbalanced tree
![[file-Pasted image 20260925001800-20260925001800339.jpg|425]]
This is fixed by a balanced search tree:
- Remains binary search tree
- But height is O(Log n) guaranteed for n items 
	- Height = maximum number of edges from root to a node
- E.g AVL and red-black trees

Red-Black tree is not balanced, but close to balanced 
- It has seem fields as BST + a colour field which is either red or black 
- Four properties:
	- 1: Every node is either red or black
	- 2: The root and the leaves (NULL nodes) are black
		- NULL/empty children count as real black leaves, so every "real" node effectively has 2 children.
	- 3: If a node is red, both its children are black.
		- You can never have two reds in a row
	- 4: Every path from a given node down to a descendant leaf contains the same number of black nodes
- This works as reds can't stack, so the longest possible path (alternating red black) is at most twice the shortest (all black), meaning tree can never get 2x out of balance, ensuring height is O(log n)
![[file-Pasted image 20260925002937-20260925002937502.jpg|500]]
- This is balanced as every path from 13 to descendant leaf nil is always 3

### Insertion
![[file-Pasted image 20260925011405-20260925011405310.jpg|200]]
![[file-Pasted image 20260925011351-20260925011351381.jpg|200]]
- **Insert 8** → drops in cleanly, coloured red, no property broken. 
- **Insert 11** → its new spot can't be red (parent 12 is red → breaks #3) 
	- Can't be black (adds a black node to one path → breaks #4)
	- **Fix: recolour** the surrounding nodes. (12 becomes black, 9 becomes red)
- Insert 10 → no colour works at all
	- the tree is too imbalanced for recolouring to save it.
	- **Fix: change the tree structure** (rotate), then recolour.
- Therefore we can sum it down into simple insertion psudeocode: 
```
Insert x into the tree, always colour x RED
Only red-black property #3 might be violated if we do this
    → it is violated only if p[x] is also red
    → if so, push the violation UP the tree until it reaches
      a place where it can be fixed (by recolouring / rotating)
Total time: O(lg n)
```

#### Rotation
![[file-Pasted image 20260925011752-20260925011752564.jpg|475]]
![[04-rotation.jpg|500]]
- When tree becomes too unbalanced, we can rotate by switching the root to the subtree with larger subtrees 
- In this case, x becomes the new root, and y becomes a right child of it (satisfying BST property)
- x keeps left child, y becomes new right child, with x's right child becoming y's left child
- This runs in O(1)

Possible insertion cases:
- Following the psudocode pattern we wrote, new node is always inserted red
- This works in most cases except when parent is also red
- When this happens, we have three possible insertion cases, and uncles colour decides which case to use
- Suppose parent (P) of the inserted node is left child of grandparent node (G), uncle is other child of grandparent (U)
![[file-Pasted image 20260925013212-20260925013212139.jpg|165]]
![[file-Pasted image 20260925013222-20260925013222052.jpg|475]]

In the following diagrams: C is GP, A is P, B is X and D is U. 

Case 1: Uncle is red -> Recolour 
- G is black, two red children. Swap colours so G becomes red, and two children become black
![[file-Pasted image 20260925013343-20260925013343060.jpg|500]]![[04-rb-case1.jpg|500]]
```java
if (y->colour == RED)          // y = the uncle
   x->p->colour    = BLACK;    // x's parent  → black
   y->colour       = BLACK;    // uncle   → black
   x->p->p->colour = RED;      // x's grandparent → red
   x = x->p->p;                // move up: grandparent is the new x
```
- Grandparent becomes new x if G is not the root, as G's parent may also be red, posing same problem one level up 
- If G is the root, simply recolour it to be black (Rule 2)

Case 3: Uncle is black & x is left child -> recolour + right rotate
![[file-Pasted image 20260925015355-20260925015355648.jpg|450]]
![[04-rb-case3.jpg|400]]
 - If U is black, making P black would give left side an extra black node (breaking 4)
- This means, we have to change tree shape
- P moves up to the top and becomes black, x and G hang below it as red children

Case 2: Uncle is black & x is right child -> Rotate
![[file-Pasted image 20260925015157-20260925015157578.jpg|450]]![[04-rb-case2.jpg|500]]
- Unlike Case 3, cannot be fixed by simple rotation
- First rotate at P, to turn bend into straight line, now it is Case 3, and can be fixed like that

There are also cases 4, 5, 6. 
- Cases 1-3 assume x's parent is a left child. 
- If parent is a right child, cases 4-6 are symmetric. 
- Swap every left for right

- RB Tree deletion not examinable

Black-height guarantee:
- Black-height bh(x) is the number of black nodes on any path from x (not counting x itself) down to a leaf (the NULL leaf is counted)
	- Property 4 guarantees this number is well-defined
- A node of height h has black-height ≥ h/2 
	- Because reds can't stack, so at least half the nodes on a path are black
- **Theorem:** a red-black tree with `n` internal nodes has height **`h ≤ 2·lg(n + 1)`**.
- Height is O(log n) is guaranteed
	- Because of this the worst case time for all operations such as insert, search, min, max, successor, predecessor and delete on tree have O(log n)

## AVLTree

- AVL (Adelson-Velskii & Landis) Tree is a BST where at every node the left and right subtree differ in height by at most 1. 
- After every insert or delete, tree is checked and fixed with one or two rotations

- Each node has a balance factor
- `bf(n) = height(n.left) − height(n.right)`
	- Height: number of edges on the longest path down to a leaf
		- Single lead node has height 0
		- Empty subtree (null) has height -1
	- bf of 0, means tree is even
	- +1 means left side is 1 taller (left-heavy)
	- -1 means right side is 1 taller (right-heavy)
	- Every node in AVL must have bf of -1, 0 or 1. 
	- +2, -2 means tree is broken, and must be rebalanced 

![[05-avl-bf-example.jpg|350]]
### Fixing an AVL Tree

Every AVL fix follows the same three steps:
1. Find lowest unbalanced node: 
	- Walk up from where you inserted or deleted. 
	- First node with bf of +- 2 is what needs to be fixed and called n
2. Read two signs: bf(n) and bf of n's child on the heavy side
3. Rotate
	- If both are the same sign: (`+2`/`+1` or `−2`/`−1`)
		- Path is straight line, one rotation at n needed from the heavy side
	- If both are opposite sign:(`+2`/`−1` or `−2`/`+1`)
		- Path is zig-zag, two rotations needed
		- First rotate child to straighten it, then rotate n

#### Rotations
- Same as in Red-Black tree
- Only change few pointers so O(1), keep BST order after rotation 

Single right rotation:
- Tree leans left in straight line, y is +2, left child x is +1
![[file-Pasted image 20261003150614-20261003150614636.jpg|475]]
- `x` goes up to the top.
- `y` drops down to be `x`'s right child.
- `B` was `x`'s right child; it becomes `y`'s **left** child. (Every key in `B` is between `x` and `y`, so it fits there.)

Single left rotation:
- Exact mirror, x is -2, right child y is -1
![[file-Pasted image 20261003150727-20261003150727776.jpg|500]]

Double rotations:
- Left Right: `n` is left-heavy (`+2`), but its left child is right-heavy (`−1`)
![[file-Pasted image 20261003172344-20261003172344226.jpg]]
![[05-avl-double-LR.jpg]]
- Right Left: `n` is right-heavy (`−2`), but its right child is left-heavy (`+1`)
![[file-Pasted image 20261003172353-20261003172353576.jpg]]
![[05-avl-double-RL.jpg]]

![[file-Pasted image 20261003173618-20261003173618861.jpg]]

### Insertion 

Method
- Insert like a normal BST (new node becomes leaf)
- Walk back up towards the root updating height and bf
- At first node with bf +-2, apply the above matching case and stop

- Insertion needs at most one fix, as rotation after insert brings subtree back to same height before insert

Worked Examples:

Insert 10, then 20, then 30
![[05-avl-insert-step1-30.jpg|475]]
- Node 10 becomes -2, right child is -1, therefore RR (rotateleft on 10)

Insert 40 and then 50:
![[05-avl-insert-step2-50.jpg]]
- Goes right of 30, 20 has bf of -1, 30 as bf of -1, so tree is still balanced
- Inserting 50, 40 is still fine at -1, but 30 becomes -2.
	- Lowest unbalanced node, so we fix it there instead of the root
	- Right child 40 is -1, therefore RR -> rotateLeft(30). 

Insert 25:
![[05-avl-insert-step3-25.jpg]]
- Goes left of 30. 
- Walk up, 30 bf is +1, 40 is +1, 20 is -2
- We work on 20, whose right child 40 is +1, therefore RL -> rotateRight(40) and then rotateLeft(20)

### Deletion 

Method:
- Delete like normal BST
	- leaf → remove it
	- one child → replace it with that child
	- two children → copy in the successor's key, then delete the successor
- Walk back up toward the root, updating heights / balance factors.
- Fix EVERY node with bf = ±2 on the way up. Don't stop after the first.
- Deletion can cascade rotations as it makes subtree one level shorter causing a ripple effect. Leads to O(log n) rotations

Worked examples: 

Deletion where child has bf of 0: 
![[05-avl-delete-bf0.jpg|650]]
- 20 is -1, right child is 0. Therefore RR -> rotateLeft(20)

Deletion that cascades:
![[05-avl-delete-cascade-1.jpg]]
- Delete 1 -> 2 looses left child, becomes -2, child 3 is -1 -> RR -> rotateLeft(2)
![[05-avl-delete-cascade-2.jpg]]
- Now 5 has a bf -2,  right child 8 has bf of -1 -> RR again -> rotateLeft(5)
- Two levels of rotation from a single delete, but every node is now balanced

### AVL Height
![[file-Pasted image 20261003183622-20261003183622949.jpg|525]]