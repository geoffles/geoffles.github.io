---
layout: default
title: Type Injection Pattern?
categories: Development
tags: [java,c#]
date: 2018-01-18 9:26:00+0200
---

***Abstract:** A design pattern which allows you to "inject" the return type of an abstract method from a subclass in C++, C#, and Java*

# Introduction

Have you ever needed to create say a family of builders all with common methods on and also wanted to use a fluent style for the builders?

You may have found yourself doing things like defining protected "do" methods to perform the common functions on the abstract base class and then wrapping those on your concrete subclasses, like this:

{% highlight csharp %}
abstract class AbstractBuilder {
    protected void DoAddSomeStuff() {
        //...
    }
}

class FooBuilder : AbstractBuilder {
    public FooBuilder AddSomeStuff() {
        DoAddSomeStuff();
        return this;
    }
}
{% endhighlight %}

It would be super nice if the compiler could magically figure out that I want to return the subclass type... in C++ this is pretty trivial:

{% highlight cpp %}
template <class T, class R> class AbstractBuilder {
protected:
    T* instance;

public:
    AbstractBuilder(){
        instance = new T();
    };

    T* Build(){
        return instance;
    }

    R* DoSomething() {
        // ...
        return dynamic_cast<R*>(this);
    };

    virtual ~AbstractBuilder() {}
};

class FooBuilder : public AbstractBuilder<Foo,FooBuilder> {
    //...
}
{% endhighlight %}

Calling `DoSomething` `FooBuilder` will return a pointer to `FooBuilder`.

But C# and Java are a bit more strict when it comes to type casting... so how does one achieve this in those languages?

# C\#

Being a little more strict on type casting, the solution in C# is to provide a marker interface that will allow you to constrain your *builder* type to actually a Builder of the type you want to build.

This way your abstract base class implements the marker interface and constrains `R` to that type:

{% highlight csharp %}
interface IBuilder<T>
{
}

abstract class AbstractBuilder<T,R> : IBuilder<T> 
    where R: class, IBuilder<T> 
    where T: new() 
{
    protected T instance;

    public AbstractBuilder()
    {
        instance = new T();
    }

    public T Build()
    {
        return instance;
    }

    public R DoSomething()
    {
        return this as R;
    }
} 
{% endhighlight %}

This works because it allows you to specify that the type `R` is type compatible with `AbstractBuilder` without actually being the AbstractBuilder. The compiler wont allow you to static cast `this` up to `R`, but you can use the safe casting which will always work provided your implementing classes specify themselves as type `R`.


# Java

The same trick for C# also works in Java, although there are few Java-isms to to get the equivalent behaviour we have for our C#, specifcally providing the abstract method for getting the instance (although this isn't really an issue for the pattern).

{% highlight java %}
interface Builder<T> { }

static abstract class AbstractBuilder<T,R extends Builder<T>> implements Builder<T> {
    protected T instance;
    protected abstract T factory();

    public AbstractBuilder() {
        instance = factory();
    }

    public T build() {
        return instance;
    }

    public R doSomething(){
        @SuppressWarnings("unchecked")
        R result = (R)this;
        return result;
    }
}
{% endhighlight %}

# Bringing it All Together

I've provided complete working single file examples for C++, C# and Java below.

## C++

{% highlight cpp %}
#include <iostream>
#include <string>

using namespace std;

class ObjectBase {
public:
    string ObjectString;
};

class Foo : public ObjectBase {
public:
    string FooValue;
};

class Bar : public ObjectBase {
public:
    string BarValue;
};

template <class T, class R> class AbstractBuilder {
protected:
    T* instance;

public:
    AbstractBuilder(){
        instance = new T();
    };

    T* Build(){
        cout << "building" << endl;
        return instance;
    }

    R* DoSomething() {
        ObjectBase* base = dynamic_cast<ObjectBase*>(instance);

        if (base){
            base->ObjectString = "From AbstractBuilder";
        }

        cout << "Doing something from base" << endl;
        return dynamic_cast<R*>(this);
    };

    virtual ~AbstractBuilder() {}
};

class FooBuilder : public AbstractBuilder<Foo, FooBuilder> {
public:
    FooBuilder* FooSpecific() {
        instance->FooValue = "From FooBuilder";
        cout << "foo specific" << endl;
        return this;
    };

    virtual ~FooBuilder() {}
};

class BarBuilder : public AbstractBuilder<Bar, BarBuilder> {
public:
    BarBuilder* BarSpecific() {
        instance->BarValue = "From BarBuilder";
        cout << "bar specific" << endl;
        return this;
    };

    virtual ~BarBuilder() {};
};


int main()
{
    FooBuilder fooBuilder = FooBuilder();
    BarBuilder barBuilder = BarBuilder();

    Foo* foo = fooBuilder.DoSomething()->FooSpecific()->Build();
    Bar* bar = barBuilder.DoSomething()->BarSpecific()->Build();

    cout << foo->ObjectString << endl << bar->ObjectString << endl;
    cout << foo->FooValue << endl << bar->BarValue << endl;

    return 0;
}
{% endhighlight %}

## C\#

{% highlight csharp %}
using System;

namespace TypeInjectionDemo
{
	interface IBuilder<T>
	{
	}

	abstract class AbstractBuilder<T,R> : IBuilder<T> 
		where R: class, IBuilder<T> 
		where T: ObjectBase, new()
	{
		protected T instance;

		public AbstractBuilder()
		{
			instance = new T();
		}

		public T Build()
		{
			return instance;
		}

		public R DoSomething()
		{
			instance.ObjectString = "From AbstractBuilder";
			return this as R;
		}
	} 

	abstract class ObjectBase
	{
		public string ObjectString;
	}

	class Foo : ObjectBase
	{
		public string FooValue;
	}

	class FooBuilder : AbstractBuilder<Foo, FooBuilder>
	{
		public FooBuilder FooSpecific()
		{
			instance.FooValue = "From FooBuilder";
			return this;
		}
	}

	class Bar : ObjectBase
	{
		public string BarValue;
	}

	class BarBuilder : AbstractBuilder<Bar, BarBuilder>
	{
		public BarBuilder BarSpecific()
		{
			instance.BarValue = "From BarBuilder";
			return this;
		}
	}

	class Program
	{
		static void Main(string[] args)
		{
			var fooBuilder = new FooBuilder();
			var barBuilder = new BarBuilder();

			Foo foo = fooBuilder.DoSomething().FooSpecific().Build();
			Bar bar = barBuilder.DoSomething().BarSpecific().Build();

			Console.WriteLine(foo.ObjectString);
			Console.WriteLine(bar.ObjectString);

			Console.WriteLine(foo.FooValue);
			Console.WriteLine(bar.BarValue);
		}
	}
}
{% endhighlight %}

## Java

{% highlight java %}
public class Main {
    interface Builder<T> {
    }

    static abstract class AbstractBuilder<T extends ObjectBase,R extends Builder<T>> implements Builder<T> {
        protected T instance;
        protected abstract T factory();

        public AbstractBuilder() {
            instance = factory();
        }

        public T build() {
            return instance;
        }

        public R doSomething(){
            instance.objectString = "From AbstractBuilder";

            @SuppressWarnings("unchecked")
            R result = (R)this;
            return result;
        }
    }

    static class ObjectBase {
        public String objectString;
    }

    static class Foo extends ObjectBase 
    {
        public String fooValue;
    }

    static class FooBuilder extends AbstractBuilder<Foo, FooBuilder> {
        @Override
        protected Foo factory() {
            return new Foo();
        }

        public FooBuilder fooSpecific() {
            this.instance.fooValue = "From FooBuilder";
            return this;
        }
    }

    static class Bar extends ObjectBase
    {
        public String barValue;
    }

    static class BarBuilder extends AbstractBuilder<Bar, BarBuilder> {
        @Override
        protected Bar factory() {
            return new Bar();
        }

        public BarBuilder barSpecific() {
            instance.barValue = "From BarBuilder";
            return this;
        }
    }

    public static void main(String[] args) {
        FooBuilder fooBuilder = new FooBuilder();
        BarBuilder barBuilder = new BarBuilder();

        Foo foo = fooBuilder.doSomething().fooSpecific().build();
        Bar bar = barBuilder.doSomething().barSpecific().build();
        
        System.out.println(foo.objectString);
        System.out.println(bar.objectString);

        System.out.println(foo.fooValue);
        System.out.println(bar.barValue);
    }
}
{% endhighlight %}

# Conclusion

I'm not sure if this is a named pattern or not, but it definitely fits what I would consider the description of a design pattern.

The purpose of this design pattern is to be able to share common functions of fluent-like interfaces and might typically be applied to the builder pattern for example.

***Intent:** Allow a subclass to automatically override the return type of a method on its parent class*

The general class diagram for the pattern is:

![Pattern class diagram](/img/templatepattern/pattern.png)
***General:*** *Class diagram for the general case of the pattern.*
{: class="figure"}

And in the case of Java like languages:

![Pattern class diagram for Java like languages](/img/templatepattern/pattern2.png)
***Java/C\#:*** *Class diagram for the pattern in the case of java like languages.*
{: class="figure"}


