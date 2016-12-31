---
layout: post
title:  "Make Java great again!"
date:   2017-01-01 08:42:27 +0100
categories: java functional javaslang
---

[Java 8][java-8] arrived in March 2014 as a savior, bringing with him some cool stuffs from functional world like lambda-calculus,
Optional and Stream.
However, it has not been released with a "licence to develop". Code is not only still crappy, but it is functionaly 
crappy! I'm not sure it was a good thing.

In this article, I will show some examples I found and I will talk about how to improve them. Then, I will introduce some 
important notions like immutability. Eventually, I will talk about the next step and what is missing in Java 8 but you
can still found in external libraries like [javaslang][javaslang].

The code is taken from a kata called [refacto][refacto]. It a spring controller which needs some small refactorings ;) .

# Real-life code

## Optional

The [Optional][java-optional] monad introduces the meaning that something may or may not (None) be here. 
Common use cases are:
 - You look for a value by id which may not exist
 - Value can't not be provided (as in a no-schema language like Json)
 - When you want to use null

### Optional as the parameter of a function

Here is a snippet of code I discovered recently :

{% highlight java %}
private void notifyIfNewPartnerAccount(String newCustomer, Optional<String> existingCustomerOpt) { 
   ...
}
{% endhighlight %}

Even Intellij complains about it : 
[optional](/images/2017-01-01-intellij-complains.png)

With this pattern, you are indicating that the value can't exist. I would advice to not do it.
You can already express it with a vararg argument or a collection of objects.
Optional should be used as a result. Moreover, it will simplify your function because you don't need to
check if the value exists. This check leads us to the next usage of Optional.

### isPresent/ifPresent

Here again, another real life example:
{% highlight java %}
if (existingCustomerOpt.isPresent()) {
   ...
} else {
   ... 
}
{% endhighlight %}

First of all, this function doesn't respect the [Single Responsibility Principle][single-responsibility-principle]. 
Yes, read it: *"If the value is present, do the processing else, provide a default value".

You can refactor it in this way:
{% highlight java %}
existingCustomerOpt.ifPresent(x -> {
   ...
})
{% endhighlight %}

This works well if you don't have any default behavior to implement. Another way that I prefer is this one:
{% highlight java %}

existingCustomerOpt
    .map(this::notifyNewPartnerAccount) // Normal behavior
    .orElse(this::defaultBehavior) // Default behavior

private void notifyNewPartnerAccount(String newCustomer, String existingCustomer) { 
   ...
}
{% endhighlight %}

Now, we have two functions that we can easily test. 

#e Modify everything

When you have a collection of objects, you may want to apply some processing to each of them. You can do it 
this way: 

{% highlight java %}
customers.forEach(customer -> {
            LOGGER.debug("customer: {} ", customer);
            try {
         saveOne(customer);
        ReportHelper.indicateSuccess(report, customer.identifier);
    } catch (SaveException e) {
        LOGGER.error("Exception while saving {} into {}. Cause {}", customer.identifier, e.getStep().toString(), e.getMessage());
        ReportHelper.indicateFailure(report, customer.identifier, e.getStep(), e);
    }
});

...

public class ReportHelper {
    public static void indicateSuccess(Report report, String identifier) {
        report.ok.add(String.format("Successfully save {}", identifier));
    }

    public static void indicateFailure(Report report, String identifier, Step step, SaveException e) {
        report.ko.add(String.format("Failed to save {} at step {}", identifier, step));
    }
}
{% endhighlight %}

What annoys me here is that some variables are modified in a unobvious way with the ReportHelper class.

I would advice to separate the production of the report from the aggregation. With a collector, we can do something like this:

{% highlight java %}
Report report = customers.stream()
	.map(this::save)
	.collect(new ReportCollector());
{% endhighlight %}

Function saveOne returns a report (success of failure), then you collect it using a specially crafted collector.
{% highlight java %}
public class ReportCollector implements Collector<Result, Report, Report> {
    private final Report report = new Report();
    @Override
    public Supplier<Report> supplier() {
        return () -> report;
    }

    @Override
    public BiConsumer<Report, Result> accumulator() {
        return (report, result) -> {
            if(result instanceof Success) {
                report.ok.add(result.toString());
            } else {
                report.ko.add(result.toString());
            }
        };
    }

    @Override
    public BinaryOperator<Report> combiner() {
        return (a, b) -> {
            a.ok.addAll(b.ok);
            a.ko.addAll(b.ko);
            return a;};
    }

    @Override
    public Function<Report, Report> finisher() {
        return report -> report;
    }

    @Override
    public Set<Characteristics> characteristics() {...}
}
{% endhighlight %}

It is true that this is not completely immutable. However, the mutable state is contained in a single class.

But, Java is still missing some useful elements to do something better

### Not enough monoids

# What is missing in Java 8

If you are not used to others functional languages like Scala or Haskell, maybe you are unaware of some great
missing parts in Java 8.

What I miss in Java 8 is :
 - Useful monads like Either or Try
 - Pattern matching
 - Zip
 - For-comprehension

Examples are made with the javaslang library

If you are not familiar with functional programming, i recommend you the book [Functional programming in Scala][fpinscala].


## Useful missing monads

Even if Java 8 has introduced the optional monad, there are still some useful monads that are 
missing. However, Javaslang implements those monads.

