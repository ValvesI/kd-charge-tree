# DCT/KDCT: Dimensional Charge Tree

## 1. Scope and terminology

A **Dimensional Charge Tree (DCT)** is a spatial instantiation of the Charge Tree idea for elements represented as vectors in \(\mathbb{R}^d\). The name **KDCT**, or *k-dimensional Charge Tree*, emphasizes its relationship with \(k\)-dimensional spaces and the axis-based partitioning inherited from KD-trees.

In this document, **DCT** and **KDCT** refer to the same central idea. DCT is used as the shorter name.

This document describes:

- the basic DCT;
- charge-based maintenance;
- leaf Internal Structures;
- boxes used for queries;
- the **DCT with Box Adjust (DCTBA)** extension.

The general Charge Tree definition is available in [charge-tree.en.md](charge-tree.en.md).

## 2. Central idea

A DCT combines:

1. **axis-based spatial partitioning**, inspired by KD-trees;
2. **local charge-based maintenance**, inherited from Charge Trees.

Rather than building a complete tree for a static dataset, a DCT begins as one leaf. Instances are inserted into its **Internal Structure (EI)** and increase its charge. The acronym **EI** is retained from the Portuguese term *Estrutura Interna*. When capacity is reached, only that leaf is split. If removals sufficiently reduce the charge of sibling regions, they may be merged again.

~~~text
insertion
    -> charge increases
    -> local split
    -> new regions

removal
    -> charge decreases
    -> local merge
    -> recomposed region
~~~

Geometry determines **how** to divide and route, while charge determines **when** the structure should change.

## 3. Elements and feature space

Each stored element has a vector used by the tree:

\[
x=(x_0,x_1,\ldots,x_{d-1})\in\mathbb{R}^d.
\]

The element may also carry information outside the indexed space:

- an identifier;
- arrival time;
- class label;
- weight;
- other metadata.

Such data may be used by the application without participating in routing, boxes, or distance. In kNN classification, for example, the class label accompanies the instance but is not a search dimension.

## 4. Node structure

### 4.1 Leaf

A leaf \(v\) represents a region of the space and contains:

- an Internal Structure \(EI_v\);
- its charge \(\chi_v\);
- its routing region;
- optionally, a box adjusted to live elements;
- a parent reference and other maintenance metadata.

Elements remain in leaves. The concrete EI type is configurable and does not alter the structural definition of the DCT.

### 4.2 Internal node

An internal node records a spatial split. In the binary form, it maintains at least:

- the split axis \(a_v\);
- the split value \(t_v\);
- the child for lower values;
- the child for higher values;
- the regions or boxes associated with its children.

One possible routing convention is:

\[
\rho_v(x)=
\begin{cases}
v_{\mathrm{left}}, & x_{a_v}<t_v,\\
v_{\mathrm{right}}, & x_{a_v}\ge t_v.
\end{cases}
\]

The choice between strict and inclusive comparison is not essential, but the same convention must be used during split redistribution and future insertions.

## 5. DCT charge

Let \(S_v\) be the set or multiset of instances stored in leaf \(v\). As in any Charge Tree, DCT charge is defined by a configurable function:

\[
\chi_v=\phi(S_v).
\]

The most direct option counts all instances:

\[
\phi(S_v)=|S_v|.
\]

For data containing many repetitions, another considerable option is to count different feature vectors:

\[
\phi_{\mathrm{distinct}}(S_v)
=\left|\operatorname{distinct}(S_v)\right|.
\]

Under this configuration:

- duplicate instances remain stored;
- every occurrence remains available for querying and removal;
- inserting another occurrence of an existing vector does not increase structural charge;
- removing one occurrence decreases charge only when it was the last occurrence of that vector.

This option may prevent repeated copies of the same point from causing splits that cannot produce useful spatial partitions. It is not required for a structure to be a DCT.

## 6. Structural parameters

The main parameters are:

- \(d\): number of indexed dimensions;
- \(C\): capacity or charge that triggers a split;
- \(SN\): number of subregions produced by a split;
- \(Y\): optional merge threshold, with \(Y<C\).

In the binary form:

\[
SN=2.
\]

