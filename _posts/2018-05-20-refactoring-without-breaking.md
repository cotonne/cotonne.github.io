---
layout: post
title:  "Refactoring code without breaking it"
date:   2018-05-20 08:42:27 +0100
categories: refactoring baby steps
---

"Oops, I did it again" [1]. This is what comes to your mind after entering "git reset --hard". 
You try to refactor this piece of code. In the beginning, it seems simple. But, hour after hour, 
you have started to understand that this part is coupled with this part, that some magical 
number has a strong business meaning, ... The refactoring of the code that should take only
one hour is not finished. You have other features to do and the code doesn't compile anymore.
But, was there any other way of not finishing in this situation?

This situation, you and I, we have met it often. In this article, we will see some technics that
we allow us to refactor the code without breaking it.

# Before we start

Refactoring a code means changing the internal structure without modifying the external behavior.
This is a really sensitive activity. You should not alter the external behavior as it will result
in bugs. You should also not end in a worse state than before. As a benefice, you can expect 
a code easier to understand and to maintain.

Before refactoring a code, you have to make sure that the code is correctly covert by some tests in
 order not to introduce regressions. Here, you can use metrics like code coverage or technics like 
mutation testing or golden master to ensure that the code is under control. Tests should be also 
fast to run. If not, we will be annoyed to run them and wait a long time before getting a feedback.

Once you have selected a piece of code to refactor, you should keep in mind that your objective
is to stay in the **green** zone. You don't want to break a test without knowing which modification
you did to reach this state. We will learn to do atomic modifications, the smallest one we can do.
This approach is close to the "Baby steps TDD".

# Extract variable, ... the easy part

As an example, I will use the code from the kata **Ugly Trivia**.  When you look at the code, 
you will notice this section:

```java
	private String currentCategory() {
		if (places[currentPlayer] == 0) return "Pop";
		if (places[currentPlayer] == 4) return "Pop";
		if (places[currentPlayer] == 8) return "Pop";
		if (places[currentPlayer] == 1) return "Science";
		if (places[currentPlayer] == 5) return "Science";
		if (places[currentPlayer] == 9) return "Science";
		if (places[currentPlayer] == 2) return "Sports";
		if (places[currentPlayer] == 6) return "Sports";
		if (places[currentPlayer] == 10) return "Sports";
		return "Rock";
	}
```

Quite ulgy, isn't it? Let's start by extracting variables. If you use a correct IDE, it is just a
shortcut to extract it. Because we rely on the IDE, we can have a good confidence that the code is
not broken. Anyways, let's run the tests.

# If, only If

Duplication can be easily spotted here. For example, the function returns "Pop" given the place is 
equals to 0, 4 or 8. **If** can be merged. An IDE will provide you a shortcut for it.

```java
	private String currentCategory() {
		int place = places[currentPlayer];
		if (place == 0 || place == 4 || place == 8) return "Pop";
		if (place == 1 || place == 5 || place == 9) return "Science";
		if (place == 2 || place == 6 || place == 10) return "Sports";
		return "Rock";
	}
```

Still in the **green** zone. You would say that it is easy to see what how to reduce the expression
into something more simple. However, we can introduce a regression without noticing and this is an
exercice, so let's stick to our objective to do atomic changes.

One approach can be to make the duplication explicit. What about that:

```java
    private String currentCategory() {
        int place = places[currentPlayer];
        if (place == (0 * 4 + 0) || place == (1 * 4 + 0) || place == (2 * 4 + 0)) return "Pop";
        if (place == (0 * 4 + 1) || place == (1 * 4 + 1) || place == (2 * 4 + 1)) return "Science";
        if (place == (0 * 4 + 2) || place == (1 * 4 + 2) || place == (2 * 4 + 2)) return "Sports";
        return "Rock";
    }
```

Yes, now it is obvious that the rest of the division indicates which categories the function 
returns. Again, this is an example of atomic modifications:

```
  if ((place == (0 * 4 + 0) || place == (1 * 4 + 0) || place == (2 * 4 + 0)) || (place % 4 == 0)) return "Pop";
  if ((place == (0 * 4 + 0) || place == (1 * 4 + 0)) || (place % 4 == 0)) return "Pop";
  if ((place == (0 * 4 + 0)) || (place % 4 == 0)) return "Pop";
  if (place % 4 == 0) return "Pop";
```

We introduce the new expression at the end. Then, we remove one condition at a time. So, if the new
condition is wrong, we will be able to spot it in no time.

We want to be sure that an atomic modification will not introduce regressions. Yes, you can 
argue that, here, this is obvious. But, in a more complex condition, how can you be as confident 
as now?

# Change type

Regarding the function **currentCategory**, a good thing is to replace the category as a String by
an enumeration. 

So, we can simplify the code
```java
    private String currentCategory() {
        int place = places[currentPlayer];
        int categoryIndex = place % Category.values().length;
        if (categoryIndex == 0) return Category.POP.toString();
        if (categoryIndex == 1) return Category.SCIENCE.toString();
        if (categoryIndex == 2) return Category.SPORT.toString();
        if (categoryIndex == 3) return Category.ROCK.toString();
        return Category.values()[categoryIndex].toString();
    }
```

We delete one **if** at a time. Which gives us :
```java
    private String currentCategory() {
        int place = places[currentPlayer];
        int categoryIndex = place % Category.values().length;
        return Category.values()[categoryIndex].toString();
    }
```

In order to be able to stay in the green zone and change the return type of the function, the 
simplest solution is to duplicate currentCategory, replace it and check every time that tests are
OK. At the end, we delete the unused function.

# Make it static

Regarding the class **Game**, we can easily understand that the function **currentCategory** doesn't
belong to this class. However, where does it belong?

For the moment, it is a little early for answering the question. But, we can do something more in
order to give us a signal for later: make the function static. If the function is static, it means
that it has nothing to do with this class.

After extracting the only class attribute as a parameter and renaming the function, 
we end with something like this:

```java
    private static Category selectCategory(int place) {
        int categoryIndex = place % Category.values().length;
        return Category.values()[categoryIndex];
    }
```

Given a place, we got the associated category. I think that it is a rule for defining the category for
a **board** (one of the hidden concept of the kata).

During this article, we use different techniques (introduce variable, make duplication explicit, ...)
in order to refactor the code without breaking it **at any time**. Even if it seems a lot of work, we 
have never been too far. At every modification, we were able to deliver in production a working 
software. This is a good improvement compared to a big bang refactor.

*[1] : Yes, I'm old... this is an 18-year old song...*


