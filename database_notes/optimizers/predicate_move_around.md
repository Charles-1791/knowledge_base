# Predicate Move around
## TL;DR
predicate move around = predicate pull up + predicate transition + predicate push down
## Introduction
### Classic Predicate Transition
Based on given predicates, the database can deduce additional ones to optimize query execution. For example, consider the query:
```SQL
SELECT *
FROM foo f JOIN bar b ON f.id = b.id AND f.id = 100
```
which has the following LogicalPlan:

<img src="../images/pm_introduction1.jpg" alt="Plan Tree" width="30%"/>

Thanks to the transitivity of equality, we can infer that *b.id = 100* must also hold for any tuple to appear in the output. Thus, the above SQL is equivalent to:
```SQL
SELECT *
FROM foo f JOIN bar b on f.id = b.id AND f.id = 100 AND b.id = 100
```
The both the predicate *f.id = 100* and the generated one can be push down to nodes below the inner join, resulting in a Plan like this:

<img src="../images/pm_introduction2.jpg" alt="Plan Tree" width="30%"/>

Since the filter *id = 100* is applied before, the amount of data flowing into join is reduced. Less data means less memory expense and faster join speed.

### Predicate Move around
Traditional predicate transition typically derives new predicates only at join nodes, using conditions from the JOIN ON clause. In contrast, our algorithm infers predicates from all explicitly and implicitly defined conditions throughout the entire plan tree. Here is an example:

```SQL
SELECT * 
FROM (
        SELECT * 
        FROM t1 JOIN t2 ON id1 = id2
    ) temp1 
    JOIN 
    (
        SELECT * 
        FROM t3 JOIN t4 ON id3 = id4
    ) temp2 
    ON temp1.id1 = temp2.id3 
WHERE id4 > 2020;
```
<img src="../images/pm_introduction_complex.jpg" alt="Plan Tree" width="65%"/>

(For clarity, I labeled each node.)

If we only iterate from top to bottom, at node 3, we only have `id4 > 200` and `id1 == id3`, which alone don’t allow us to infer much. By comparison, our algorithm performs a pull-up before push-down, effectively "gathering information" from all nodes. 

For instance, at node 4, we learn `id1 = id2`, and at node 5, we have `id3 = id4`. When we reach node 3, which contains `id1 = id3`, we can infer that `id1 = id2 = id3 = id4`. Upon arriving at node 2, we can deduce that `id1 = id2 = id3 = id4 > 2020`.

With this knowledge, during push-down phase, we generate predicates `id1 > 2020`, `id2 > 2020`, `id3 > 2020`, and `id4 > 2020` and push them just above the table scan node.

An optimized Plan tree looks like:

<img src="../images/pm_introduction_complex_after.jpg" alt="Plan Tree" width="65%"/>

### Recognizing Redundant Predicates
Deriving as many predicate as possible is not always beneficial. Considering the following query:

```SQL
SELECT *
FROM (
    SELECT id1, id2 
    FROM t1, t2
    WHERE id1 = id2
    ) temp1 JOIN t3 ON id1 = id3
```

It is easy to infer that `id1 = id2 = id3`, but adding an extra  predicate like `id2 = id3` does not improve performance - in fact, it can slow down execution due to increased computational overhead or redundant checks.

We consider a newly derived predicate P to be `redundant` if it can be inferred from the set of all predicates already present in the subtree rooted at the lowest plan node N to which P can be pushed.

## An Overview of Logical Plan Nodes
[Logical Plan Node](../architecture/logical_plan_node.md)

## Algorithm Implementation
Different from the regular practice in papers, where a closure of predicates is generated, our approach is to 'infer only when necessary' - we do not generate predicates as much as possible(predicate closure) and eliminate redundancy afterwards. Instead, only meaningful and "pushable" predicates are deduced.

### Data Structure
``` c++
struct PredicateSummary {
    // a disjoint set
    equalSets set<set<uid>>
    // predicates
    conditions expression[]
} 
```
The struct PredicateSummary is, as its name suggests, a summary of the predicates a node implies. It has two members - `equalSets` and `conditions`.

