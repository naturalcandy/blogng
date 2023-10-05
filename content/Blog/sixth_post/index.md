---
title: "How Does Contraction Facilitate Parallelism in Traditionally Sequential Algorithms?"
date: 2023-10-04T22:39:43-04:00
draft: false
tags: ["Technology", "Algorithms", "C++", "Optimization", "Programming", "Parallel Programming"]
summary: "In this blog post, we explore the mathematical underpinnings of contraction algorithms and their role in transforming sequential algorithms into Work efficient parallelizable constructs."
---

In my previous blog post, we talked about the limitations of purely parallel algorithms, including the challenges they pose for efficiency under circumstances when they must be executed sequentially. This blog post explores the algorithmic technique of Contraction, a specialized form of Divide and Conquer (DnC) algorithms that harmonizes parallelism with sequential efficiency.

## How can we define Contraction?

Contraction can be reduced to a simple set of steps:

{{< katex >}}
Given a problem, \\(P\\) , a Contraction algorithm has the following structure:

**Base Case**

* For trivially small instances of problem \\(P\\), directly compute and yield the solution, potentially employing an alternative algorithmic approach.

**Inductive Step(s)**
For non-trivial instances of the problem \\(P\\), we repeatedly apply the following two steps:

1. **Contract**: Transform the current instance of problem \\(P\\) into a subproblem \\(P'\\), a reduced scale verison of problem \\(P'\\).
2. **Solve**: Solve the smaller instance \\(P'\\) recursively.

Finally we expand the smaller solutions to our collective \\(P'\\) problems and combine them into our final solution. To better understand how Contraction works, let's examine its application in implementing a `reduce` function.

## How can we implement `reduce` using Contraction?

Our `reduce` function takes in a binary operation \\(f\\) and an iterable \\(S\\) as inputs, and it returns a single accumulated result by applying the operation across the elements of the iterable. For simplicity lets just assume we are trying to reduce a sequence of integers using the addition operation.

### An initial sequential approach?

Before diving into the parallelization of `reduce` using contraction, let's first examine its sequential implementation:

```cpp
template <typename T>
T reduce(Func f, const std::vector<T>& vec, T bc) {
    T result = bc;
    for (const auto& item : vec) {
        result = f(result, item);
    }
    return result;
}
```

Pretty straightforward, right? We begin with our default base-case value (which would be \\(0\\) as \\(0\\) is the identity of the addition operation). We then iterate through the sequence and accumulate our result. Since we perform a single pass through the entire sequence we know the Work (total number of steps performed assuming sequential execution) is \\(O(n)\\). And the Span (the longest chain of dependent steps when executed in parallel, assuming infinite processors), is also \\(O(n)\\), given we cannot apply any parallelism with this iterative approach.

### What are the motivations for contraction?

In sequential algorithms, results are often stored and accumulated in a manner inherently dependent on the order of operations. To parallelize such an algorithm, we first ask whether the result of one computation is needed for the next. In this particular case, we know that the order of operations does not matter, we are not forced to build up a result by folding from left to right (or in any other direction for that matter).
This is due to the very nature of the function we are applying to the sequence. In order to employ contraction we need to have a function \\(f\\) that satisifes these two rules:

1. **Associativity**: Associatvity ensures that the order of applying \\(f\\) does not matter, i.e. \\(f(a, f(b,c)) = f(f(a,b), c)\\).
2. **Identity**: Our function \\(f\\) must have an identity element, as the identity element is crucial for cases when we need to combine a single element with an "empty" value without altering the result, i.e \\(f(a, Id) = a\\).

Since addition is both associative and has an identity element, let's proceed to write an algorithm that employs Contraction to compute the reduction of a sequence of integers:
$$
<1, 2, 3, 4, 5>
$$

In contraction algorithms, the role of the function \\(f\\) is pivotal. Understanding \\(f\\) will help us determine how we choose to contract our problem. In this case \\(f\\) serves two purposes: it reduces the problem size by aggregating elements and later aids in the expansion phase, accumulating these reduced elements to construct the final solution. One effective strategy is to apply \\(f\\) to consecutive elements, thereby reducing the problem size by half in each contraction step.
$$
<f(1,2), f(3,4), f(5,Id)>\\\
\implies <3, 7, 5>
$$
We then solve this smaller instance of our problem recursively:
$$
<f(3,7), f(5,Id)>\\\
\implies <10, 5>\\\
\implies <f(10,5)>\\\
\implies<15>
$$

This implementation of this algorithim in C++ looks something like this:

```cpp
template <typename T, typename Func>
T reduceContract(Func f, T id, const std::vector<T>& a) {
    if (a.size() == 1) {
        //directly retrieve our answer
        return a[0];
    } else {
        //we will transform problem P into a subproblem P'
        std::vector<T> b;
        for (size_t i = 0; i < a.size(); i += 2) {
            //apply our function to consecutive elements. 
            if (i + 1 < a.size()) {
                b.push_back(f(a[i], a[i + 1]));
            } else {
                //uneven vector size case, combine element with identity
                b.push_back(f(a[i], id));  
            }
        }
        //recursively compute our subproblem P'
        return reduceContract(f, id, b);
    }
}

int main() {
    auto add = [](int a, int b) { return a + b; };
    std::vector<int> vec = {1, 2, 3, 4, 5};
    int result = reduceContract(add, 0, vec);
}
```

At every level of recursion we are halving our problem size due to our consecutive pairing of elements until we end up 'reducing' to our result. Since addition is associative and has an identity it does not matter the order in which we pair elements nor does it matter if if the number of elements in our vector is even or odd. 

## Contraction Vs. General Divide and Conquer?

Contraction is a more specialized version of a DnC algorithm. The core essence of DnC is definetly there but there are some differences to take note of:

1. **Number of subproblems**
    * DnC typically breaks the problem into multiple independent subproblems.
    * Contraction focuses on creating a single smaller instance of the same problem.

2. **Nature of subproblems**
    * DnC subproblems are often independent and can be solved in parallel
    * Contraction subproblems could be dependent and may need to be solved sequentially.

3. **Parallelism**
    * DnC problems can be parallelized easily due to independent subproblems.
    * Contraction can also be parallelized but may require more careful design to ensure that the dependent subproblems are handled correctly.

4. **Efficiency**
    * DnC approach may not be work efficient if the 'division' and 'combination' steps are costly.
    * Contraction approach is  more work-efficient if it reduces the problem size geometrically and the contraction and expansion steps are efficient.

Let's take a look at the general approach to the DnC approach for implementing our `reduce`:

```cpp
template <typename T, typename Func>
T reduceDnC(Func f, T id, const std::vector<T>& a, size_t start, size_t end) {
    // Base case: If the range contains only one element, return it.
    if (end - start == 1) {
        return a[start];
    }
    
    // Base case: If the range is empty, return the identity element.
    if (end == start) {
        return id;
    }

    // Divide step: Find the middle of the range.
    size_t mid = start + (end - start) / 2;

    // Conquer step: Recursively solve the subproblems.
    T left = reduceDnC(f, id, a, start, mid);
    T right = reduceDnC(f, id, a, mid, end);

    // Combine step: Combine the solutions of the subproblems.
    return f(left, right);
}
```

As you can see, in our DnC approach we divide a problem into multiple subproblems whereas in contraction we just reduce the size of the problem instance in each recursive call. Although both algorithms exhibit a \\(O(logn)\\) span, their work complexities differ significantly, leading to noticeable performance variations.

### Work Complexity Analysis

The Work recurrence for `reduceDnC` given an input size of \\(n\\) looks something like this:
$$
W(n) = 2 * W(n / 2) + c_k
$$
Or in other words at every level of recursion we make two recursive calls on problem sizes that are half the size of our original problem plus some constant work being done (\\(c_k\\)).

A node representing a problem state at a given level \\(i\\) (parent node) will take about \\(c_k\\) work (the non-recursive portion of our recurrence). While the children nodes at level \\(i + 1\\)—since we are recursively calling twice for each parent at level \\(i\\), will take about \\(c_k * 2\\) work. Thus we know the recurrence will be dominated by the work done at the children nodes—the recurrence is "leaf dominated"—and in order to find the total cost expressed by this recurrence we can just calculate the total cost of all the children nodes.

Given we are halving our input size at every recursive level we will have \\(logn\\) levels of recursion. At each level we will have \\(2^i\\) instances of our problem with each of them being size \\(n/2^i\\). Thus the cost of one level is: \\(2^i * n/2^i = n\\). Note that we have \\(logn\\) levels with each level costing about \\(n\\): the overall work complexity is in \\(O(nlogn)\\).

The Work recurrence for `reduceContract` given an input size of \\(n\\) looks something like this:

$$
W(n) = W(n / 2) + O(n) + c_k \\\
\implies W(n / 2) + c_m * n + c_k
$$
{{< alert "circle-info" >}}
The second step follows by the definition of [asymptotic complexity](https://en.wikipedia.org/wiki/Asymptotic_computational_complexity), where \\(c_m\\) and \\(c_k\\) are constants.
{{< /alert >}}

A node representing a problem state at a given level \\(i\\) (parent node) will take about \\(n\\) work. While the node for its only child at level \\(i + 1\\)—since we are recursively calling once—will take about \\(n/2\\) work. Thus, we can conclude that the overall work of our Contraction approach is dominated by the root node, resulting in an \\(O(n)\\) work complexity. A lot better than \\(O(nlogn)\\)!

### Why does work efficiency matter in parallel algorithms?

{{< alert "circle-info" >}}
Quick reminder: Work refers to the total number of steps needed for sequential execution. Span, on the other hand, is the longest chain of dependent steps when executed in parallel, assuming infinite processors. Both can be upper-bounded with asymptotic analysis.
{{< /alert >}}

A common question that may arise is why work efficiency matters when the focus is on parallel execution. Implementing `reduce` using both DnC and Contraction yields a \\(O(logn)\\) Span, so does the choice between the two matter?

The distinction lies in efficiency and resource constraints. Span's assumption of infinite processors is often unrealistic; we will not always have a processor available to execute a computation for a problem/subproblem. Additionally, DnC can incur extra overhead by generating multiple subproblems, whereas Contraction focuses on solving a single, reduced subproblem. For more on the practical limitations of parallel algorithms, see my earlier [blog post](../fifth_post/).

## C++ reduce Vs. accumulate? A real example of the benefits of contraction.

In C++, `std::accumulate` is essentially a left fold operation, as it processes elements in a strictly left-to-right order. It's pretty much our purely sequential approach. But introduced in C++17, we have `std::reduce`, which allows for unordered and potentially parallelized execution. However a pre-requirement is that the operation we use for our reduction must be associative and have an identity, otherwise the behavior of `std::reduce` is undefined.

{{< alert "lightbulb" >}}
Does the requirement for associativity and identity sound familiar? Yes! `std::reduce` is actually implemented using Contraction under the hood.
{{< /alert >}}

A performance comparison available on [cppreference](https://en.cppreference.com/w/cpp/algorithm/reduce) reveals a significant increase in efficiency when using `std::reduce` over `std::accumulate`:

```json
// POSIX: g++ -std=c++23 ./example.cpp -ltbb -O3; ./a.out
std::accumulate (double)    sum: 10,000,000.7    time: 356.9 ms
std::reduce (seq, double)   sum: 10,000,000.7    time: 140.1 ms
std::reduce (par, double)   sum: 10,000,000.7    time: 140.1 ms
std::accumulate (long)      sum: 100,000,007     time: 46.0 ms
std::reduce (seq, long)     sum: 100,000,007     time: 67.3 ms
std::reduce (par, long)     sum: 100,000,007     time: 63.3 ms
 
// POSIX: g++ -std=c++23 ./example.cpp -ltbb -O3 -DPARALLEL; ./a.out
std::accumulate (double)    sum: 10,000,000.7    time: 353.4 ms
std::reduce (seq, double)   sum: 10,000,000.7    time: 140.7 ms
std::reduce (par, double)   sum: 10,000,000.7    time: 24.7 ms
std::accumulate (long)      sum: 100,000,007     time: 42.4 ms
std::reduce (seq, long)     sum: 100,000,007     time: 52.0 ms
std::reduce (par, long)     sum: 100,000,007     time: 23.1 ms
```
