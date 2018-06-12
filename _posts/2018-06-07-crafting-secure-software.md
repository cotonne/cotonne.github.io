---
layout: post
title:  "Crafting Secure Software"
date:   2018-06-07 08:42:27 +0100
categories: agile security craft secure-coding secure-by-design
---

A new day of work begins. Peacefully, you arrive at your work. You are the first to arrive today. A lot of people will arrive later due to the complicated deployment of the new version of the software. But, the whole team manages to do it. A big win!

You turn on your computer, open your mailbox. Suddenly, a lot of emails appears, all in red, indicated as importance. You start to read some of them: "Security alert". Without having the time to continue to look for all of them, Bob, your chief, arrives, really stress... well, more stress than usual. "We have an issue! Someone has found a weakness in the new system!". 
"What!?", you gasp, "That's not possible, we deployed it yesterday". 

"Well, someone has found that we can bypass the front and send a strange value to the backend. So, he was able to get negative money and be reimbursed. He has twitted about it and now, a lot of persons is exploiting this vulnerability! We need you to fix it ASAP!."

# Red alert! Red alert!

## Fixing the problem

In no time, you start to investigate the problem. How can a negative value proceed? The frontend checks that the value is between 0 and 10. You understand that they are directly sending their request to the backend. The business team is in charge of the test. They are only using the web interface. Well, tested... Mainly checking that no regression has been introduced... Due to the absence of tests, most of the time is spent on testing that important features still work.

You continue to investigate. Surprisingly, the value, which is a quantity, is enforced by the backend. If you send a negative value, the server responds with an "HTTP 400", indicating a bad request. But, the value received is a huge positive number. How can it become negative? 
 
You start to dive into the code. So messy! As an architect, you have been safe from writing code. 
Now, a bunch of contractors does the "bad work". The small software you left some years ago has become a monster. After some time, you find this piece of code:

```java
if ( quantity > 5 ) { 
    // New feature : one free item if you buy a lot of products!
    quantity = quantity + 1; 
}
```

You fix it. You run the tests. Gosh! They are either broken or ignored. There is almost no coverage of the code.

## What just happen?

The strange number received was: 2147483647. Why don't you had an exception? You are puzzled. 
Then, you start to remember your lessons at school. The number is an integer, interpreted as a set of bits with a limited size. When you add one, the sign is inverted: that's an **Integer Overflow**!

You comment this line of code. "In how many places do we do something like this?". After some research, you find a lot of similar things for calculating the price (you have discounts resulting in a negative value, ...). Your day is not over...

## Why?

What can be the cause of this situation? I would say that one of the main reason is linked to an organizational problem (lack of security knowledge and training, demotivated developers, ...).

Another reason is the messy code. The lack of consistency can be seen in the manipulation of quantity. Also, The cognitive charge to understand the code doesn't help to view the "big picture". The lack of tests goes in the way if you want to refactor the code.

A quote that states this situation is: *"You can't have quality without security."*

# Crafting secure software

Let's explore the relation between Security and Quality. You can have Quality by crafting software. Crafting here means applying quality-code technics like TDD, Clean Coe, Pair Programming, ... to get a well-design software which answers to all the requirements of your business.

Given the fact we want to maximize the security aspect for an amount of time, we should focus on the central/core/money generating/business sensitive part of your application. Hardening it will also prevent new vulnerabilities to appear when you change the rest. It does mean that the new technologies will come with vulnerability-free features ;).

## Business concerns

### Security helps Quality

