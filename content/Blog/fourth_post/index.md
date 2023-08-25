---
title: "How Can Linear Algebra Simplify and Optimize Graph Algorithms? An Introduction to GraphBLAS "
date: 2023-08-13T14:00:31-04:00
draft: false
tags: ["Technology", "Algorithms", "Graphs", "Linear Algebra", "Programming"]
summary: "GraphBLAS leverages sparse matrix computations in an extended algebra of semirings, providing a scalable and efficient framework for expressing graph algorithms through linear algebra."
---

What if I told you that the task of finding whether a path exists between two nodes in a graph could be elegantly distilled into just 6 lines of code thanks to the power of linear algebra?

This:

```python
#a standard implementation of BFS tailored to check for path existence
#between two nodes.
def bfs(G, start, target):    
    seen = set()
    seen.add(start)

    frontier = deque([start])
    
    while frontier:
        vertex = frontier.popleft()
        if vertex == target:
            return True
        
        neighbors = G[vertex]
        for neighbor in neighbors:
            if (neighbor not in seen):
                seen.add(neighbor)
                frontier.append(neighbor)
    return False
```

**is the exact same as this**:

```python
def bfs_graphBLAS(M, v, target):  #my implementation leveraging graphBLAS
    while not v[target]:
        w = v.dup()
        v << semiring.lor_land(v @ M)
        if v.isequal(w):
            return False
    return True
```

While the traditional BFS algorithm is straightforward to articulate, the complexity increases with more intricate graph algorithms. Writing these algorithms, especially in a parallel computing environment, becomes increasingly challenging when focusing on a node and edge-centric model.

What if we could leverage the established mathematics of linear algebra to approach graph algorithms in a new way? What if we could view graphs as vector spaces and use isomorphisms to translate graph problems into the language of linear algebra? This perspective opens up a whole new world of possibilities and leads us to a powerful tool: GraphBLAS

## What is GraphBLAS?

{{< lead >}}
*Enter GraphBLAS a specification that defines a core set of linear algebra building blocks optimized for graph algorithms.*
{{< /lead >}}

