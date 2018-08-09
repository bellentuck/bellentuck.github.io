This brief builds toward a strategy for testing the validity of a linked tree; recursive and iterative implementations are provided in Javascript and Python. A linked tree refers to a binary search tree ("BST") implemented via a linked list. The premise entails some preliminary question-posing. Before we can ask what it means for a linked tree to be valid, we ought to untangle the distinctions between linked tree, linked list, and BST. What even *is* a tree?

## What even *is* a tree?
A BST is, among other things, a tree. And a tree is an abstract data type ("ADT"). An ADT, in turn, is a behavioral specification. Think of a specification as a blueprint, a subjunctive term referring to something desired. An ADT can be concieved of as a blueprint for behavior. In computer science, a behavioral specification can also be referred to as an *interface*. An ADT is an interface for data.

### What behavior does the tree ADT specify?
What's a tree to do? Being an interface for data, the tree ADT specifies how to relate or manage data in "tree-like" fashion. What conceptual baggage does managing data in a tree-like way require?
1. A tree maintains a hierarchy of entities conventionally referred to as *nodes*.
2. Any node can have other nodes to which it relates in a hierarchical fashion; these are its *children*. **All nodes are trees**; children are subtrees. There are to be no duplicate children.
3. Trees can branch but not converge; the "final"-i.e.-childless nodes of a branch are called *leaves*.
4. Not only do trees have nodes; every tree contains a single, special *root* node that acts as the entry point into the tree.
5. The *depth* of a node relative to a tree containing it is equal to the number of edges from node to root of tree. Because trees can branch but not converge, this number is constant. Any root node's depth as such is `0`.
6. The *height* of a node relative to a tree containing it is equal to the largest number of edges that make up a single path from that node to any leaf. Any root node's height as such is `n`, the number of "levels" of children in the tree.

### What behavior does the BST ADT specify?
What is a *binary search* tree to do? In addition to the above properties, a BST requires the following three.
1. A given node can have <=2 children (either 0, 1, or 2 children).
2. A BST is "ordered" such that a given node's "left" child's value is less than that of the given node, and the given node's "right" child's value is greater than or equal to that of the given node.
3. A BST supports "CRUD" (Create, Read, Update, Delete) operations. I.e., it should be possible to create a BST, find a value in a BST, insert a value into a BST, and delete a value from a BST.

## What's the difference between a linked tree, a linked list, and a BST?
A BST can be implemented in several ways. In other words, we have multiple options in terms of the set of data structure(s) we use to achieve the specified criteria for a BST. The two most common primary data structures for implementing a BST are an array and a linked list. A BST implemented via a linked list is known as a "linked tree."

Implementing a BST using a linked list is semantically straightforward in that access to children is a matter of calling `BST.leftChild` or `BST.rightChild`. A child's accessing its parent could be a matter of calling `BST.parent`; note, however, that BST children are at least conventionally not able to access their parents.

## What does it mean for a BST to be valid?
With some idea of the criteria making up the BST spec, what can we say about what it means for a BST to be "valid"? A BST is valid if
1. `BST.leftChild.value < BST.value`, and
2. `BST.rightChild.value >= BST.value`.
Crucially, this implies
3. `BST.value < BST.parent.value` if the BST is itself a left child, and
4. `BST.value >= BST.parent.value` if the BST is itself a right child.

## How can BST validity be assessed?
To determine the validity of a BST, we need to visit every node in the tree and determine whether *it* fulfills the two above criteria for a valid BST. Again, let's assume we do not have access to a "BST.parent" method.

### Brute forcing it: quadratic time
Carrying BST validation out in a "brute force" manner would entail checking each and every tree-within-umbrella-BST, all the way down the umbrella tree, verifying that each subtree is or is note *discretely* a valid BST.

The "discrete" part of the brute force approach is significant. We're calculating whether subtree A is valid independently of whether subtree B is valid. This can entail a *lot* of repetitive operations: consider the degenerate linked tree that is itself a singly-linked list, where each element points to a child uniquely its own. We would be checking up to all the nodes to validate the first subtree, up to all but one of the nodes to validate the second subtree, etc., leading to an O(n^2) runtime where "n" is the number of nodes in the tree. We can do better.

### Doing better: linear time
Reasoning about "why" a linear-time approach would work is straightforward enough. We do need to check every node to ensure a BST is valid. But there's nothing stopping us from *only* checking each node once.

Reasoning about *how* a linear-time approach would work is less straightforward. Essentially, we want to be able to partially check multiple subtrees at each given node.

#### BST validation base case
Consider the vacuous case. The vacuous or "base" case in validating a BST is *not* a BST with no children, because of reasons 3 and 4 from the "What does it mean for a BST to be valid?" section. For a given BST, we need to ensure it is a valid *child*. And this insight leads us to our formula and base case both.

First, the latter: our base case is a `null` BST, no BST at all. Trivially (but crucially!), the complete lack of a BST itself denotes a valid BST. (Consider the validity test to be true by default. We could amend our approach to account for the edge case of a "null" umbrella BST if needed by checking at the very start of our `validateBST` umbrella procedure.)

