---
layout: post
title: Recursion, dynamic programming, and memoization
---

## Background and motivation 

In computer science, a <i>recursive</i> definition, is something that is defined in terms of itself. More formally, recursive definitions consist of

1. A simple **base case**, or termination step that cannot be reduced further
2. One or more **recursive cases** that reduce the problem toward the base case

The [factorial function](https://en.wikipedia.org/wiki/Factorial) is a great example of a function that is defined recursively. 5! (read 5 "factorial", not "FIVE!") is computed by multiplying all the whole numbers between one and five inclusive (5 x 4 x 3 x 2 x 1). The value of 0! is one, by convention. 

Let's work out a recursive definition for the factorial function. The base case must be 0! as this can't be reduced further. What about the recursive case? Take a look at 5! a little more closely. 5! is just 5 * 4! (since 4! is 4 x 3 x 2 x 1). But what's 4! Well that's just 4 x 3! We can keep doing this until we get to 0! which we know to be equal to one by definition. 

Once you get around the idea of a function calling itself, it's pretty easy to implement the above algorithm in Python. Here we'll ignore the case where the input number is negative. 

{% highlight python %}
def factorial(n):
    return 1 if n == 0 else n * factorial(n-1)

>>> factorial(5)
120
{% endhighlight %}

Exactly what we want! 

Another famous recursive function produces the numbers in the below sequence.

{% highlight python %}
1, 1, 2, 3, 5, 8, 13, 21, 34, 55

{% endhighlight %}

These are of course the first ten digits in the [Fibonacci sequence](https://en.wikipedia.org/wiki/Fibonacci_number) where the definition for the next number is always the sum of the two preceding numbers (the first and the second Fibonacci numbers are one). We can again easily implement this recursive definition with one line of Python.  

{% highlight python %}
def fibonacci(n):
    return 1 if n == 0 or n == 1 else fibonacci(n-1) + fibonacci(n-2)

>>> for n in range(10): print(fibonacci(n))
  
1
1
2
3
5
8
13
...
>>> 
{% endhighlight %}

Try running the above code on input sizes greater than 30. You'll notice that the time to compute <code>fibonacci(n)</code> takes considerably longer with each increase in <code>n</code>.

## Introducing dynamic programming

What's the problem? It's not that recursion itself is bad (although in Python, for more technical reasons, recursion can be inefficient). The problem is that our code guarantees that some inputs will get computed <i>twice</i>. Consider, for example, computing <code>fibonacci(5)</code>.

{% highlight python %}
>>> fibonacci(5)
fibonacci(5)
  fibonacci(4)
    fibonacci(3)
      fibonacci(2)
        fibonacci(1)
        fibonacci(0)
      fibonacci(1)
    fibonacci(2)
      fibonacci(1)
      fibonacci(0)
  fibonacci(3)
    fibonacci(2)
      fibonacci(1)
      fibonacci(0)
    fibonacci(1)
{% endhighlight %}

The two calls to <code>fibonacci(3)</code> aren't just two function calls but rather a whole <i>tree</i> of calls all the way down to <code>fibonacci(1)</code> and <code>fibonacci(0)</code>. This makes sense, of course; <code>fibonacci(3)</code> isn't the base case, and we have to recursively compute it. 

Computing anything twice is unnecessary. What if we were instead able to save the result of <code>fibonacci(3)</code> the first time we compute it, and somehow use it the next time it's needed? We already know that <code>fibonacci(0)</code> and <code>fibonacci(1)</code> must return <code>1</code>. Computing <code>fibonacci(2)</code> only needs to look at <code>fibonacci(1)</code> and <code>fibonacci(0)</code>. 

This is the key insight: while computing <code>fibonacci(n)</code> we can keep the computed Fibonacci numbers in an array (call it <code>a</code>), indexed by <code>n</code>, where <code>a[0] = a[1] = 1</code> (the base case). While iterating up to <code>n</code>, to get the next Fibonacci number, all we have to do is add <code>a[n-1]</code> to <code>a[n-2]</code> and the value for <code>fibonacci(n)</code> will be kept at the last index in the array. 

{% highlight python %}
def fibonacci(n):
    a = [1, 1]
    for i in range(1, n):
        a.append(a[i] + a[i-1])
    return a[-1]

>>> fibonacci(100)
573147844013817084101
{% endhighlight %}

Wow! Much better!

The idea of storing previous computed values rather than recomputing them is called [dynamic programming](https://en.wikipedia.org/wiki/Dynamic_programming) or DP, for short. In this context, the word "dynamic" should signify that we're <i>building up</i> the solution to a bigger problem from a smaller one. 

### The knapsack problem

Let's take a look at another example, the so called [knapsack problem](https://en.wikipedia.org/wiki/Knapsack_problem). Given a set of items, each with a weight and a value, a solution to the knapsack problem determines which subset of items to include in a knapsack such that the total knapsack weight is less than or equal to a given limit and the total value of the knapsack is as large as possible. The items cannot be broken apart and can only be taken once. 

If we have no items to take, our solution is empty (no items, no weight, no value). If, however, we have items to take, there are two possibilities to consider when choosing whether or not to pack an item. If the item and the current knapsack weight are less than the weight limit, we can either take the item, thereby increasing our total value and weight, or leave the item behind (we don't have to take it just because we can). The second possibility forces the decision on us: if instead the item puts us over the capacity, we can't take it, and it must be left behind. 

This sounds like a recursive solution. We look at the first item in the available set, and solve the problem on the remaining items recursively. The choice that produces the best total value wins and is returned. 

{% highlight python %}
from collections import namedtuple

Item = namedtuple("Item", "name weight value".split())
Solution = namedtuple("Solution", "taken weight value".split())

def kp(items, C):
    """ 
    Solves the 0/1 knapsack problem by brute force 
    where items is a list of type Item and C is 
    the problem weight constraint 
    """
    def inner(taken, weight, value, available):
        if not available:
            return Solution(taken, weight, value)
        
        first = available[0]
        items = available[1:]
        
        if weight + first.weight <= C:
            l = inner(taken + (first,), weight + first.weight,
                      value + first.value, items)

            r = inner(taken, weight, value, items)

            return l if l.value > r.value else r
        return inner(taken, weight, value, items)
    return inner(tuple(), 0, 0, items)
{% endhighlight %}

Using the items list from [rosettacode](http://rosettacode.org/wiki/Knapsack_problem/0-1), we get the right answer but our code takes an alarmingly long time to complete. The problem is essentially the same as that which we encountered in our first implementation of Fibonacci. At each level in the recursion, given that there are two decisions to make (include the item, or leave it behind) how many possible configurations are there to examine? 

The correct answer is 2 <sup>#items</sup> and it quickly gets impractical to solve the knapsack problem with this algorithm for large input sizes. However, provided that the knapsack weight and the item values are both integral, we can develop a similar array based solution not dissimilar from the approach we took to improve the performance of our Fibonacci function. 

Consider the problem of filling a knapsack with capacity 7 with the item-weight pairs (1,1), (3,4), (4,5), (5,7). Rather than solving the problem of whether or not to pack the first item for a knapsack of capacity 7, let's enumerate the possibilities for taking item one for knapsacks of <i>all</i> weights between 0 and 7 inclusive. If the total weight of the knapsack is 0, the item can't be packed in the knapsack (it has weight 1). For all other knapsacks of weight 1 through 7, the total weight is always less than or equal to the weight of the first item and we can guarantee a value of 1 for all knapsacks by taking item one.

Things get more interesting with the second item. If the knapsack capacity is less than 3, we can't take the item; the best we can do is take item 1, since it's weight is less than the total capacity. With 4 units of capacity we can take item 2 with 1 extra unit of capacity to spare. For a knapsack of capacity 1, we just figured out that the maximum value is got by taking item 1. 

This is the key idea: reuse solutions to smaller problems that have already been solved optimally. We can systematize this decision process by creating a table where the columns correspond to the integral knapsack capacities between 0 and 7 inclusive. The number of rows in the table is just the number of items, since the items themselves cannot be broken apart. To decide whether or not to pack an item, we just look at how much weight is left <i>after</i> taking the item. We've already solved that problem, and can lookup what total value we can get using that solution. As before, we compare the gain in value from taking an item with that got by leaving it behind. The best decision for not including the item will always be in the cell of the preceding row since that row contained the best solutions so far without including the current item under consideration.

It helps to sketch out this table for yourself to get a handle for filling things in. If you're interested, the video corresponding to this example can be found [here](https://www.youtube.com/watch?v=8LusJS5-AGo).

The full code is pretty straightforward once you understand why and how the table works. My implementation isn't ideal, but I wanted something quick and dirty to illustrate the algorithm. In a real application, the items taken need not be recorded in each cell, and can be deduced by iterating over the table in reverse. 

{% highlight python %}
def dp(items, C):
    """ 
    Solves the 0/1 knapsack problem by dynamic programming
    where items is a list of type Item and C is 
    the problem weight constraint 
    """
    n = len(items)
    
    first = items[0]
    table = ([[Solution(tuple(), 0, 0) if first.weight > capacity
               else Solution((first,), first.weight, first.value) for capacity in range(C+1)]] +
             
             [[0] * (C+1) for _ in range(n-1)])

    for i in range(1, n):
        item = items[i]
        for capacity in range(C+1):
            if item.weight > capacity:
                table[i][capacity] = table[i-1][capacity]
            else:
                table[i][capacity] = max(table[i-1][capacity],
                                         Solution(table[i-1][capacity - item.weight].taken  + (item,),
                                                  table[i-1][capacity - item.weight].weight + item.weight,
                                                  table[i-1][capacity - item.weight].value  + item.value),
                                         key = lambda c : c.value)
    return table[-1][-1]
{% endhighlight %}

The true insight of dynamic programming lies in recognizing when a solution can be reused in the service of computing something else. Unfortunately, the two problems we've worked through show that we often need to scrap our basic recursive algorithms altogether in favor of an iterative solution.

## Introducing memoization
It turns out that there's a more flexible implementation of dynamic programming that prevents us from having to completely change our algorithm. The idea is to cache, or [<i>memoize</i>](https://en.wikipedia.org/wiki/Memoization) the results of previous calculations in a dictionary indexed by the arguments of each function call.

With this in mind, let's revisit the first Fibonacci definition, this time leaving the recursive algorithm but adding a dictionary to cache the results.

{% highlight python %}
def fibonacci(n):
    d = {}
    def inner(n):
        if n == 0 or n == 1: return 1
        if n not in d:
            d[n] = inner(n-1) + inner(n-2)
        return d[n]
    return inner(n)
{% endhighlight %}


The above solution is appealing because we didn't have to deduce how to fill out the data structure holding previous solutions. Memoization also has the added benefit of not wasting space saving solutions that are never needed (or rather correspond to arguments that are never computed). What's more, we can write a general purpose memoization utility function to take care of the caching for us. 

{% highlight python %}
def memoize(f):
    """ Returns a memoized function, f """
    d = {}
    def inner(*args):
        if args not in d:
            d[args] = f(*args)
        return d[args]
    return inner # returning a pointer to the inner function!
{% endhighlight %}

In the above function, we memoize the arguments passed to <code>inner</code>. Since <code>inner</code> lies inside <code>memoize</code>, the dictionary <code>d</code> will be available to whatever consumes the returned <code>inner</code> function (the more technical term for the code we've written is a [function closure](https://en.wikipedia.org/wiki/Closure_(computer_programming))). With our general purpose memoizing function in hand, here's how we now might memoize the original <code>fibonacci</code> function.

{% highlight python %}
>>> def fibonacci(n):
        return 1 if n == 0 or n == 1 else fibonacci(n-1) + fibonacci(n-2)
>>> fibonacci = memoize(fibonacci)
>>> fibonacci(100)
573147844013817084101
{% endhighlight %}

Even better, Python's built in [decorator](https://www.python.org/dev/peps/pep-0318/) syntax actually supports this kind of function wrapping natively. 

{% highlight python %}
@memoize
def fibonacci(n):
    return 1 if n == 0 or n == 1 else fibonacci(n-1) + fibonacci(n-2)
{% endhighlight %}

This the most Pythonic solution I can think of.

### Party optimization 

Here's another problem that can be solved concisely by applying memoization to a naturally recursive algorithm. 

Suppose a company's organization chart is a [binary tree](https://en.wikipedia.org/wiki/Binary_tree). Each employee has an associated "fun value", and we want to maximize the total fun value (sum of the fun values of the invited employees). There's only one constraint: we can either invite a supervisor, and exclude the supervisor's subordinates, or we can invite the subordinates and exclude the supervisor. 

Given an employee in the tree how do we determine whether or not to include him or her? Simple! If we decide to include, it must be the case that the direct subordinates are excluded from the solution. If, on the other hand, we've excluded someone, we need to consider both the case of including <i>and</i> excluding the left and right subtrees (both are valid possibilities). We select the choice that gives the greatest fun value.  

In my solution, I defined two data structures to better specify what specifically should be cached. 

{% highlight python %}
import namedtuple

Tree = namedtuple("Tree", ["name", "fun_value", "left", "right"])
Solution = namedtuple("Solution", ["fun_value", "invited"])
{% endhighlight %}

This is important. Don't just assume that subproblems will get cached correctly and be sure to check that what's being cached is a valid solution to a subproblem. It's also particularly advantageous to use the <code>namedtuple</code> type here because it's hashable and can be used as the key in a dictionary. Here's the full code.

{% highlight python %}
@memoize
def party(tree):
    """ 
    Solves party optimization problem by memoization 
    where tree is of type Tree
    """
    def include(tree):
        if not tree:
            return Solution(0, tuple())
        
        l = exclude(tree.left)
        r = exclude(tree.right)
        return Solution(tree.fun_value + l.fun_value + r.fun_value,
                        l.invited + r.invited + (tree.name,))

    def exclude(tree):
        if not tree:
            return Solution(0, tuple())
        
        l = party(tree.left)
        r = party(tree.right)
        return Solution(l.fun_value + r.fun_value,
                        l.invited + r.invited)

    return max(include(tree), exclude(tree), key = lambda t: t.fun_value)
{% endhighlight %}

This algorithm scales great for trees of arbitrary depth. Here's some code I used to generate random trees.

{% highlight python %}
import random

def random_tree(depth=10, N=100):
    if depth <= 0:
        return None
    
    return Tree(str(depth), random.randint(0, N),
                random_tree(depth-1, N),
                random_tree(depth-1, N))
{% endhighlight %}

The results on a tree 12 levels deep illustrate how truly powerful dynamic programming can be.

{% highlight python %}
--- 10.0826981067657471 seconds (brute force) ---
--- 0.06369280815124512 seconds (memoization) ---
{% endhighlight %}

### Revisiting our solution to the knapsack problem 

It might be tempting to toss <code>@memoize</code> on top of the recursive algorithm we wrote to solve the knapsack problem. Doing so, however, won't actually improve the performance of our code. 

In our algorithm, we're caching the arguments of the <code>inner</code> function in <code>kp</code> (items taken, current weight, current value, items available). This isn't really the same information we kept track of in our array based solution. Rather, in that algorithm we only cared about the items available to take, and the remaining capacity after taking an item. 

This underscores how important it is to actually understand what <code>@memoize</code> will store. I won't provide a memoized solution here, but instead refer anyone interested to [this implementation on codereview](http://codereview.stackexchange.com/questions/20569/dynamic-programming-solution-to-knapsack-problem/20581#20581). It's essentially the same array based solution we saw before but uses a dictionary and recursion instead.

## Final thoughts
To recap. We worked out two methods for improving the performance of recursive algorithms with overlapping subproblems: we can either iteratively maintain partial solutions in a table, or we can cache the results of recursive calls in a dictionary. Both illustrate dynamic programming, or the method of solving a bigger problem by breaking the problem into a collection of simpler subproblems that have been solved optimally. 


Thanks for reading!