#### EqualSets

`equalSets` is a collection of sets of `UIDs`; `UIDs` in the same set are considered equivalent, as they have the same type and value. For instance, given these predicates: `#1 = #2`, `#1 = #3`, `#4 = #5`, the resulting equalSets would be: `{{#1, #2, #3}, {#4, #5}}`.

#### Conditions

`conditions` is a list of `expressions`. An `expression` is a predicate that *DOES NOT* having the form `col1 = col2`; otherwise it should be recorded in `equalSets` instead. A typical `conditions` looks like 

`[#1 + #2 = #3, #4 < 100, concat(#5, #6) is like '%123%']`

A predicate in `conditions` should be interpreted as "a relation among equivalent sets", rather than "a relation among columns". Considering the following equal sets:

```
EqualSets: {#1, #2, #3}, {#4, #7}, {#5, #6}, {#8}, {#9}
```

The predicates:
```
#1 + #2 - f(#4) * g(#5) = #8 + #9
```
should be construed as a 'template'
```
$0 + $0 - f($1) * g($2) = $3 + $4
```
where the $ are placeholders -- `$0` stands for the first equivalent set, `$1` stands for the 2nd equivalent set, etc... In this way, a predicate reveals the arithmetic relation among equivalent sets. By replacing every placeholders with a column from its corresponding set, a valid predicate is generated. In the example given above, in total `3 * 3 * 2 * 2 * 1 * 1 = 36` different predicates can be generated. But again, we only generate new predicates when needed.


### Two Phases
Predicate Move Around is implemented in two steps -- predicate pull up and push down.

<img src="../images/pm_tp_1.jpg" alt="Plan Tree" width="65%"/>

In the pull up phase, every node(except the table scan) receives from its child a `PredicateSummary`, which is processed differently according to the nature of the node to generate a new `PredicateSummary`. The newly generated summary is then returned to its father node, where further modifications are made. Similar to a bubble rising to the water surface, one by one, the `PredicateSummary` ascends from the leaves to the root.

The push down phase begins upon the finish of the pull up phase. The summary returned by root now traverses down from root to leaves. While permeating through a node, the summary changes accordingly. Meanwhile, some new predicates may be generated and are attached to the node. When a summary reaches the leaf nodes, which are usually Table Scans, it 'flattens' itself into a group of predicates and becomes a `Selection` Node, hanging just above the leaves.

For a certain plan node, if we name as `summary_up` the predicate summary returned by it during pull up phase, and `summary_down` the summary it receives from its father during push down phase, it's always true that `summary_up <= summary_down`. In other words, the `summary_down` always contains more "knowledge" or "information" than `summary_up`. 

Another thing is certain: **a summary received by a node must include and only include columns returned by its children**.

### Pull Up and Push Down Logic
Since the pull-up and push-down logic are closely related, the algorithm is explained node by node, with each node followed by its corresponding pull-up and push-down details.

#### TableScan
##### Pull up
A table scan does not have any predicates, so the pull up process is quite simple: create an empty summary and add all output uids into summary.equalSets, each of them forms a separate equal set.
##### Push down
As table scan has no child, summary must be converted back into a group of predicates, which are then carried by a `Selection` node. 

Similar to stringing some pearls into a necklace, the process of flattening `equalSets` is using equation marks to connect each column in the same set. 

```c++
flatten([ {#1}, {#2, #3, #4}, {#5, #6} ]) = [#2 = #3, #3 = #4, #5 = #6]
```

Since `conditions` contains predicates that can be used directly, we can add them to the `Selection` Node.

<img src="../images/pm_pupdl_ts_1.jpg" alt="Plan Tree" width="80%"/>

PS: A better practice would be converting predicates into new conditions more 'favored' by database through substituting columns. After replacing column with an indexed(and equivalent) one, a new predicate usually outperforms the original one thanks to the more efficient index scan. 

#### Selection



