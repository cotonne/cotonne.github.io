---
layout: post
title:  "To be true or not to be true, that is the question!"
date:   2018-07-01 08:42:27 +0100
categories: primitive obsession
---

Some weeks ago, I gave an interview for hiring a new architect. During an interview, we use to give a technical test which involves writing code. It is a good way to see if people know how to write some code, how they think, design and exchange with teammates.

In the model, you can have students with different properties. Some can have a scholarship (money given to the student for helping him). When we were talking, I asked: "Why does the variable SCHOLARSHIP have a type boolean instead of an enumeration?". An interesting debate followed. Her point was that, with only two states, you only need a boolean.

# True is ... errr... 

The first point I would raise is the fact that enumeration helps you make your code clearer. For example, let's say that the student has two states:**SCHOLARSHIP** and **GRADUATED**. 

With a boolean, building a new student will be : 
```java
val student = new Student(true, false);
```

The property **SCHOLARSHIP** is the first argument? The second? With an enumeration:
```java
val student = new Student(SCHOLARSHIP, NOT_GRADUATED);
```

Here, we can't do a mistake. The code is clearer. Also, true mean with or without a scholarship? Sometimes, you can mix the meaning behind the value true and false. One funny fact: she did this mistake during the interview.

A way to prevent it from happening is the use of a builder:
```java
val student = Student.withScholarship().withoutGraduated();
```

# Primitive obsession and YAGNI

The "enumeration over boolean" is a kind of primitive obsession. Why do you use a boolean to represent an information from the model when there are **two** states? Why don't you use an integer to represent another information when you have **three** states? 

IMHO, there is a kind of contradiction. To express a domain information, you use an enumeration. POINT. If you don't do it, you are falling into the trap of a **"Primitive Obsession"**.

One can argue: "This is over-engineering!" or "YAGNI". When you detect a meaningful domain element (which is the case here), you should use an enumeration. Moreover, here, a new state can't appear; You have a scholarship or not, you can't an "half-scholarship". We want to be expressive and prevent mistakes. We are not forecasting the future.