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

Our algorithm

## An Overview of Logical Plan Nodes
[Logical Plan Node](../architecture/logical_plan_node.md)

## Algorithm Implementation
Different from the regular practice in papers, where a closure of predicates is generated, my approach is to 'deduce only when you have to' - we do not generate predicates as much as possible(predicate closure) and eliminate redundancy afterwards. Instead, only meaningful and pushable predicates are deduced.

### Data Structure



