---
layout: post
title:  "TDD and TPP applied to Prolog"
date:   2017-01-16 08:42:27 +0100
categories: jekyll update
---
Some months before, I have rediscovered [Prolog][prolog] with a more Craft point of view. When I learnt it at school, it was more introduced as a toy language for understanding logic programming rather than a industrial-ready
tool.

However, you can have a Craft approach to Prolog as you would do for other 
languages.

From this point, I assume you have a basic knowledge of Prolog and logic 
programming.

## Test framework
### Build your own TF!

As I re-entered the Prolog world with a Craft view, I decided to create my test framework. Yeah!.
In prolog, you can easily create an assertEquals:
{% highlight prolog %}
assertEquals(M, X, X) :- write('[OK] '), write(M),nl.
assertEquals(M, X, Y) :- X\==Y,
	write('[KO] '), write(M),nl,
	write("Expecting "),write(X),
	write(", got "),write(Y),nl,fail.

assertEquals("shouldBeEquals", 1, 1).
assertEquals("shouldBeEquals", 1, 2).
{% endhighlight %}

The first predicate indicates that if we can unified the second argument with
the third, everything is OK. The second predicate does the opposite. Simple and 
convenient. 

I'm using [swipl][swipl]. If you don't like my wonderful test framework ;), 
you can use the built-in one shipped with swipl

### Test framework in swi-pl
Examples are based on the [riddle of the wolf, the cabbage and the cabbage][wolf-goat-cabbage] 
that have to cross a river.

You start by enclosing your tests between two tags : 
{% highlight prolog %}
:- begin_tests(cabbage).

:- end_tests(cabbage).
{% endhighlight %}

Then, you can add your tests:
{% highlight prolog %}
:- begin_tests(cabbage).
test(should_allow_a_cabbage_and_a_wolf) :-
  valid([wolf, cabbage]).
:- end_tests(cabbage).
{% endhighlight %}

And a failing test:
{% highlight prolog %}
:- begin_tests(cabbage).
test(should_not_allow_a_cabbage_and_a_goat_together_insensitive_order, [fail]) :-
  valid([goat, cabbage]).
:- end_tests(cabbage).
{% endhighlight %}

You can then load and run your tests from the swipl program:
```
$ swipl
?- consult("cabbage.pl"), run_tests(cabbage).
% PL-Unit: cabbage ................. done
% All 17 tests passed
true.
```

For more information, check the [unit test page][swipl-unit-test].

## TDD
### Red - Green - Refactor

The basic first thing in TDD is the cycle "[Red-Green-Refactor][red-green-refactor]". 

First, we write a failing test:
{% highlight prolog %}
:- begin_tests(cabbage).
test(should_not_allow_a_cabbage_and_a_goat_together, [fail]) :- 
  valid([cabbage, goat]).
:- end_tests(cabbage).
{% endhighlight %}

Then, we need to make it pass as fast as we can. Here, he most simple 
solution is to introduce a constant:
{% highlight prolog %}
valid([cabbage, goat]) :- fail.
{% endhighlight %}

Then, we need to refactor. Here, we can't really have something better.
So let's add another failing test:
{% highlight prolog %}
:- begin_tests(cabbage).
test(should_not_allow_awold_and_a_goat, [fail]) :- 
  valid([goat, wolf]).
:- end_tests(cabbage).
{% endhighlight %}

And, we introduce another predicate :
{% highlight prolog %}
valid([cabbage, goat]) :- fail.
valid([goat, wolf]) :- fail.
{% endhighlight %}

Now, we can refactor. We should look for duplication. Here, we have goat
which is a member of both predicates:
{% highlight prolog %}
valid(T) :- member(goat, T), fail.
{% endhighlight %}

Yes, we did it.

### Transformation Priority Premise
[Transformation Priority Premise][tpp] is a programming approach defined by 
Robert C. Martin. The idea is to apply the simpler transformation to your code
to make the new test pass.

We can transpose this approach to prolog:
1. {} ==> simple predicate with constant
2. simple predicate ==> failing predicate (rule with fail)
3. predicate ==> anonymous variable
4. predicate ==> simple unifiable variable (equals(X, X))
5. predicate ==> recursive unifiable variable (equals(X, [X | T]))
6. variable  ==> variable with domain (ins/in)
7. predicate ==> rule (with a body)
8. predicate ==> comparaison
9. predicate ==> recursion
10. predicate ==> cut
11. predicate ==> operation (X is X+1)

This is just a proposition. I'm looking forward for your comments :).

[prolog]:             https://en.wikipedia.org/wiki/Prolog
[swipl]:              http://www.swi-prolog.org/
[swipl-unit-test]:    http://www.swi-prolog.org/pldoc/doc_for?object=section(%27packages/plunit.html%27)
[red-green-refactor]: http://blog.cleancoder.com/uncle-bob/2014/12/17/TheCyclesOfTDD.html
[wolf-goat-cabbage]:  http://britton.disted.camosun.bc.ca/jbwolfgoat.htm
[tpp]:                https://8thlight.com/blog/uncle-bob/2013/05/27/TheTransformationPriorityPremise.html