---
layout: post
title: Writing a simple LISP interpreter in C#
---

A few months ago, I decided to develop an interpreter for a subset of the LISP programming language. This was a great exercise, and I’d like to share a few things I learned along the way. 

### Full code and where we’re going

By the end of this blog you'll know how to implement a simple interpreter. And your code will be able to interpret code like this.

{% highlight racket %}
> (define fib (lambda (n) ;; Fibonacci!
    (if (< n 2) 
      1 
  (+ (fib (- n 1)) (fib (- n 2))))))

> (fib 10)
55
{% endhighlight %}

For reference, the final project can be found [here](https://github.com/gjdanis/Licsp). 

## Background
An interpreter is a program that reads raw text in one language and translates that text into something executable in another language. It does this by constructing an [abstract syntax tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree) (or "AST" for short), an object representation that captures the execution semantics of the raw text. The tree is then traversed by an evaluation algorithm and the result is returned. 

This is different from, although similar to, the process used by a compiler to execute code. A compiler, however, will typically try to optimize your code.[^fn-switch_compile] Because of this optimization step, a representation of the compiled code needs to be saved to the disk for later use. In contrast, interpreters execute the input text in memory. 

It should be noted that compilers and interpreters are not a feature of the language but rather a feature of the language implementation. One could, for example, implement the C programming language with an interpreter, and the Python programming language with a compiler. The choice of whether or not to implement a language with an interpreter or a compiler is strictly up to the developer.

## Parsing

Unlike infix notation (operators appear <i>in-between</i> operands), LISP is a relatively easy language to parse. Since everything is fully parenthesized, we don't need to pay special attention to operator precedence and the order of operations while parsing. Rather, we're more or less given the AST and it's up to us to translate the tree into a set of syntax classes by looking at the first element in each parenthesized expression.[^fn-s_expressions]

When parsing the text, we have two types to consider; atomics, (indivisible values like numbers and bools), and lists which contain multiple expressions. If we see an atomic, there’s nothing more to do — we simply return an object that holds the atomic data (the raw string). 

If we see an opening parenthesis, we know that we’re populating a list. We should continue parsing from the next token in the input stream, and stop once we see a closing parenthesis. At that point, we can return the list. 

In my implementation, I defined two types, <code>SExpAtom</code> and <code>SExpList</code>, both of which derive from <code>SExp</code>, an abstract class that is used to group the two child classes (more formally, the LISP syntax uses [<i>s-expressions</i>](https://en.wikipedia.org/wiki/S-expression)). I then apply the recursive algorithm described above to the input text. 

{%highlight csharp linenos %}
public static SExp Parse(string input)
{
    var tokens = input
        .Replace("(", " ( ")
        .Replace(")", " ) ")
        .Split().Where(x => x != "").ToList();

    return Next(tokens);
}

private static SExp Next(List<string> tokens)
{
    var token = tokens.Pop(); // removes first element
    if (token == ")") throw new Exception("Unexpected '(' character");

    if (token == "(")
    {
        var list = new SExpList();
        while (tokens.First() != ")")
            list.Contents.Add(Next(tokens));
        tokens.Pop();
        return list;
    }

    return new SExpAtom(token);
}
{% endhighlight %}

## AST

The preceding step produced a structure that is far easier to process than the input text. With a valid <code>SExp</code> in hand, we can proceed to the next step, translating the <code>SExp</code> tree to an abstract syntax tree.

The base type for all syntax classes is <code>Expression</code>. 

{%highlight csharp linenos %}
abstract class Expression
{
    public abstract Value Evaluate(Dictionary<string, Expression> env)
}
{% endhighlight %}

This base class requires that subclasses provide an implementation for the <code>Evaluate</code> method. <code>Evaluate</code> produces a <code>Value</code>, a derived type of <code>Expression</code> that evaluates to itself. 

{%highlight csharp linenos %}
abstract class Value : Expression 
{
    public Value(object underlying)
    {
        Underlying = underlying;
    }

    public T GetValue<T>()
    {
        return (T)Underlying;
    }

    public override Value Evaluate(Dictionary<string, Expression> env)
    {
        return this;
    }

    protected readonly object Underlying; 
}
{% endhighlight %}

Don't worry about the argument of <code>Evaluate</code>. We'll discuss that and the overrides of <code>Evaluate</code> in detail later. I'll further show how we can improve the design by hiding <code>env</code> from direct exposure to the syntax classes. We'll also make <code>Value</code> a generic class, to avoid having to cast <code>Underlying</code>.

Our interpreter will support two <code>Value</code> types: <code>Bool(bool b)</code> and <code>Number(double x)</code>. The other expressions are as follows

* <code>Call(Expression body, List&#60;Expression&#62; arguments)</code> represents a function call e.g. <code>(myfunc 1 2 3)</code>

* <code>Define(string name, Expression expression)</code> represents a variable definition e.g. <code>(define name e1)</code>

* <code>If(Expression test, Expression trueBranch, Expression elseBranch)</code> represents conditional execution e.g. <code>(if (e1) e2 e3)</code>

* <code>Lambda(Expression body, List&#60;string&#62; parameters)</code> represents an anonymous function e.g. <code>(lambda (n) (* 2 n))</code>

* <code>ListFunction(string op, List&#60;Expression&#62; arguments)</code> represents a built-in operation on a list of arguments e.g. <code>(+ 1 2 3)</code>

* <code>Variable(string name)</code> represents a variable reference to lookup the <code>Expression</code> stored at <code>name</code> e.g. <code>x</code>

Given an <code>SExp</code>, or goal will be to turn the <code>SExp</code> into one of the above types. If we’re given an <code>SExpAtom</code>, we need to parse the string token to a <code>Value</code> and return the result. 
{%highlight csharp linenos %}
private static Expression From(SExpAtom atom)
{
    double x;
    if (double.TryParse(atom.Value, out x))
        return new Number(x);
    if (atom.Value == "#t")
        return new Bool(true);
    if (atom.Value == "#f")
        return new Bool(false);
    return new Variable(atom.Value);
}
{% endhighlight %}


If we’re given an <code>SExpList</code>, we need to examine the first element of the list. If it’s an <code>SExpAtom</code>, it’s string token should parse to one of the syntax keywords, (for example <code>if</code> or <code>lambda</code>) which will tell us how to proceed parsing the rest of the contents of the list. If it doesn't, we’ll return a <code>Call</code> as the string should be a variable reference. 

{%highlight csharp linenos %}
private static Expression From(SExpList root)
{
    // "Require" throws an exception if the test condition is not satisfied

    var head = root.Contents.Pop(); 
    if (head is SExpList)
        return new Call(From(head), root.Contents.Select(x => From(x)).ToList());

    var cast = head as SExpAtom;

    switch (cast.Value)
    {
        case "+":
        // ... similar for -, *, /, =, etc
            return new ListFunction(cast.Value, root.Contents.Select(x => From(x)));

        case "lambda":
            Require(root.Contents.First() is SExpList,
                "\"lambda\" statements should be followed by list of parameters");

            var parameters = (SExpList)root.Contents.Pop();
            Require(parameters.Contents.All(x => x is SExpAtom),
                "\"lambda\" statements should be followed by list of string parameters");

            var body = From(root.Contents.Pop());
            return new Lambda(body, parameters.Contents.Select(x => (x as SExpAtom).Value));

        case "if":
            Require(root.Contents.Count == 3,
                "\"if\" statements should be followed by three expressions");
            return new If(From(root.Contents.Pop()),
                From(root.Contents.Pop()), From(root.Contents.Pop()));

        case "define":
            Require(root.Contents.First() is SExpAtom,
                "\"define\" statements should be followed by string identifier");
            var atom = (SExpAtom)root.Contents.Pop();
            return new Define(atom.Value, From(root.Contents.Pop()));

        default:
            return new Call(From(cast), root.Contents.Select(x => From(x)).ToList());
    }
}
{%endhighlight %}

One final remark: if the first element is not an <code>SExpAtom</code>, it's an <code>SExpList</code> and must also be parsed to a <code>Call</code>. In this case, the function call must <i>return</i> a function that will be applied to the rest of the arguments in the list. For example, the inner expression of <code>((myfunc 1) 2)</code> returns a function that is then called on <code>2</code>. 

## Evaluation

Great! We have an AST that can be evaluated!

I mentioned earlier that we'd save discussion of the <code>env</code> argument in the <code>Evaluate</code> method for later. This is simply the evaluation <i>environment</i>, the place <code>Define</code>expressions put their definitions. You can think of this dictionary as the "context" that an <code>Expression</code> in the tree has access to during evaluation. 

Evaluation is pretty straightforward in most cases. If an <code>Expression</code> contains sub-expressions, evaluate those to <code>Value</code> types first, and then perform the required operations to coalesce the sub-expressions to a <code>Value</code>. We'd do this for evaluating a <code>ListFunction</code>, for example. 

{%highlight csharp linenos %}
// Evaluate method for "ListFunction"
public override Value Evaluate(Dictionary<string, Expression> env)
{
    switch (Op)
    {
        case "+":
            var nums = Arguments.Select(x => 
                x.Evaluate(env).GetValue<double>());
            return new Number(nums.Sum());

        // cases for other Number/Bool operations
    }
    throw new NotImplementedException();
}
{%endhighlight %}

The final code is a little cleaner. I put the operation methods in an extension class and have a separate function to evaluate <code>Arguments</code> down to a <code>Number</code>. At least you have the idea for now.

<code>If</code> statements evaluate <code>Test</code> to a <code>Bool</code> and branch as expected. <code>Define</code> statements put their definition in the current environment, and return null. <code>Variable</code> expressions conversely return the <code>Expression</code> located at <code>Name</code>.

{%highlight csharp linenos %}
// Evaluate method for "If"
public override Value Evaluate(Dictionary<string, Expression> env)
{
    return Test.Evaluate(env).GetValue<bool>() ? 
        TrueBranch.Evaluate(env) : ElseBranch.Evaluate(env);
}

// Evaluate method for "Define"
public override Value Evaluate(Dictionary<string, Expression> env)
{
    env[Name] = Expression;
    return null;
}

// Evaluate method for "Variable"
public override Value Evaluate(Dictionary<string, Expression> env)
{
    return env[Name].Evaluate(env);
}
{%endhighlight %}

It's easy enough to check how we're doing. 

{%highlight csharp linenos %}
public static void Main(string[] args)
{
    var env = new Dictionary<string, Expression>();
    var expression = Parser.Parse("(+ 2 (+ 1 1))");
    double val = expression.Evaluate(env).GetValue<double>();

    expression = Parser.Parse("(define x #t)");
    expression.Evaluate(env);

    expression = Parser.Parse("(if x (+ 1 1) 0)");
    val = expression.Evaluate(env).GetValue<double>();
}
{%endhighlight %}

The most complicated (and interesting!) evaluation case is that of functions and function calls. <code>Lambda</code> expressions evaluate to <code>Closure</code> expressions, a <code>Value</code> type we haven’t discussed yet. A <code>Closure</code> simply consists of a <code>Lambda</code> and a copy of the environment in which the function was defined. 

If you think about it, this is no different than the type of nesting that needs to happen whenever we evaluate functions in any other language. Moreover, functions have their own <i>local</i> environment that exists separate from whatever top-level environment in which the function is called. If we didn’t support this type, we’d have no way to store the variables that get defined in a function. A variable <code>n</code> that is defined at the global level relative to the function definition and defined again inside a function should leave the outer <code>n</code> untouched.

Our parsing algorithm actually introduced the idea of closures earlier when I discussed the procedure for parsing an <code>SExpList</code> where the first element is <i>not</i> an <code>SExpAtom</code>. In the example <code>((myfunc 1) 2)</code> the expression <code>(myfunc 1)</code> must <i>return</i> a function to apply to <code>2</code>. But really we must return a function <i>with</i> the context of everything defined <i>inside</i> <code>myfunc</code> as any definitions should be available when we apply the returned function on <code>2</code>. This is exactly what a <code>Closure</code> is; a function and an environment. 

Evaluation of a <code>Call</code> thus evaluates its <code>Body</code> to a <code>Closure</code>, and extends the environment of the <code>Closure</code> with the evaluation-time arguments of the <code>Call</code>. The body of the <code>Lambda</code> in the <code>Closure</code>is then evaluated in the environment of the <code>Closure</code>. 

{%highlight csharp linenos %}
// Evaluate method for "Lambda" 
public override Value Evaluate(Dictionary<string, Expression> env)
{
    return new Closure(this, env);
}

// Evaluate method for "Call"
public override Value Evaluate(Dictionary<string, Expression> env)
{
    var closure = Body.Evaluate(env).GetValue<Closure>();
    var runtime = Arguments.Select(x => x.Evaluate(env));

    foreach (var pair in closure.Lambda
        .Parameters.Zip(runtime, (p, arg) => new {p, arg}))
    {
        closure.Environment[pair.p] = pair.arg;
    }
    return closure.Lambda.Body.Evaluate(closure.Environment);
}
{%endhighlight %}

That’s it! A little complicated, yes, although there’s not a lot of code needed to implement correct evaluation of <code>Call</code> expressions. <code>Closures</code> seem conceptually different from any of the other Expressions we’ve discussed. But really they're just a <code>Value</code> type; they're the <code>Value</code> type that functions must evaluate to in order for the function's body to be properly scoped.

It doesn't hurt to throw in a few tests. Looks like I delivered on that promise at the start of this post!

{%highlight csharp linenos %}
public static void Main(string[] args)
{
    expression = Parser.Parse(
        @"(define fact (lambda (n) (if (< n 1) 1 (* n (fact (- n 1))))))");
    expression.Evaluate(env);

    expression = Parser.Parse("(fact 5)");
    val = expression.Evaluate(env).GetValue<double>();

    expression = Parser.Parse(
        @"(define fib (lambda (n) (if (< n 2) 1 (+ (fib (- n 1)) (fib (- n 2))))))");
    expression.Evaluate(env);

    expression = Parser.Parse("(fib 10)");
    val = expression.Evaluate(env).GetValue<double>();
}
{%endhighlight %}

## Improving our design

Our current design requires each syntax class know how to evaluate itself. While this approach is fairly OOP, it’s not particularly scalable. If we wanted to add any other set of operations, for example type inference or pretty printing, we'd have to change the definition of <code>Expression</code> and re-compile/retest all the AST classes. Our current code also exposes the evaluation environment to the syntax classes directly. If you think about it, the AST classes should really just hold expressions, and do nothing else. They don't need to know anything about a environments and shouldn't be able to modify the evaluation context at will. 

What I’m getting at is that it would be better if we put the evaluation algorithm in a separate function, and not have it distributed across the AST classes. In a functional programming language, the approach would be to switch on the <code>Expression</code> types using pattern matching. We can do that in C# by using the <code>is</code> keyword and casting to the correct type (not ideal). 

Fortunately there’s a design pattern that addresses this problem exactly. It’s called the [visitor pattern](https://en.wikipedia.org/wiki/Visitor_pattern). There are visitors, which implement a <code>Visit</code> method for all types in the tree, and visitables, classes that can accept an implementation of the visitor interface through implementations of an <code>Accept</code> method. When a visitable accepts a visitor, all it does is call the visitor's <code>Visit</code> implementation for the visitable’s type. It’s now the visitor’s responsibility to choose what to do next.

## Refactored implementation of the evaluation algorithm 

The first step in refactoring our code is to provide an interface for all operations on the tree. 

{%highlight csharp linenos %}
interface IExpressionVisitor
{
    void Visit(Call exp);
    void Visit(Define exp);
    void Visit(If exp);
    void Visit(Lambda exp);
    void Visit(ListFunction exp);
    void Visit(Variable exp);
    void Visit(Bool exp);
    void Visit(Closure exp);
    void Visit(Number exp);
    void Visit(Literal exp);
}
{% endhighlight %}

This of course makes perfect sense; we want to visit every variant of the <code>Expression</code> type. Our base type correspondingly has to change to accept an <code>IExpressionVisitor</code>. Easy enough.

{%highlight csharp linenos %}
abstract class Expression
{
    public abstract void Accept(IExpressionVisitor v);
}
{% endhighlight %}

Finally, when an <code>Expression</code> accepts a visitor, all it needs to do is tell the visitor to visit it. The type does this by calling <code>Visit</code> on <i>itself</i>.

{%highlight csharp linenos %}
public override void Accept(IExpressionVisitor v)
{
    v.Visit(this);
}
{% endhighlight %}

The above code shows how the visitable tells the visitor to "visit me." This design pattern is essentially a switch statement on the types of the visitables. It’s like we’re using the <code>is</code> keyword to determine how to proceed when given an <code>Expression</code>. The nice thing about the pattern, however, is that we don't then need to cast to target type. As an example, here is how we would visit a <code>Call</code> expression. 

{%highlight csharp linenos %}
// "Call" just accepted me; here's how I visit a "Call"
public void Visit(Call exp)
{
    exp.Body.Accept(this);
    var closure = (Closure)evaled.Pop(); // evaluation stack
    var runtime = Eval<Expression>(exp.Arguments);

    var old = env;
    env = closure.Environment;
    foreach (var pair in closure.Lambda
        .Parameters.Zip(runtime, (p, arg) => new { p, arg }))
    {
        env[pair.x] = pair.y;
    }

    closure.Lambda.Body.Accept(this);
    env = old;
}
{% endhighlight %}

There's one small caveat. Since our syntax classes can no longer recursively evaluate inner expressions, the implementation of the evaluation visitor must store results on a stack. This takes a little getting use to, but we can more or less factor out the overrides of the <code>Evaluate</code> method and rename them <code>Visit</code>. The full code for the visitor class is a little too long to post as a snippet; please check [here](https://github.com/gjdanis/Licsp/blob/master/Visitors/Evaluation/EvaluationVisitor.cs) for the complete implementation.

The huge bonus of using this design pattern is that we gain interchangeability. Look at the return types for <code>Visit</code> and <code>Accept</code>. They’re <code>void</code>! The <code>Accept</code> method is just there to notify the visitor that its type should be visited next. Because the methods are <code>void</code>, the AST classes can accept <i>any</i> implementation of <code>IExpressionVisitor</code>; it’s up to the visitor to track the “result state” it wants to expose to clients. In our evaluation visitor, we wanted to visit the tree to a <code>Value</code>. For a pretty print visitor, we may want to store the result of visiting each node as a string, and expose that to a client for printing. We can leave the syntax classes untouched, no matter what set of new operations we want to define on the tree. 

## Final thoughts
Oddly enough, the first implementation of <code>Evaluate</code> is sometimes called the [interpreter pattern](https://en.wikipedia.org/wiki/Interpreter_pattern). To me, interchangeability is what makes the visitor pattern attractive for writing an interpreter: we can provide as many tree traversal algorithms as we want without ever having to change the code in the AST classes. It's a much more scalable approach and more naturally allows more than one developer to work on the project. 

If you'd like to learn more about programming languages, I highly recommend the [programming languages](https://www.coursera.org/course/proglang) course offered online by Coursera. [This blog post](http://norvig.com/lispy.html) by Peter Norvig is also a great resource. 

Whew -- I hope you enjoyed reading! If you have any thoughts, questions, or think that I've made a mistake, please leave a comment below. Thanks!

[^fn-switch_compile]: Switch statements in C/C++ are often rewritten to use [binary search](http://tinyurl.com/q3csobu)
[^fn-s_expressions]: More on the virtues of this [syntax](https://en.wikipedia.org/wiki/S-expression)