### Either

[Either][either-javaslang] can have two types: Left(A) or Right(B), with A and B, two others types.
The difference with Optional, which can be Some(A) or None, Either carries a value
(A or B) in both cases.

More striclty, Either is a *disjoint union of two types*, a [coproduct][coproduct].


It is used to represent a success or an failure. Let's reuse our example. The function **save**
has a new return type : Either.

{% highlight java%}
    private Either<Failure, Success> save(Customer customer) {
        LOGGER.debug("customer: {} ", customer);
        try {
            saveOne(customer);
            return Either.right(new Success(customer.identifier));
        } catch (SaveException e) {
            LOGGER.error("Exception while saving {} into {}. Cause {}", customer.identifier, e.getStep().toString(), e.getMessage());
            return Either.left(new Failure(customer.identifier, e.getStep()));
        }
    }
{% endhighlight %}

We will talk about how the collector is modified in the pattern matching part.

### Try

[Try][try-javaslang] is another useful monad. Try is a subset of Either. It can return a success if not exception is thrown or a 
failure.

We can rewrite the function **save** using Try instead of the try-catch exception :
{% highlight java %}
    private Result save(Customer customer) {
        return Try.of(() -> saveOne(customer))
                .getOrElseGet(ex -> treateFailure(customer, ex));
    }

    private Result treateFailure(Customer customer, Throwable ex) {
        if(ex instanceof SaveException) {
            SaveException e = (SaveException)ex;
            LOGGER.error("Exception while saving {} into {}. Cause {}", customer.identifier, e.getStep().toString(), e.getMessage());
            return new Failure(customer.identifier,e.getStep());
        }
        return new Failure(customer.identifier, Step.UNKNOWN);
    }
{% endhighlight %} 

## Pattern matching

Pattern matching is a structure which can recognize type and values. It is a kind of super switch.
For example, with the monad **Either**, we have :
{% highlight java %}
Match(_either).of(
    Case(Left($()), value -> ...),
    Case(Right($()), x -> ...)
);
{% endhighlight  %}

Our collector should be more cleaner : 
{% highlight java %}
public class ReportCollector implements Collector<Either<Failure, Success>, Report, Report> {
    private final Report report = new Report();

    @Override
    public Supplier<Report> supplier() {
        return () -> report;
    }

    @Override
    public BiConsumer<Report, Either<Failure, Success>> accumulator() {
        return (report, eitherResult) ->
                Match(eitherResult).of(
                        Case(Right($(instanceOf(Success.class))), result -> report.ok.add(result.toString())),
                        Case(Left($(instanceOf(Failure.class))), result -> report.ko.add(result.toString()))
                );
    }
   ...
}
{% endhighlight %}

## For-comprehension

The last structure I will introduced is the for-comprehension. It aim is to replace chained calls to map functions.
For example, imagine that you want to retrieve an entity (a customer for example) from a repository. This entity 
may or may not exist. So, we get it as an optional. Then, you want to retrieve the last order. But, he could have not
made orders. So, we get it as an optional. Eventually, for each order, we want the delivery. Order can not be deliver.
So, again, a optional.

To reach the last part, we had to do something like this:
{% highlight java %}
Optional<Customer> customer = customerRepository.findByEmail(email);
Optional<Collection<Order>> orders = customer.map(c -> orderRepository.findByCustomerId(c.getId());
Optional<Order> order = orders.map(o -> Optional.of(o.size() > 0? o.get(0):null));
Optional<Date> date = order.map(o -> o.getDeliveryDate());
{% endhighlight %}

The for-comprehension helps us to simplify this structure. In Scala, you would get something like this:
{% highlight scala %}
for(
 customer <- customerRepository.findByEmail(email);
 orders <- orderRepository.findByCustomerId(customer.getId());
 order <- if(o.size() > 0) Some(o.get(0)) else None;
 date <- o.getDeliveryDate();
) yield date
{% endhighlight %}

Quite convenient, isn't it? You can have the same with javaslang.
{% highlight java %}
 For(customerRepository.findByEmail(email), customer ->
            For(orderRepository.findByCustomerId(customer.getId()), orders ->
                For(Optional.of(orders.size() > 0? orders.get(0):null), order ->
                    For(order.getDeliveryDate())
                            .yield()
                )
            )
        ).toOption();
{% endhighlight %}
Well, a little bit less "sexy" than the Scala version :'(.

I hope you get a glitch of what the functional world is made of. A lot of people complains about the lack
of functional elements in Java. If you don't know, have a look to Scala or Haskell and you will found 
something bigger inside them!


[java-8]: https://en.wikipedia.org/wiki/Java_version_history#Java_SE_8
[refacto]: https://github.com/cotonne/katas/tree/master/refacto
[java-optional]: https://docs.oracle.com/javase/8/docs/api/java/util/Optional.html
[single-responsibility-principle]: https://en.wikipedia.org/wiki/Single_responsibility_principle
[javaslang]: http://www.javaslang.io/
[fpinscala]: https://www.manning.com/books/functional-programming-in-scala
[coproduct]: http://blog.higher-order.com/blog/2014/03/19/monoid-morphisms-products-coproducts/
[either-javaslang]: http://www.javaslang.io/javaslang-docs/#_either
[try-javaslang]: http://www.javaslang.io/javaslang-docs/#_try
