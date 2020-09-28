---
layout: post
title: Code generation in Java.
permalink: 2011/08/17/code-generation-in-java.html
categories:
- blog
---
An interesting problem appeared recently at my work, for some time we've been using various convenience methods - statics methods written in one class - a typical utility class.  When switching projects some of those methods were dearly missed so they were refactored to a more common library so taht we could use them in a new project. 

Unfortunately, the new project must use java 1.4 - and that meant no varargs which were used extensively in those connivance methods.

The interesting part was writing a code generator which would take some method description and create multiple overloaded methods of a given up to a given arity so that method like:

{% highlight java %}
public static void <T> println(T... tees) {
	for (T t : tees) {
		println(t);
	}
}
{% endhighlight %}

Would become:

{% highlight java %}
public static void println(Object o1) {
	System.out.println(o1)
}
public static void println(Object o1, Object o2) {
	System.out.println(o1);
	System.out.println(o2);
}
// ...
{% endhighlight %}

And so on and so forth. My first idea was to parse the original method and translate it to the overloaded methods - and that would be generally the best approach... but I didn't want to muck about longer with this generator then I'd spend hacking a few macros in vim and creating those methods by hand.

So I thought of a quick and dirty approach. First, I started writing high level code I would want to write without thinking (much) about the internals. I started with concept code for generating a single method, knowing that all else would fall into place after that.

I wanted to describe methods in a java-dslish like syntax such as:

{% highlight java %}
Method("println").Visibility(VISIBILITY.PUBLIC).
	Static().ArgumentTypes(Types.OBJECT).Returns(Returns.VOID).DefineBody(
		Println(ARG)
	).Arity(10)
{% endhighlight %}

Disclaimer: I won't present working the code here - just the reasoning I used to came to my results.

The methods would have the C++ style naming so that I could avoid clashing with reserved java words (I can define a method named ``If'', ``Static'', etc).  The proof of concept code was a little bizarre but definitely readable.

To support that kind of syntax I would have to use method chaining (a la Builders):

{% highlight java %}
public Method Static() {
        isStatic = true;
        return this;
}
public Method ArgumentTypes(String argumentType) {
        this.argumentType = argumentType;
        return this;
}
{% endhighlight %}

Nothing really interesting here - only setters that return a reference to `this` to support chaining. I defined the `DefineBody` method as a vararg method which took `Statement` s.

`Statment` was defined as a abstract class with an abstract `toString` method such as

{% highlight java %}
public static class Println extends Statment {
	private final String arg;

	public Println(String arg) {
		this.arg = arg;
	}

	@Override
	public toString() {
		return "System.out.println(" + arg + ");\n";
	}
}
{% endhighlight %}

One additional trick was to define static methods with the same name as the
class - an stolen from the ease of use of companion objects in scala: 

{% highlight java %}
public static Println Println(String arg) {
	return new Println(arg)
}
{% endhighlight %}

Such method calls were definitely easier on the eyes then writing ``new'' all
the time.

The last hack was to define ARG as uncommon character - I took \u0001 - and the
code generator would substitute the \u0001 character with a real argument name.

So the full to String method in the Method class look something like this:

{% highlight java %}
StringBuilder buff = new StringBuilder();

for (int currentArity = 0; currentArity <= arity; currentArity++) {
	buff.append(visibility + " ");
	if (isStatic) {
		buff.append("static ");
	}

	buff.append(returnType);
	buff.append(" ");
	buff.append(methodName)
	buff.append("(");
	// print the parameters 

	for (int i = 0; i <= currentArity; i++) {
		buff.append(argumentType + " ");
		buff.append(parName + i);
		if (i != currentArity) {
			buff.append(", ");
		}
		buff.append(") {\n") {
		// method body goes here
		for (int i = 0; i < currentArity; i++) {
			String bodyStr = body.toString();
			bodyStr = bodyStr.replaceAll(ARG, parName + i);
			buff.append(bodyStr)
		}
	}
}
{% endhighlight %}

The parName field depends on the argumentType - it created method from the first letter of the argument type with numbers such as:

{% highlight java %}
public static void println(Object o0, Object o1, Object o2) 
// or
arePositive(int i0, int i2, int i3)
{% endhighlight %}

Having done this boilerplate jotting down the rest was a breeze, the final
generator code looked something like:

{% highlight java %}
Class("Convenience").AddMethod(
        Method("println").Visibility(VISIBILITY.PUBLIC).
                Static().ArgumentTypes(TYPES.OBJECT).DefineBody(
                        Println(ARG)
                ).ARITY(15)
).ADD_METHOD(
        Method("sum").
                Visibility(Visibility.PUBLIC).
                Static().ArgumentTypes("int").BODY(
                        DeclareInt("sum"),
                        Assign("sum", 0)
                        Add("sum", ARG)
                ).ARITY(4)
).
        // and so on
).Visibility(Visibility.PUBLIC).Package("some.package").write();
{% endhighlight %}

For each overloaded method that was generated by the `Body` 's class the `toString` method looked analyzed the statement string looking for the special ARG char - if it was found the method would "loop around" that statement each method parameter, substituting the ARG character for a real argument name so that the definition: 

{% highlight java %}
Method("sum").
        Visibility(Visibility.PUBLIC).
        Static().ArgumentTypes("int").Returns("int").DefineBody(
                DeclareInt("sum"),
                Assign("sum", 0),
                Assign("sum", Add("sum", ARG).toString),
                Return("sum")
        ).ARITY(4)
{% endhighlight %}
Produced this code:

{% highlight java %}
public static int sum(int i0) {
	int sum;
	sum = 0;
	sum = sum + i0; // the toString method looped here
	return sum;
}
public static int sum(int i0, int i1) {
	int sum;
	sum = 0;
	sum = sum + i0; // the toString method looped here
	sum = sum + i1; // the toString method looped here
	return sum;
}
//
// ...
//
public static int sum(int i0, int i1, int i2, int i3) {
	int sum;
	sum = 0;
	sum = sumk+ i0; // the toString method looped here
	sum = sum + i1; // the toString method looped here
	sum = sum + i2; // the toString method looped here
	sum = sum + i3; // the toString method looped here
	return sum;
}

{% endhighlight %}

Although very hackish, without defining an external dsl and using complex parsing rules with syntax trees, this code generation method proved to be somewhat elegant and pretty understandable. The only thing dirty about it is string substitution .

Although I was giving an abstract example - methods that I'd defined at work were nowhere near those that I'd shown here, the code I'd done at work was about 250 LOC (so slightly longer then this article), and didn't define anything more complex then the ideas I've presented here.

All in all, I'd recommend this approach to code generation for mostly throw away code (write once, and forget) or code written in very tight time constraints.  For bigger projects and complex code I'd go with some compiler-compiler or more complex parsing methods.