Secure coding can be understood in two ways:
 - Following technical advice from guidelines (like the [Java Secure Coding Guidelines](http://www.oracle.com/technetwork/java/seccodeguide-139067.html))
 - Or using a style of programming where you develop in a way to prevent vulnerabilities. I will focus on this one which doesn't depend on the technology you are working on.

When you apply this style of programming, you will enforce conditions to prevent well-known attacks.

For example, you can have a function for defining the price. Normally, you will do like :

```java
int calculatePrice(int quantity, int pricePerProduct) {
  return quantity * price;
}
```

Applying a secure coding approach, you know that both quantity, price and final price can't be negative. So, you will enforce this property :

```java
int calculatePrice(int quantity, int pricePerProduct) {
  if (quantity < 0) throw new IllegalStateException();
  if (price < 0) throw new IllegalStateException();
  int finalPrice =  quantity * price;
  if (finalPrice < 0) throw new IllegalStateException();
  return finalPrice;
}
```

The style is close to defensive programming. The difference is that we look for corner and dangerous cases. Doing so, we tend to think also in a business way, asking about where is the limit. Here, we can ask: "Do you think 2 billion is a reasonable quantity? No? Can you define what is a normal quantity? Should we prevent or ask for special acknowledgment?". In this way, security helps us to find new business scenarios.

But, this is not perfect. Looking at the previous piece of code, we can spot a lot of duplication (test of a positive quantity). Moreover, we have to be sure that the logic is respected in every part of our software. Isn't there a way to do it?

### Crafting Quality Software brings Security

#### The importance of testing

To verify the rules, you can use two strong technics: Secure TDD and Property-based testing.

Secure TDD means that you will define unit tests which will check that your system can't be used in an illegal way. For example, you need to verify that a password is strong enough. So, you will add tests for testing rules (one letter, one number, ...). You will also look for strange passwords that should not be accepted (with dangerous characters or exotic characters). When defining your test cases, you should also consider extreme/special values (like 0, 1, -1,  Integer.MAX_VALUE, ...).

How, as a developer, can I know which values should be tested? We can't obviously test everything. A good technique is [equivalence partionning](https://en.wikipedia.org/wiki/Equivalence_partitioning). You define classes of values and you pick up a representative element from them. You can also use checklists like ["You are not done it"](http://www.thebraidytester.com/downloads/YouAreNotDoneYet.pdf) or like [this one](http://testobsessed.com/wp-content/uploads/2011/04/testheuristicscheatsheetv1.pdf). As you have mob programming, you have [Mob Testing](https://dojo.ministryoftesting.com/dojo/lessons/mob-testing-an-introduction-experience-report), a great exercise.

Property-based testing can also be used in order to secure your application. You can think of it as a fuzzer. It will generate random cases that you would not even think about.

Last but not least: you should look for a really correct coverage (if not 100%) at the core of your application. Look at code coverage and mutation testing. The first one highlights parts of your application which are not correctly tested while the other indicates weaknesses in your tests, which can lead to failure of **Fail Securely** principle.

#### Design your code to be secure

The best way to enforce security and improve the expressivity of your code is to 
introduce an immutable object (Price and Quality in this example).

The object needs to be immutable because you should not be able to create an instance which doesn't respect all the business rules. This way, rules are only in one place (**DRY**) and are checked every time you create a new object.

```java
@EqualsAndHashCode
public class Quantity {
        public final int quantity;
    
        public Quantity(int quantity) {
            if (quantity < 0) throw new IllegalStateException();
            this.quantity = quantity;
        }
}
```

For objects with a state, you should take care of auditability, the property that you should know with a state has evolved, why, who do it and so on. Building entity on events can be a good way for it.

#### Thinking in a mathematical way

A strong tool against mistakes in mathematics. 

Introducing algebra of types, function composition, ... and a lot of things coming
from Functional Programming (FP) can leverage the complexity of your system. For example, 
we can introduce a function **add** to the class Quantity:

```java
public class Quantity {
        ...
 
        public Quantity add(int i) {
            return new Quantity(quantity + i);
        }
}
```
We got : **add: Quantity -> Quantity**. I strongly advise having a look at FP and what it
can bring you in terms of designs (composition, monads, ...).

Types can serve as a "poka-yoke", a term from Lean which means a "keyed". Let's imagine the following functions:

```java
public void calculatePrice(int quantity, int price) {...}
```

What if you mix quantity and price? You will be unable to detect it except by closely testing all cases. A huge amount of time which could have been saved by relying on the compiler.

```java
public void calculatePrice(Quantity, Price price) { ... }
```

If you are good with mathematics, you can also include formal definitions in your code. Design by contract, formal provers, ... If you are able to prove some parts of your system, you can be bug-free for a long time! (given you don't make any mistake in your prove).

### KISS

KISS, a well-known principle from **Secure by Design** (the Economy of Mechanism as defined by Saltzer and Schroeder) can be found in the domain of crafting software.

You can detect bugs and vulnerabilities by reading the code. However, a complex and untangling code is harder to understand and so, it is harder to detect weaknesses during the code review. 

Keeping things simple will not only ease the addition of new features but also will make detections of holes in your applications easier and cheaper.

## Is it secure?

Is my application secure? It is something really hard to answer. New bugs, new weaknesses are discovered every day. Even if the core of your application is safe, what about the rest? Dependencies, architectures, ... should be also secured as well.

You can see this as a **Security in Depth** approach.

### Quick wins

You can look for quick wins. You can include a lot of tools in your CI/CD to detect common weaknesses. Tools like OWASP ZAP can detect Reflected XSS, missing HTTP headers, ...

According to [Sonatype](https://blog.sonatype.com/2014/11/42000-nexus-repository-managers-and-growing/),

*"Another interesting statistic: 20% of the requests to Central are coming from repository managers, meaning 80% are coming from another tooling directly"*. Which means that a huge amount of your attack surface can be reduced by constantly using up-to-date components. For example, Dependency Check (OWASP) can detect out-of-date libraries and warns you about it.

## Learning

Sun Tzu said: *"If you know the enemy and know yourself, you need not fear the result of a hundred battles. If you know yourself but not the enemy, for every victory gained you will also suffer a defeat. If you know neither the enemy nor yourself, you will succumb in every battle.”*

It is important that you know your opponent. You need to think like a hacker. Security training can give you a good understanding of common vulnerabilities (like the OWASP Top 10). You will be able to understand their "modus operandi", their way of thinking, planning and attacking.

# One last word

My point here is to show you that writing secure software is not harder than writing secure software. Like crafting software, it is a day-to-day work which needs energy and care. Security and crafting don't arrive by a mere accident but by a consciousness will of doing a good software. 

Security and Crafting are going well together. 

References
 - Secure by Design, https://www.manning.com/books/secure-by-design
 - Writing Secure Code, https://www.microsoftpressstore.com/store/writing-secure-code-9780735617223
 - ANSSI, a French administration in charge of security, has a lot of good papers about FP and security