A leaf becomes eligible for splitting when:

\[
\chi_v\ge C.
\]

A split should occur only when a valid, nonempty partition exists. A leaf may reach capacity without being separable on the current axis. The implementation must then try another axis, choose another valid policy, or temporarily keep the leaf above capacity.

## 7. Axis selection

The policy most directly related to a KD-tree cycles through the axes:

\[
a_v=\operatorname{depth}(v)\bmod d.
\]

~~~text
depth 0   -> axis 0
depth 1   -> axis 1
...
depth d-1 -> axis d-1
depth d   -> axis 0 again
~~~

This is a simple deterministic policy. A DCT may also select a separable axis or an axis with greater dispersion, provided that the resulting decision defines consistent regions for future routing.

## 8. Split

### 8.1 Binary form

For a leaf \(v\) and an axis \(a_v\):

1. obtain the distinct values on that axis;
2. select a cut, normally near the median;
3. create two leaves;
4. redistribute every instance according to the cut;
5. replace the original leaf with an internal node;
6. compute the charge and boxes of the children.

\[
S_{\mathrm{left}}=\{x\in S_v:x_{a_v}<t_v\},
\]

\[
S_{\mathrm{right}}=\{x\in S_v:x_{a_v}\ge t_v\}.
\]

The cut must produce two nonempty sets. Duplicate values require a stable convention so that the rule installed in the node reproduces the same distribution for future insertions.

### 8.2 Form with \(SN\) children

A more general form orders the values on the selected axis, determines \(SN-1\) boundaries, and creates up to \(SN\) contiguous intervals.

A multiway split follows the same requirements:

- no created child is empty;
- each instance belongs to exactly one child;
- the boundaries correctly route new instances;
- the operation remains local to the split leaf.

### 8.3 Split cost

With an unordered EI, selecting a partition through sorting typically costs:

\[
O(C\log C).
\]

Distribution among children then costs \(O(C)\). An implementation may instead use linear selection, partial sorting, or another EI-specific strategy.

## 9. Insertion

~~~text
insert(x):
    v <- root
    while v is internal:
        v <- child selected by the axis and cut of v

    EI_v.insert(x)
    update the state needed to compute the charge of v
    update live boxes, if maintained by the variant

    if chi_v >= C and a valid partition exists:
        split(v)
~~~

Traversal uses only the stable cuts installed in internal nodes. Inserting new values does not automatically move old cuts.

## 10. Removal

Removal must identify either a specific occurrence or an equivalent element:

~~~text
remove(id or x):
    locate the leaf containing the element
    remove one occurrence from the EI
    update charge state
    adjust live boxes, if present
    evaluate a local merge
~~~

Location may repeat spatial routing. A FIFO window may instead maintain an auxiliary mapping:

~~~text
instance identifier -> current leaf
~~~

The map must be updated whenever a split or merge moves elements.

Removing one duplicate must not remove all other occurrences. Its effect on charge depends on \(\phi\). Under instance count, every removal decreases charge; under distinct-vector count, charge decreases only when the final occurrence of that vector leaves the leaf.

## 11. Merge

Merge is the inverse maintenance operation. In the binary form, two sibling leaves may replace their parent when their combined charge is below \(Y\):

\[
\phi\!\left(S_{\mathrm{left}}\uplus S_{\mathrm{right}}\right)
\le Y.
\]

A merge:

1. combines every occurrence from the leaves;
2. builds the EI of the new leaf;
3. recomputes charge through \(\phi\);
4. removes the parent split decision;
5. recomputes the live box when applicable;
6. updates external references to the new leaf.

Using a separate threshold \(Y<C\) reduces oscillation between split and merge.

## 12. Routing regions and live boxes

Two kinds of spatial bounds must be distinguished.

### 12.1 Stable routing region

The routing region is defined by ancestor cuts. It answers:

> Which path should a new instance follow?

These boundaries should remain stable while the node exists. Shrinking them solely from current elements could make a future instance inconsistent with the routing that created the subtree.

### 12.2 Live-element box

A live box summarizes the vectors currently stored below a node:

\[
B_v=\prod_{i=0}^{d-1}[\ell_{v,i},u_{v,i}],
\]