I first found out about GraphBLAS in [Hacker News](https://news.ycombinator.com/news), and thought it was pretty cool how a linear algebraic approach led to such flexibility when dealing with the implemenation of various graph algorithims. Now I'm well aware that the similarities between graphs and linear algebra is not something new (*cough, cough, adjancency matrixes*). But GraphBLAS takes it a step further. I'll be getting to that shortly, don't worry.

The fundamental action in graph algorithms involves traversing from a vertex to its neighboring vertices. Surprisingly, this action can be represented within the matrix representation of a graph, specifically as matrix multiplication. By repeatedly multiplying the same matrix, we can enable the traversal of the graph using a Breadth-First Search (BFS) method.

Don't believe me? Lets try an example. Say we have the following unweighted directed graph where we want to use BFS to find the path from node `A` to node `F`:

## What's a Simple Way We Can Implement BFS Using GraphBLAS?

{{< mermaid >}}
graph TD
    id1(A) --> id4(D)
    id1(A) --> id2(B)
    id2(B) --> id3(C)
    id2(B) --> id1(A)
    id2(B) --> id5(E)
    id3(C) --> id4(D)
    id3(C) --> id6(F)
    id5(E) --> id6(F)
    id6(F) --> id1(A)
    id4(D) --> id5(E)
    style id1 fill:#c026d3
    style id6 fill:#7e22ce
{{< /mermaid >}}

We can represent this graph as an adjacency matrix {{< katex >}} \\(M\\),

{{< katex >}}
$$ M =
\begin{bmatrix}
    .& 1 & . & 1 & . & . \\\
    1& . & 1 & . & 1 & .  \\\
    .& . & . & 1 & . & 1 \\\
    .& . & . & . & 1 & .  \\\
    .& . & . & . & . & 1 \\\
   1 & . & . & . & . & .
\end{bmatrix}
$$
{{< alert "circle-info" >}}
For clarity we use `.` to represent the abscence of an edge.
{{< /alert >}}

Where `1` at index {{< katex >}} \\((i,j)\\) indicates there is an edge going from node \\(i\\) to node \\(j\\). 

In BFS we start at a given node and iterate through our *frontier set*, or the nodes that our current node has an edge to. We can represent this "frontier set" as a vector \\(v\\) in which \\(v_k = 1\\) \\(\iff\\) node \\(k\\) is part of our frontier set. So at the start of our BFS algorithim our frontier set should look like:

{{< katex >}}
$$ v =
\begin{bmatrix}
1\\\
.\\\
.\\\
.\\\
.\\\
.\\\
.
\end{bmatrix}
$$

given BFS traversal will begin from node \\(A\\), meaning \\(A\\) is the only node in our frontier at the moment.

Now that we have our adjacency matrix \\(M\\) and frontier vector \\(v\\), does that mean we can go straight into some matrix-vector multiplication using \\(M × v\\) to propagate the frontier to the next level of vertices? Well, not exactly. Let's step back and interpret how we view our matrix.

### How Do Matrix Transposition and Vector Multiplication Shape Graph Traversal? 

Currently we interpret \\(M_{ij}\\) as the edge going from node \\(i\\) to node \\(j\\). This means that row \\(i\\) in \\(M\\) hold the nodes that are currently in the frontier of \\(i\\), showing where we can go next ("successors"). And column \\(j\\) in \\(M\\) hold the nodes that point to node \\(j\\), showing where we could have came from ("predecessors"). We interpret \\(v\\) as a column vector indicating the nodes that are currently in the frontier within an iteration of BFS.

However, if we want to find the nodes that are available in the subsequent frontier we do not do \\(M × v\\) but rather \\(M^T × v\\). But why?

\\(M^T\\) is \\(M\\) just with the edge directions reversed. This means that rows identify predecessor nodes and columns identify successor nodes. Mathematically, the \\(i\\)th element of the result is given by:
{{< katex >}}
$$
(M^T \times v_i )= \sum_j M^T_{ij} \cdot v_j
$$
In other words When you multiply \\(M^T\\) by the current frontier vector \\(v\\), you are effectively looking at the inbound edges to the current frontier nodes—this identifies the nodes that lead to the current frontier, effectively finding the next set of nodes to explore. This is known as the 'pull' approach.

GraphBLAS also supports \\(v \times M\\), where \\(v\\) is a row vector (vector matrix multiplication). This approach identifies the next set of nodes reachable from the current frontier, it's also known as the 'push' approach.

Both approaches give the same result, but I'll just choose to do \\(M^T × v\\) given that most people are familiar with matrix-vector multiplication.

{{< alert "circle-info" >}}
For those intrigued by the internals of the 'push/pull' approach in matrix multiplication, especially as it applies to sparse graphs, I highly recommend checking out pages 297-298 in the [GraphBLAS User Guide](https://github.com/DrTimothyAldenDavis/GraphBLAS/blob/stable/Doc/GraphBLAS_UserGuide.pdf).
{{< /alert >}}

So now, let us compute \\(M^T \times v\\):

{{< katex >}}
$$ M^T \times v =
\begin{bmatrix}
    .& 1 & . & . & . & 1 \\\
    1& . & . & . & . & .  \\\
    .& 1 & . & . & . & . \\\
    1& . & 1 & . & . & .  \\\
    .& 1 & . & 1 & . & . \\\
   .& . & 1 & . & 1 & .
\end{bmatrix}
\times
\begin{bmatrix}
1\\\
.\\\
.\\\
.\\\
.\\\
.
\end{bmatrix} = \begin{bmatrix}
.\\\
1\\\
.\\\
1\\\
.\\\
.
\end{bmatrix}
$$

{{< lead >}}
*Yes! We found the new frontier set! But you'll soon see one issue...*
{{< /lead >}}

In the result vector, only the 2nd and 4th components (referring to nodes \\(B\\) and \\(D)\\) are set to 1. This means after one iteration of BFS this is our new frontier set. This mirrors what we expect to happen when performing one iteration of BFS:
 {{< mermaid >}}
graph TD
    id1(A) --> id4(D)
    id1(A) --> id2(B)
    id2(B) --> id3(C)
    id2(B) --> id1(A)
    id2(B) --> id5(E)
    id3(C) --> id4(D)
    id3(C) --> id6(F)
    id5(E) --> id6(F)
    id6(F) --> id1(A)
    id4(D) --> id5(E)
    style id1 fill:#c026d3
    style id6 fill:#7e22ce
    style id4 fill:#c084fc
    style id2 fill:#c084fc
{{< /mermaid >}}

However a problem occurs when we try to calculate the next frontier-set using our current frontier-set. We expect that our next-frontier-set should have nodes \\(E\\) and \\(C\\) since they are the only nodes accessible from either \\(B\\) or \\(D\\). So in our matrix we want a 1 in the second and fifth index positions (zero-indexed) of our resulting vector to portray this.

However when performing \\((M^T) \times v \\) we get an unexpected answer:

{{< katex >}}
$$ M^T \times v =
\begin{bmatrix}
    .& 1 & . & . & . & 1 \\\
    1& . & . & . & . & .  \\\
    .& 1 & . & . & . & . \\\
    1& . & 1 & . & . & .  \\\
    .& 1 & . & 1 & . & . \\\
   .& . & 1 & . & 1 & .
\end{bmatrix}
\times
\begin{bmatrix}
.\\\
1\\\
.\\\
1\\\
.\\\
.
\end{bmatrix} = \begin{bmatrix}
1\\\
.\\\
1\\\
.\\\
2\\\
.
\end{bmatrix}
$$

{{< alert "xmark" >}}
Yikes, why is there a 2? Also do we want to consider A in our frontier set if we already visited it? How can we keep track of visited nodes?
{{< /alert >}}

Let's address the first problem. The standard matrix-vector multiplication in the context of graph traversal identifies the frontier set correctly. However, these conventional arithmetic operations may not always align with the desired behavior for combining and propagating values within the frontier set. This is where one of the core concepts of GraphBLAS comes in.

### What are Semirings?

Semirings provide a framework to define how values are combined and propagated within matrix multiplication. By allowing for specialized definitions of arithmetic operations, semirings enable a tailored representation of matrix multiplication that aligns with the specific logic and requirements of whatever algorithim you are implementing.

You're basically 'redefining' what it means to **add** and **multiply** values in the context of matrix multiplication.

More formally, a semiring is a set \\(S\\) equipped with two binary operations, often denoted as \\(⊕\\), the monoid, and \\(⊗\\), the multiplier, that satisfy the following properties:

{{< katex >}}
$$
\text{Closure: }(a ⊕ b \in S) \text{ and } (a ⊗ b \in S) \text{ for all } a, b \in S \\\
\text{Associativity: }a ⊕ (b ⊕ c) = (a ⊕ b) ⊕ c \text{ and } a ⊗ (b ⊗ c) = (a ⊗ b) ⊗ c \\\
\text{Distributivity: }a ⊗ (b ⊕ c) = (a ⊗ b ) ⊕ (a ⊗ c)\\\
\text{Identity elements: } \exists \text{ elements } 0 \text{ and } 1 \text{ s.t: } a ⊕ 0 = a \text{ and } a ⊗ 1 = 1 \text{ for all } a \in S\\\
\text{Annihilation: } a ⊗ 0 = 0 \text{ for all } a \in S
$$

A semiring is a mathematical structure that generalizes the notions of *addition* and *multiplication*. Whenever we are performing standard matrix multiplication we are concerned with these two operations only. Semirings give us increased flexibiltiy to redefine how matrix multiplication should be done.

Semirings are a huge topic of their own, and I'll probably write a blog about the intuition for using different semirings in the near future.

Coming back to our problem, we tried using the standard `PLUS_TIMES` semiring (which corresponds to conventional matrix multiplication) and found it unsuitable. Instead, let's try using the logical `OR` and logical `AND` semiring, where `OR` serves as the monoid (⊕) and `AND` as the multiplier (⊗). My choice of this semiring is due to three main reasons:

1. **Binary Representation**: In the context of path finding, we are often interested in the existence of a path rather than its weight or length. The logical OR and logical AND operations provide a binary representation that captures this existence.
2. **Propagation of Connectivity**: The logical `OR` operation, as the monoid, allows the propagation of connectivity information across the graph. If there is a path from one node to another, the OR operation ensures that this connection is carried forward in the computation.
3. **Intersection of Paths**: The logical `AND` operation, as the multiplier, ensures that only the intersections of paths are considered. This aligns with the step-by-step traversal of BFS, where we explore the nodes that are reachable from the current frontier.

Let's try the multiplication again using this semiring:
{{< katex >}}

Given a row \\(i\\) we can define our `OR_AND` semiring as such:

$$
f(i) = \bigvee_{j=1}^{6} (M_{ij}^T \land v_j)
$$

Note how we compute AND first and OR second..

$$ M^T \times v =
\begin{bmatrix}
    .& 1 & . & . & . & 1 \\\
    1& . & . & . & . & .  \\\
    .& 1 & . & . & . & . \\\
    1& . & 1 & . & . & .  \\\
    .& 1 & . & 1 & . & . \\\
   .& . & 1 & . & 1 & .
\end{bmatrix}
\times
\begin{bmatrix}
.\\\
1\\\
.\\\
1\\\
.\\\
.
\end{bmatrix} = \begin{bmatrix}
f(1)\\\
f(2)\\\
f(3)\\\
f(4)\\\
f(5)\\\
f(6)
\end{bmatrix} = \begin{bmatrix}
1\\\
.\\\
1\\\
.\\\
1\\\
.
\end{bmatrix}
$$

Now we just have 1's to dictate what is in our current frontier set. However, you can notice that we readded node A to our frontier set given \\(v_1 = 1\\). How do we prevent ourselves from visiting already visited nodes?

### How is Masking Leveraged in GraphBLAS?

While concern over visiting previously visited nodes does not impact the correctness of using BFS to find if a path exists between two nodes, there can be use cases in which we may not want to revisit nodes in our "visited set". Luckily we can leverage straightforward masking operations to provide us with a method of targeted computation.

We do this by taking advantage of an *accumulator* or a vector that keeps track of the nodes already visited in the traversal. This vector is a dense vector and is represented in boolean format. We update it each step of the traversal typically by using a logical `OR` operation with the current frontier set.

The mask is simply the **negation** of the accumulator. We're basically saying that in our computation we want to disregard the nodes we already visited. Let's go back to the first step of our example where we start with our intial frontier set contaiing just the node \\(A\\):

$$
v = \begin{bmatrix}
1\\\
.\\\
.\\\
.\\\
.\\\
.
\end{bmatrix}
\implies
accum = \begin{bmatrix}
1\\\
0\\\
0\\\
0\\\
0\\\
0
\end{bmatrix}
$$
Now we perform first iteration to get \\(M^T \times v\\) = \\(v'\\):
$$
v' = \begin{bmatrix}
.\\\
1\\\
.\\\
1\\\
.\\\
.
\end{bmatrix}
\implies
accum' = accum | v' = \begin{bmatrix}
1\\\
1\\\
0\\\
1\\\
0\\\
0
\end{bmatrix} \implies mask = \begin{bmatrix}
0\\\
0\\\
1\\\
0\\\
1\\\
1
\end{bmatrix}
$$
Now if we now proceed with the iteration that caused us to traverse to an already visited node, you'll see the power of the mask in preventing this situation from happening:
$$
M^T \times v \odot mask = \begin{bmatrix}
1\\\
.\\\
1\\\
.\\\
1\\\
.
\end{bmatrix} \odot \begin{bmatrix}
0\\\
0\\\
1\\\
0\\\
1\\\
1
\end{bmatrix} = \begin{bmatrix}
.\\\
.\\\
1\\\
.\\\
1\\\
.
\end{bmatrix}
$$
Now, our resulting vector will only include nodes that are in the current frontier, eliminating the possibility of revisiting an already visited node. Observe the state of the graph at this stage, as it illustrates this point:

{{< mermaid >}}
graph TD
    id1(A) --> id4(D)
    id1(A) --> id2(B)
    id2(B) --> id3(C)
    id2(B) --> id1(A)
    id2(B) --> id5(E)
    id3(C) --> id4(D)
    id3(C) --> id6(F)
    id5(E) --> id6(F)
    id6(F) --> id1(A)
    id4(D) --> id5(E)
    style id1 fill:#c026d3
    style id6 fill:#7e22ce
    style id4 fill:#c026d3
    style id2 fill:#c026d3
    style id3 fill:#c084fc
    style id5 fill:#c084fc
{{< /mermaid >}}


### How do we put it all together and write a simple GraphBLAS function?

Now let's revisit my implementation from earlier.

In my implementation I am using python-graphblas, a Python library implementation of the GraphBLAS standard found [here](https://github.com/python-graphblas/python-graphblas).

Now if I were to give you 3 key concepts to keep in mind when working with GraphBLAS they'd probably be:

- Know the right semirings/operations to use for your algorithm.

- Use masks (if needed) for targeted computation.

- Understand the exit/termination conditions for the algorithm in the language of linear algebra.


In the context of using BFS to check for path existence, we already know to use the `OR-AND` semiring. We can disregard using masks since they are not needed in our use case. But what is our exit condition? Well we know that we want to return `True` if we find the node we are looking for in our frontier vector, since then we know it is accessible by an edge.
In other words we want to show \\(v_k = 1\\), where \\(k\\) is our target node.

How would we know when to return `False` however? Well, if the frontier set does not change after an iteration, it means that no new nodes were discovered. In other words, all reachable nodes from the starting node have been explored, and the target node was not among them. Thus if the frontier vector remains unchanged after an iteration we return `False`.

Looking back at my code now:

```python
def bfs_graphBLAS(M, v, target):  
    #while our target node is not in our frontier vector
    while not v[target]:
        w = v.dup()
        #calculate new frontier vector using push method 
        v << semiring.lor_land(v @ M)
        #if frontier vector does not change, no new nodes discovered
        if v.isequal(w):
            return False
    #found target in frontier vector, return True
    return True
```

## Why should we use GraphBLAS?

There are many existing libraries that make it easy to compute graph algorithms. Why does GraphBLAS matter?

1. **Efficiency in Sparse Graphs**
    - GraphBLAS defines operations over sparse matrixes and vectors, focusing only on existing edges. In most real world scenarios, the graphs we end up dealing with are sparse. GraphBLAS is specialized to handle such datastructures.
2. **Potentials for Parallelism**
    - Matrix operations are inherently parallelizable. GraphBLAS's use of matrix-vector multiplication opens up possibilities for parallel processing, potentially leading to faster computations on multi-core processors.
3. **Flexibility in Algorithm Design**
    - GraphBLAS provides a unified framework for various graph algorithms. By choosing the right semirings and operations, you can tailor the matrix multiplication to align with different algorithms, promoting code reusability.
    - For example, if we wanted our algorithim to work with weighted edges and find the single shortest path between two nodes, we'd just have to change the semiring from `OR-AND` to `MIN-PLUS`.
4. **Ease of Masking and Targeted Computation**
    - GraphBLAS offers straightforward masking operations, allowing for targeted computation and efficient handling of visited nodes.
5. **Scalability**
    - GraphBLAS's linear algebraic approach naturally scales to handle large graphs, making it suitable for big data applications.
6. **It's Easy to Write**
    - Algorithms using GraphBLAS can be effortlessly written in various languages such as C, Python, Julia, etc. Simultaneously, their performance competes with that of specialized, highly-tuned parallelized kernels.

My favorite aspect of GraphBLAS is the creative exploration of semirings. Each semiring opens up different use cases, and the process of discovering the perfect one for a particular problem is both challenging and rewarding. For instance, Tim Davis, the creator of GraphBLAS, utilized the `ANY-SECONDI` semiring to represent BFS in a [seminar](https://www.youtube.com/watch?v=hCfGCxqUgek&t=387s), while I chose the `OR-AND` semiring for my implementation.

This journey of redefining matrix multiplication in the context of graph algorithms is endlessly fascinating to me. I hope you've found something intriguing in this article as well. It's a bit lengthy, but I appreciate your time and interest in reading it. Thank you!