#### BST validation formula
To our formula. If a BST is not itself null, it has a value. Rather than consider this value to be "greater than the left child's" or "less than the parent's if the BST's a left child," let's abstract our limits.

For a given BST, let there exist a "minimum" and a "maximum"; and let the BST be at least partially valid (i.e., not necessarily invalid) if the value at this node is greater than or equal to the minimum, and less than the maximum, like so: **minimum <= node.value < maximum**.

#### BST validation approach
The resulting approach runs as follows: until we hit a null BST, let's test that the BST we *have* hit is "not necessarily invalid" vis-a-vis min and max values, as established in the previous paragraph. To do so, let's initialize some "maximum" and "minimum" values as Infinity and -Infinity, respectively. So far, so tautologically good: the first tree we hit will be "not necessarily invalid."

We still need to check every node, however, to discern whether the BST is necessarily valid. So, we'll move down left and right branches. The catch is that now, rather than passing Infinity and -Infinity as "max" and "min" into our comparison procedure, we'll pass in `BST.value` (!).

Recall criteria 1 and 2 from "What does it mean for a BST to be valid?" Given a parent node, the left child node should be smaller than the parent node's value; the right child node should be greater than or equal to the parent node's value. Therefore:
1. For a `BST.leftChild`, the comparison should be `minimum <= BST.leftChild.value < BST.value`.
2. For a `BST.rightChild`, the comparison should be `BST.value <= BST.rightChild.value < maximum`.

We will continue following a given branch--call it "branch X"--until we've reached our vacuous case. Once we've hit a null tree (trivially a valid BST), we can bubble up this result appropriately: "branch X" is good! Once all branches are necessarily valid BSTs themselves we can conclude that the overall "umbrella" BST is valid, too.

### Linear time implementations of BST validation
Procedures entailing tree traversal can be implemented via either recursion or iteration. We'll go over both implementations in validating a BST.

#### Recursive implementation
While recursion can appear less self-documenting than iteration in the abstract, I find a recursive approach more in keeping with the mental model of a tree, as a tree of more than one element is necessarily made of other trees.

If we cared about preserving the arity of our `validateBst` method (if we wanted it to only take one argument, the tree to test) we could factor out the validation procedure into a `validateBstHelper` method. Or, as below, we could just set default params for `min` and `max`.

Javascript implementation:
```js
function validateBst(tree, min = -Infinity, max = Infinity) {
  // 3 conditions to check:

  // (1) Base case: no tree--so, vacuously, a valid BST.
  if (tree === null) return true;

  // (2) Bad tree!
  if (tree.value < min || tree.value >= max) return false;

  // (3) Potentially good tree--still have to recurse until reach all vacuous cases.
  return validateBst(tree.left, min, tree.value) && validateBst(tree.right, tree.value, max);
}
```
Python implementation:
```python
def validateBst(tree, min = float('-inf'), max = float('inf')):
    # 3 checks:

    # (1) vacuously valid BST
    if tree == None:
        return True

    # (2) non-valid BST
    if tree.value < min or tree.value >= max:
        return False

    # (3) not necessarily invalid BST
    return validateBst(tree.left, min, tree.value) and validateBst(tree.right, tree.value, max)
```


#### Iterative implementation

Recall how in the brute force approach, we iterate through every node *and separately* check that every subtree touching that node is valid. Yet this procedure conflates subtree with node. Just as one can't iterate through subtrees as such, one can't verify a BST based on a node. One needs to verify whether this node is itself a valid subtree, period.

We will thus keep track of `min` and `max` values relative to each subtree we test, pushing an object of this information onto a stack (we could equivalently implement this procedure using a queue). As in breadth-first search, once our supplementary data structure is empty we know we're out of options. In the case of testing the validity of a tree, being out of options means that the entire tree has been deemed valid: i.e., we haven't found any invalid subtrees. If we ever find one, we can short-circuit the procedure entirely.

JS:
```js
function validateBst(tree) {
  let min = -Infinity;
  let max = Infinity;
  let bstsToValidate = [{ tree, min, max }];

  while (bstsToValidate.length) {
    const { tree, min, max } = bstsToValidate.pop();

    if (tree === null) continue;
    if (tree.value < min || tree.value >= max) return false;

    bstsToValidate.push(
      { tree: tree.left, min, max: tree.value },
      { tree: tree.right, min: tree.value, max }
    );

  }
  return true;
}
```
Python:
```python
def validateBst(tree):
    initMin = float('-inf')
    initMax = float('inf')
    bstsToValidate = [(tree, initMin, initMax)]

    while bstsToValidate:
        tree, minimum, maximum = bstsToValidate.pop()

        if tree == None:
            continue
        if tree.value < minimum or tree.value >= maximum:
            return False

        bstsToValidate.append((tree.left, minimum, tree.value))
        bstsToValidate.append((tree.right, tree.value, maximum))

    return True
```

## Concluding Complexity Analysis
Hopefully by now you've gained some intuition about how to test the validity of a linked tree! As shown above, we can do so in O(n) time, where "n" equals the number of subtrees to test (i.e., the number of nodes in the overall umbrella tree); and in O(d) space, where "d" equals the depth of the tree (we will never keep more than "d" items on our call stack in the recursive solution, or in our manually-created stack in the iterative solution).