where:

\[
\ell_{v,i}=\min_{x\in S_v}x_i,
\qquad
u_{v,i}=\max_{x\in S_v}x_i.
\]

It answers a different question:

> What is the minimum possible distance between the query and any instance in this subtree?

A live box may grow after insertions and shrink after removals. It supports querying and pruning; it does not redefine insertion routing.

## 13. Base DCT and DCT with Box Adjust

### 13.1 Base DCT

A base DCT may use the regions induced by its cuts to guide search. These regions are correct but may contain large empty portions of the space.

Because their boundaries remain tied to split history, removing elements does not necessarily tighten the query box.

### 13.2 DCTBA

The **DCT with Box Adjust (DCTBA)** maintains boxes fitted to the live elements of each subtree in addition to stable routing regions.

During insertion, bounds may be expanded incrementally:

\[
\ell_{v,i}\leftarrow\min(\ell_{v,i},x_i),
\qquad
u_{v,i}\leftarrow\max(u_{v,i},x_i).
\]

During removal, if the removed element defined a minimum or maximum, the affected bound must be recovered from live elements or from an auxiliary summary.

The DCTBA principle is:

~~~text
stable cuts -> correct routing
live boxes  -> tighter pruning
~~~

Box Adjust does not change which elements are neighbors and does not make the query approximate. It only improves the bound used to decide which regions may still contain results.

## 14. Lower bound of a box

For a query \(q\) and a box \(B\), define:

\[
LB(q,B)=\min_{x\in B}d(q,x).
\]

For coordinate-separable metrics, construct the point in the box most favorable to the query:

\[
p_i=
\begin{cases}
\ell_i, & q_i<\ell_i,\\
q_i, & \ell_i\le q_i\le u_i,\\
u_i, & q_i>u_i.
\end{cases}
\]

Then:

\[
LB(q,B)=d(q,p).
\]

For Euclidean distance:

\[
LB_2(q,B)=\sqrt{\sum_i(q_i-p_i)^2}.
\]

For Canberra distance:

\[
LB_C(q,B)=
\sum_i\frac{|q_i-p_i|}{|q_i|+|p_i|},
\]

with a zero contribution when numerator and denominator are both zero.

If \(q_i\) lies inside the box interval, that dimension contributes zero to the lower bound. Wide or highly overlapping boxes therefore produce weak bounds even though the exact query remains correct.

## 15. Exact kNN search

An exact kNN search may use best-first branch-and-bound with two separate priority structures:

1. a **region queue**, implemented as a min-heap ordered by \(LB(q,B_v)\);
2. a **neighbor heap**, implemented as a max-heap limited to the current \(k\) best elements.

The worst element in the neighbor heap defines the current radius \(\tau\). Until \(k\) candidates exist, \(\tau=+\infty\).

~~~text
knn(q, k):
    insert the root into the region queue
    neighbors <- empty
    tau <- +infinity

    while the region queue is not empty:
        v <- remove the region with the smallest LB

        if k neighbors exist and LB(q, B_v) > tau:
            discard v
            continue

        if v is internal:
            for each child u of v:
                compute LB(q, B_u)
                if u may still improve the answer:
                    insert u into the region queue
        else:
            for each element x in EI_v:
                compute d(q, x)
                update the heap of k best elements
                update tau

    return the k best elements
~~~

When the smallest remaining lower bound is greater than \(\tau\), no unexplored region can contain a strictly better neighbor.

Ties at distance \(\tau\) require a convention. If an application must preserve every tied item or a particular deterministic order, regions with \(LB=\tau\) must not be pruned prematurely.

## 16. Radius query

The same lower bound supports radius queries:

\[
LB(q,B_v)>r
\quad\Longrightarrow\quad
\text{the subtree may be pruned}.
\]

A safe upper bound may prove that an entire region lies inside the radius, but it is not required for basic kNN pruning. The lower bound is the essential information for exact kNN.

## 17. Invariants

A DCT should preserve:

