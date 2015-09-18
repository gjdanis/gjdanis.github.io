---
layout: post
title: Writing a simple LISP interpreter in C#
unpublished: true
---

A few months ago, I decided to develop an interpreter for a subset of the LISP programming language. This was a great exercise, and I’d like to share a few things I learned along the way. 

### Full code and where we’re going

By the end of this blog, you'll know how to implement a simple interpreter. And your code will be able to interpret code like this:

{% highlight racket %}
> (define fib (lambda (n) // Fibonacci!
    (if (< n 2) 
      1 
  (+ (fib (- n 1)) (fib (- n 2))))))

> (fib 10)
55
{% endhighlight %}

For reference, the final project can be found [here](https://github.com/gjdanis/Licsp). 

## Background
An interpreter is a program that reads raw text in one language and translates that text into executable instructions in another language. It does this by constructing an [abstract syntax tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree) ("AST" for short), or an object representation that captures the execution order of the raw text. The tree is then traversed by an evaluation algorithm and the result is returned. 

This is different from, although similar to, the process used by a compiler to execute code. A compiler, however, will optimize your code.[^fn-switch_compile] Because of this optimization step, a representation of the compiled code is usually saved to the disk for later use. In contrast, interpreters have no need to do this and interpret the input text in memory. 

It should be noted that compilers and interpreters are not a feature of the language but rather a feature of the language implementation.[^fn-implementation_feature] One could, for example, implement the C programming language with an interpreter, and the Python programming language with a compiler. 

## Parsing

Unlike infix notation (operators appear <i>in-between</i> operands), LISP is a relatively easy language to parse. Since everything is fully parenthesized, we don't need to pay attention to operator precedence and the order of operations while parsing. Rather, we're more or less given the AST and it's up to us to translate the tree into a set of syntax classes.

When parsing the text, we have two types to consider; atomics, (indivisible values like numbers and bools), and lists which contain multiple expressions. If we see an atomic, there’s nothing to do — we simply return an object that holds the atomic data (the raw string). If we see an opening parenthesis, we know that we’re populating a list. We should continue parsing from the next token in the input stream, and stop once we see a closing parenthesis. At that point, we can return the list. 

In my implementation, I defined two types, <code>SExpAtom</code> and <code>SExpList</code>, both of which derive from <code>SExp</code>, an abstract class that is used to group the two child classes. I then apply the recursive algorithm described above to the input text. 

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
    var token = tokens.Pop();
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

The preceding step produced a structure that is far easier to process than the input text. With a valid <code>SExp</code> tree in hand, we can proceed to the next step, translating the <code>SExp</code> tree to an abstract syntax tree.

The base type for all syntax classes in the language is <code>Expression</code>, which requires that subclasses provide an implementation for the <code>Evaluate</code> method. <code>Evaluate</code> produces a <code>Value</code>, a derived type of <code>Expression</code> that evaluates to itself. 

{%highlight csharp linenos %}
abstract class Expression
{
    public abstract Value(Dictionary<string, Expression> env)
}

abstract class Value : Expression 
{
    public Value(object underlying)
    {
        Underlying = underlying;
    }

    public override Value Evaluate(Dictionary<string, Expression> env)
    {
        return this;
    }

    // new design is generic
    // stores Underlying as T rather than object
    public readonly object Underlying; 
}
{% endhighlight %}

Don't worry about the argument of <code>Evaluate</code>. We'll discuss later, and try and work on removing its direct exposure to the syntax classes. 

Here are the <code>Value</code> types we’ll support

* <code>Bool(bool b)</code>
* <code>Number(double x)</code>

The other expressions are more complicated and involve one or more <code>Expression</code>

* <code>Call(Expression body, List &#60;Expression&#62; arguments)</code> represents a function call e.g. <code>(myfunc 1 2 3)</code>

* <code>Define(string name, Expression Expression)</code> represents a variable definition e.g. <code>(define name e1)</code>

* <code>If(Expression test, Expression trueBranch, Expression elseBranch)</code> represents conditional execution e.g. <code>(if (e1) e2 e3)</code>

* <code>Lambda(Expression body, List&#60;string&#62; parameters)</code> represents an anonymous function e.g. <code>(lambda (n) (* 2 n))</code>

* <code>ListFunction(string op, List&#60;Expression&#62; arguments)</code> represents a built-in operation on a list of arguments <code>e.g. (+ 1 2 3)</code> 

* <code>Variable(string name)</code> represents a variable reference to lookup the <code>Expression</code> stored at <code>name</code> e.g. <code>x</code>

Given an <code>SExp</code>, or goal will be to turn the <code>SExp</code> into one of the above types. If we’re given an <code>SExpAtom</code>, parse the data to a Value and return the result. If we’re given an <code>SExpList</code>, look at the first element of the list. If it’s an <code>SExpAtom</code>, it’s Value will parse to one of the syntax keywords, (i.e. <code>if”</code> or <code>“lambda”</code>) which will tell us how to proceed parsing the rest of the contents of the list. By default, we’ll return a <code>Call</code>. Notice that if the first element is not an <code>SExpAtom</code>, it is an <code>SExpList</code> (the expression is a function that returns a function that gets called). The full code for this portion of the parsing algorithm is presented below:

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
        case "-":
        case "*":
        case "/":
        case "<":
        case ">":
        case "=":
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

## Evaluation

Great! We have an AST that can get evaluated. 

I mentioned earlier that we'd save discussion of the <code>Dictionary&#60;string, Expression&#62; env</code> argument in the <code>Evaluate</code> method until discussion of the evaluation algorithm. This is simply the evaluation <i>environment</i>, the place <code>Define</code>expressions put their definitions. You can think of this dictionary as the "context" that an <code>Expression</code> in the tree has access to during evaluation as the override of the <code>Evaluate</code> method will use the current environment to resolve itself to a <code>Value</code>.

Evaluation is pretty straightforward in most cases. If an <code>Expression</code> contains sub-expressions, evaluate those to <code>Value</code> types first, and then perform the required operations. <code>If</code> statements evaluate <code>Test</code> to a <code>Bool</code> and branch as expected. <code>Define</code> statements put their definition in the current environment, and return null. <code>Variable</code> expressions conversely return the <code>Expression</code> located at <code>Name</code>.

The most complicated (and interesting!) evaluation case is that of functions and function calls. <code>Lambda</code> expressions evaluate to <code>Closure</code> expressions, a <code>Value</code> type we haven’t discussed yet. A <code>Closure</code> consists of a <code>Lambda</code> and a copy of the environment in which the function was defined. If you think about it, this is no different than the type of nesting that happens whenever we write functions in any other language: functions have their own local environment that exists separate from whatever top-level environment in which the function is defined. If we didn’t support this type, we’d have no way to store the variables that get defined in a function; a variable <code>n</code> that is defined at the global level and defined again inside a function should leave the outer <code>n</code> untouched.

{%highlight csharp linenos %}
public class Call : Expression
{
    public Call(Expression body, List<Expression> arguments)
    {
        Body = body; Arguments = arguments;
    }

    public Value Evaluate(Dictionary<string, Expression> env)
    {
        var clo = (Closure)Exp.Evaluate(outer);
        var inner = new Dictionary<string, Value>(clo.Env);

        foreach (var p in clo.Lambda.Parameters
            .Zip(Arguments, (x, y) => Tuple.Create(x, y)))
            env[p.Item1] = p.Item2.Evaluate(outer);

        return clo.Lambda.Body.Evaluate(inner);
    }

    public readonly Expression Body;
    public readonly List<Expression> Arguments;
}
{%endhighlight %}

Evaluation of a <code>Call</code> thus evaluates its <code>Body</code> to a <code>Closure</code>, and extends the environment of the <code>Closure</code> with the evaluation-time arguments of the <code>Call</code>. The body of the <code>Lambda</code> in the <code>Closure</code>is then evaluated in the environment of the <code>Closure</code>. That’s it! A little complicated, yes, although there’s not a lot of code needed to implement correct evaluation of <code>Call</code> expressions. <code>Closures</code> seem conceptually different from any of the other Expressions we’ve discussed. But really, they're just a <code>Value</code> type; the <code>Value</code> that functions must evaluate to in order for the function's body to be lexically scoped.

## Improving the design

Our current design requires each syntax class know how to evaluate itself. While this approach is fairly OOP, it’s not particularly scalable. If we wanted to add any other set of operations, for example type inference or pretty printing, we'd have to change the definition of <code>Expression</code> and re-compile/retest all the AST classes. We also expose the evaluation environment to the syntax classes directly. If you think about it, the AST classes should really just hold expressions, and do nothing else. They don't really need to know anything about a environments and shouldn't be able to modify the evaluation context at will. 

What I’m getting at is that it would be better if we put the evaluation algorithm in separate function, and not have it distributed across the AST classes. In a functional programming language, the approach would be to switch on the <code>Expression</code> types using pattern matching. We can do that in C# by using the <code>is</code> keyword (correct but not ideal). 

Fortunately there’s a design pattern that addresses this problem exactly. It’s called the [visitor pattern](https://en.wikipedia.org/wiki/Visitor_pattern). There are visitors, which implement a <code>void Visit</code> method for all types in the tree, and visitables, classes that can accept an implementation of the visitor interface through implementations of the <code>void Accept</code> method. When a visitable accepts a visitor, all it does is call the visitor's <code>Visit</code> implementation for the visitable’s type. It’s now the visitor’s responsibility to choose what to do next. 

This design pattern is essentially a switch statement on the types of the visitables. It’s like we’re using the <code>is</code> keyword to determine how to proceed when given an <code>Expression</code>. The huge bonus of using this design pattern, however, is that we gain interchangeability. Look at the return types for <code>Visit</code> and <code>Accept</code>. They’re <code>void</code>! <code>Accept</code> is just there to notify the visitor that its type should be visited next. 

Because the methods are <code>void</code>, the AST classes can accept any visitor; it’s up to the visitor to track the “result” it wants to expose to clients through some internal state. In our evaluation visitor, we wanted to visit the tree to an <code>Value</code>; this is the result type that we’ll keep track of. For a pretty print visitor, we may want to store the result of visiting each node as a string, and expose that to a client for printing (see [here](https://github.com/gjdanis/Licsp/tree/master/Visitors/Formatting) for an example). For a type check visitor, we might want to expose a bool, to denote whether or not the tree is executable after visiting all the <code>Expressions</code> in the tree.

## Final implementation of the evaluation algorithm 

The final code puts the evaluation algorithm in an implementation of the <code>IExpressionVisitor</code> interface (too long for this post; accessible [here](https://github.com/gjdanis/Licsp/blob/master/Visitors/Evaluation/EvaluationVisitor.cs)). Our base type changes as follows

{%highlight csharp linenos %}
abstract class Expression
{
    public abstract void Accept(IExpressionVisitor v);
}

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

## Final thoughts

Oddly enough, the first implementation of <code>Evaluate</code> is sometimes called the [interpreter pattern](https://en.wikipedia.org/wiki/Interpreter_pattern). To me, interchangeability is what makes the visitor pattern attractive for writing an interpreter: we can provide as many tree traversal algorithms as we want without ever having to change the code in the AST classes.  

Whew -- I hope you enjoyed reading! If you have any thoughts, questions, or think that I've made a mistake, please leave a comment below. Thanks!

[^fn-switch_compile]: [Something You May Not Know About the Switch Statement in C/C++](http://tinyurl.com/q3csobu)
[^fn-implementation_feature]: [Programming Languages, University of Washington](http://tinyurl.com/px29okp)