1. **Routing:** every stored element satisfies all cuts on the path to its leaf.
2. **Membership:** each active occurrence belongs to one leaf.
3. **Multiplicity:** duplicates are preserved regardless of how they contribute to charge.
4. **Charge:** \(\chi_v\) matches the current value of \(\phi(S_v)\).
5. **Split:** splitting preserves every occurrence and creates no empty children.
6. **Merge:** merging preserves every occurrence from the replaced descendants.
7. **Stable region:** removals do not retroactively change routing decisions.
8. **Live box:** when present, \(B_v\) contains every live element below \(v\).
9. **Safe pruning:** a region is removed only when its lower bound proves that it cannot improve the current result.

## 18. Basic costs

Let:

- \(h\): tree height;
- \(d\): dimensionality;
- \(C\): leaf capacity;
- \(V\): number of nodes visited by a query;
- \(L\): number of visited leaves;
- \(n_j\): number of occurrences in visited leaf \(j\).

### 18.1 Insertion

Routing costs \(O(h)\) when each node performs one axis comparison. Inserting into a vector EI may cost \(O(1)\) amortized. Updating a live box along the path costs up to \(O(hd)\).

Without a split:

\[
O(h+I_{EI}),
\]

or \(O(hd+I_{EI})\) when boxes of all ancestors are explicitly updated.

### 18.2 Split

With full EI sorting:

\[
O(C\log C).
\]

### 18.3 Removal

With a direct leaf map, location may be expected \(O(1)\). The remaining cost depends on the EI and box maintenance. If a removed extreme must be rediscovered by scanning a leaf, local adjustment may cost \(O(Cd)\).

### 18.4 Merge

Combining two leaves whose total charge is bounded by \(Y<C\) typically costs \(O(C)\), in addition to rebuilding the EI and box.

### 18.5 kNN

The cost depends on how many boxes and instances cannot be pruned:

\[
O\!\left(
V(d+\log V)
+\sum_{j=1}^{L}n_jd
+K\log k
\right),
\]

where \(K\) is the number of candidates that cause neighbor-heap operations.

In the worst case, the boxes permit no pruning and every instance is examined:

\[
O(Nd).
\]

A DCT should not be described as guaranteeing \(O(\log N)\) search. Branch-and-bound effectiveness depends on data distribution, dimensionality, metric, capacity, and representation.

## 19. Relationship with KD-trees

Both structures use:

- vectors in a dimensional space;
- axis-based cuts;
- coordinate comparisons for routing;
- spatial pruning.

Their central conceptual difference is the unit of adaptation.

A traditional KD-tree commonly associates an instance with each node and may rely on balanced construction or rebuilding to recover quality after many updates.

In a DCT:

- elements remain in leaves;
- each leaf has an EI;
- leaf charge controls granularity;
- a split replaces only the overloaded leaf;
- a merge locally recomposes regions;
- the structure follows the stream without requiring global rebuilding as its normal operation.

A DCT is therefore not merely a KD-tree with larger leaves. Its defining characteristic is the combination of dimensional partitioning with a local cycle of charge, split, and merge.

## 20. Core and optional mechanisms

The basic DCT includes:

- vectors in \(\mathbb{R}^d\);
- a charge function \(\phi(S_v)\);
- capacity \(C\);
- Internal Structures in leaves;
- local axis-based splits;
- stable routing regions;
- storage of instances in leaves.

Optional mechanisms and variants include:

- merge and threshold \(Y\);
- charge based on all instances or only different vectors;
- live boxes and Box Adjust;
- kNN search and a concrete metric;
- a reference map for removal;
- adaptive axis selection;
- splits with \(SN>2\);
- specialized EIs;
- feature representations or conversions.

## 21. Short definition

> A DCT or KDCT is a Charge Tree for multidimensional vectors. Each leaf stores its instances in an Internal Structure and measures its charge through a function \(\phi(S_v)\). Counting only different vectors is a considerable configuration when many duplicates exist, but it is not required by the structure. Upon reaching capacity, a leaf is split locally through axis-aligned cuts, producing stable regions for future routing. Removals may trigger merges, while boxes associated with regions support exact spatial queries through branch-and-bound. In a DCT with Box Adjust, routing remains stable while query boxes follow only live elements, providing tighter pruning bounds.
