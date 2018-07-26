---
layout: post
title:  "TDD and FizzBuzz"
date:   2018-07-25 08:42:27 +0100
categories: tdd testing methodology
---

FizzBuzz is a really interesting kata. Somewhat simple, this kata hides a lot of simple insights
for newcomers in TDD. He took me a lot of times to explicit implicit actions when I'm doing this
kata.

This article is an example of discussion I could have about TDD and FizzBuzz. Let's say **Y** is me and **Z** is my partner.

# The rules

They are quite simple. For a list from 1 to 100, print :
 * When the number is a multiple of 3, you say "Fizz"
 * When the number is a multiple of 5, you say "Buzz"
 * For the last rule, I can say: "when the number is a multiple of 3 and 5" or "when the number is a multiple of 15", you say "FizzBuzz". 

 - **Y**: "Ok, Z, I let you develop FizzBuzz."
 - **Z**: "Do you want me to do it with TDD?"
 - **Y**: "Well, do it in the way you want."

Beginners rush to write multiple ifs with the modulo function. In the end, they notice that one case is missing.

 - **Z**: "But, what if it is not a multiple?". 
 - **Y**: "You say the number."

Some people get stuck with the loop (1 to 100). When they are done, I ask some questions:
 * How can you prove to me that your code work? Some people write a main function and say: "Look at the console". 
 * What is your opinion about your code? Do you need more time to improve it? Most of the time, they say "it's ok"
 * What if I introduce a new constraint: a multiple of 7 gives "Quack"? Then, they realize that their design will oblige them to add four more ifs

 - **Z**: "Well, it starts to be complicated. I don't know how to improve it."
 - **Y**: "Let's delete the code and restart from the beginning."

# Red/Green/...

As a first step, I introduce TDD as a cycle: RED then GREEN then REFACTOR.
 * RED: you write a failing test
 * GREEN: you make it pass in the most simple way
 * REFACTOR: my definition is "you change the inside without changing the outside". Outside and inside are only a boundary you define

First question: what will be your first test? Why not Fizz? Or Buzz? Or FizzBuzz? I exclude the loop because it is too complicated to begin with it.

Before deciding, we have a list of simple tests cases we can imagine, so we should list them.

```java
public class FizzBuzzTest {
  //  1 => "1"
  //  3 => "Fizz"
  //  5 => "Buzz"
  // 15 => "FizzBuzz"
}
```

It is true that you can start with the test you want. However, I would advise:
 * Start with the most simple one
 * Start with the most logical one
 * Start with the most valuable one

The most simple, logical and valuable test is "Given a number, say the number".
 * Simple because you will just print it
 * Logical because 1 is the first value from the range
 * Valuable because with this case, your code works more than 50% of the time. If you ship it in production, you can start to earn some money ;)

When you write a test, you should start by the end. 

 - **Y**: "What value do you want?"
 - **Z**: "1". 
 - **Y**: "Ok. How do you get it?"
 - **Z**: "From the function **fizzBuzz**."
 - **Y**: "Ok. How do you get it from the function?"
 - **Z**: "I need to provide 1."
 - **Y**: "Great. You narrow the road from what you want to how you get it. The same way as you solve a maze when you were younger: by starting from the end. Here, you have the right to cheat ;)."


```java
public class FizzBuzzTest {
  @Test
  public void aTest() { // step 4 : test name
    // Arrange (step 3: what are my initial value?)
    int value = 1;
    // Act (step 2: how do I get it?)
    String result = fizzBuzz(value);
    // Assert (step 1: what do I want to test?)
    assertEquals("1", result);
  }
} 
```

 - **Y**: "At the moment, the name of the test doesn't really matter. As long as you have not finished writing your test, you are not sure of what you are going to test. So don't bother yourself with the naming. It can wait until the refactoring part. Given this test, we go a red light because the function **fizzBuzz** doesn't exist. Create it. Still red. Let's return an empty string. Still red. But now, **it is red for the good reason**. We make the test pass in the most simple way."
 - **Z**: "The most simple way? Sure?". 

It always sounds like "Isn't it stupid?" or "How can I do something simple?". 

 - **Y**: "Yes, simple. Why? Because we don't want overengineered solutions. We don't want a complicated code. Think of the **Ockham razor**."

```java
String fizzBuzz(int value) {
  return "1";
}
```

Let's run the test: **GREEN**!!! Yes, we can start the refactoring.
 - **Y**: "Do you want to write another test or do you want to refactor?"
 - **Z**: "I want to add a new test."

Most of the time, people want to add another test. I think it is because they want to get rid of the disturbing constant value. 

 - **Y**: "Well, don't go too fast. What about the name of the test? Maybe we can improve it a little bit."

```java
public class FizzBuzzTest {
  @Test
  public void should_say_the_number() { // step 4 : test name
    ...
  } 
```

Here are two good tips from a colleague:
 * No need for the context in the first test. You will only need it to differentiate others tests
 * Don't use return/if/... in your test names. It's only a duplication from your code. Dig into your domain to find meaningful terms

# Refactor!

 - **Y**: We can also do some refactoring to make the duplication visible:
```java
String fizzBuzz(int value) {
  return "" + 1;
}
```

 - **Z**: "But, where is the duplication?"
 - **Y**: "It is between your production code and your test: the number **1**. We can triangulate (write another test of the same kind) to have a more generic code."

```java
public class FizzBuzzTest {
  @Test
  public void should_say_the_number() { 
    assertEquals("1", fizzBuzz(1));
    assertEquals("2", fizzBuzz(2));
  }
} 
```

 - **Y**: "Make it pass in the most simple way"
 - **Z**: "OK"

```java
String fizzBuzz(int value) {
  return "" + value;
}
```

 - **Y**: "Do we need to keep both asserts?"
 - **Z**: "Well yes, it tests our code."
 - **Y**: "If I keep only one of this assert and I modify the production code, will it fail?
 - **Z**: "Yes, except if you get back to a constant value."
 - **Y**: "Obviously, it can happen as a developer can do anything to make it pass. But it is really unlikely. Moreover, if you want to keep it all, write a parameterized test. If not:
 * You will not know that the second assert is red when the first assert is red
 * It is unclear what you are testing"
 - **Z**: "So what next?"
 - **Y**: "We can do the case 'multiple of 3' for example."

```java
public class FizzBuzzTest {
  ...
  @Test
  public void should_say_Fizz_for_a_multiple_of_3() { 
    assertEquals("Fizz", fizzBuzz(3));
  }
} 
```

# The Fizz case

 - **Y**: "Make it pass, again, with the most simple way."
 - **Z**: "Errr, ok..."

```java
String fizzBuzz(int value) {
  if ( value == 3 )
    return "Fizz";
  return "" + value;
}
```

 - **Y**: "Ok, it works. Now, we need to triangulate."
 - **Z**: "But, we will introduce the modulo anyway. Do we really need to wait?"
 - **Y**: "Have you notice that there are two ways to solve it?"
 - **Z**: "Two ways?"

```java
String fizzBuzz(int value) {
  if ( value == 3 ||
       value == 6 ||
       value == 9 )
    return "Fizz";
  return "" + value;
}
```

 - **Z**: "Do we need to continue? It will be endless?"
 - **Y**: "No, it is enough. Three strikes and we refactor. But before that, we need to make the duplication visible."

```java
String fizzBuzz(int value) {
  if ( value == (3 * 1 + 0) ||
       value == (3 * 2 + 0) ||
       value == (3 * 3 + 0) )
    return "Fizz";
  return "" + value;
}
```

 - **Y**: "Do you see the two ways? It is true that we can use modulo. But we can also use another property of integer (value/3) * 3 == value"
 - **Z**: "But why? With modulo, it works!"
 - **Y**: "Yes, but you were lucky. You just use your knowledge to solve it. However, you made an unconscious decision without evaluating all options and you could have missed a better solution."
 - **Z**: "Ok, so let's replace everything by modulo"
 - **Y**: "Don't go too fast! Baby steps... we don't want to break everything in one move. What if everything goes red? You need to be in control of your code."
 - **Z**: "How can I do it?"
 - **Y**: "Let me show you. First, you introduce the new condition:"

```java
String fizzBuzz(int value) {
  if ( value == (3 * 1 + 0) ||
       value == (3 * 2 + 0) ||
       value == (3 * 3 + 0) && (value % 3 == 0))
    return "Fizz";
  return "" + value;
}
```

 - **Y**: "Then, you remove a condition at a time. Try to make the smallest step possible and run your tests every time"

```java
String fizzBuzz(int value) {
  if (value % 3 == 0)
    return "Fizz";
  return "" + value;
}
```

# Duplication, duplication everywhere!

 - **Y**: "Let continue with Buzz and FizzBuzz. You don't need to do triangulation now you know what is the best solution."

```java
String fizzBuzz(int value) {
  if ( value % 15 == 0 )
    return "FizzBuzz";
  if ( value % 5 == 0 )
    return "Buzz";
  if ( value % 3 == 0 )
    return "Fizz";
  return "" + value;
}
```

Sometimes, people mix two conditions. With the tests, it prevents them from continuing with a wrong solution.

 - **Y**: "So, what next? What if we introduce 7 and Quack?"
 - **Z**: "Arg, we will need to add a lot of it."
 - **Y**: "Haven't you notice that you copy/paste the if part?"
 - **Z**: "Yes and?"
 - **Y**: "Copy/pasting means that there is a kind of duplication. We have 3 if, so maybe we have a **3 strikes**? We need to make it clear. So what if I introduce a +"

```java
String fizzBuzz(int value) {
  if ( value % 3 == 0 && value % 5 == 0 )
    return "Fizz" + "Buzz";
  if ( value % 5 == 0 )
    return "Buzz";
  if ( value % 3 == 0 )
    return "Fizz";
  return "" + value;
}
```

 - **Z**: "Oh, I get it."
 - **Y**: "Yes, we can introduce a variable we can concatenate. But, always, baby steps."
 - **Z**: "What can be the smallest step?"
 - **Y**: "What about that?"

```java
String fizzBuzz(int value) {
  String answer = "";
  if ( value % 3 == 0 && value % 5 == 0 )
    return "Fizz" + "Buzz";
  if ( value % 5 == 0 )
    return "Buzz";
  if ( value % 3 == 0 )
    return "Fizz";
  if( !"".equals(answer) ) 
    return answer;
  return "" + value;
}
```

 - **Z**: "Wait, you can't! You are adding new code without a test!"
 - **Y**: "Are you sure? What is the value of "".equals(answer)?"
 - **Z**: "Well, true. So we never go to the answer part."
 - **Y**: "Yes, this is dead code. I'm not adding a new behavior. Let's continue by inverting a condition and concatenate. Don't forget to run your tests every time!"

```java
String fizzBuzz(int value) {
  String answer = "";
  if ( value % 3 == 0 && value % 5 == 0 )
    return "Fizz" + "Buzz";
  if ( value % 3 == 0 )
    answer += "Fizz";
  if ( value % 5 == 0 )
    answer += "Buzz";
  if( !"".equals(answer) )
    return answer;
  return "" + value;
}
```

 - **Y**: "And, we can delete the first if."

```java
String fizzBuzz(int value) {
  String answer = "";
  if ( value % 3 == 0 )
    answer += "Fizz";
  if ( value % 5 == 0 )
    answer += "Buzz";
  if( !"".equals(answer) )
    return answer;
  return "" + value;
}
```

# Quack, quack does the duck...

 - **Y**: "So, we can do Quack now."

```java
public class FizzBuzzTest {
  ...
  @Test
  public void should_say_Quack_for_a_multiple_of_7() { 
    assertEquals("Quack", fizzBuzz(7));
  }
} 
```

 - **Y**: "We can use the same approach."

```java
String fizzBuzz(int value) {
  String answer = "";
  if ( value % 3 == 0 )
    answer += "Fizz";
  if ( value % 5 == 0 )
    answer += "Buzz";
  if ( value % 7 == 0 )
    answer += "Quack";
  if( !"".equals(answer) )
    return answer;
  return "" + value;
}
```

 - **Z**: "But, FizzQuack, BuzzQuack, and FizzBuzzQuack work!"
 - **Y**: "Yes. But, the business has given us examples. There was a hidden behavior: words should be concatenated for every multiple. And, we already have a test for it."
 - **Z**: "What is the maximum number of tests you should have?"
 - **Y**: "There is at least a minimal number of tests equal to the cyclomatic complexity. You can add more tests but you should not sacrifice clarity for them. Not too many, but enough tests."
 - **Z**: "So, we have finished!"
 - **Y**: "Not really! We still have three ifs. We can create a special function: **isMultiple**."

```java
String fizzBuzz(int value) {
  String answer = "";
  if (isMultiple(value, 3))
    answer += "Fizz";
  if (isMultiple(value, 5))
    answer += "Buzz";
  if (isMultiple(value, 7))
    answer += "Quack";
  if( !"".equals(answer) )
    return answer;
  return "" + value;
}
```

 - **Z**: "Done?"
 - **Y**: "Still not. As you can see, it seems that to a number, we have a word associated. We can you Pair or Map to represent it. Again, baby steps."

```java
String fizzBuzz(int value) {
  Map<Integer, String> multiples = new HashMap<>();
  multiples.put(7, "Quack");
  String answer = "";
  if (isMultiple(value, 3))
    answer += "Fizz";
  if (isMultiple(value, 5))
    answer += "Buzz";
  if (isMultiple(value, 7))
    answer += "Quack";
  if( !"".equals(answer) )
    return answer;
  return "" + value;
}
```

 - **Z**: "And now?"
 - **Y**: "Now? This:"

```java
String fizzBuzz(int value) {
  Map<Integer, String> multiples = new HashMap<>();
  multiples.put(7, "Quack");
  String answer = "";
  if (isMultiple(value, 3))
    answer += "Fizz";
  if (isMultiple(value, 5))
    answer += "Buzz";
  for(Map.Entry<Integer, String> e: multiples.entrySet()){
    if (isMultiple(value, 7))
      answer += "Quack";
  }
  if( !"".equals(answer) )
    return answer;
  return "" + value;
}
```

 - **Z**: "Well, yes..."
 - **Y**: "Eh, baby steps! My tests are still green. I have no stress. I can start to replace constants by their values.

```java
String fizzBuzz(int value) {
  Map<Integer, String> multiples = new HashMap<>();
  multiples.put(7, "Quack");
  String answer = "";
  if (isMultiple(value, 3))
    answer += "Fizz";
  if (isMultiple(value, 5))
    answer += "Buzz";
  for(Map.Entry<Integer, String> e: multiples.entrySet()){
    if (isMultiple(value, e.getKey()))
      answer += e.getValue();
  }
  if( !"".equals(answer) )
    return answer;
  return "" + value;
}
```

 - **Z**: "Oh!"
 - **Y**: "So, after some more baby steps, we got:"

```java
String fizzBuzz(int value) {
  Map<Integer, String> multiples = new HashMap<>();
  multiples.put(3, "Fizz");
  multiples.put(5, "Buzz");
  multiples.put(7, "Quack");
  String answer = "";
  for(Map.Entry<Integer, String> e: multiples.entrySet()){
    if (isMultiple(value, e.getKey()))
      answer += e.getValue();
  }
  if( !"".equals(answer) )
    return answer;
  return "" + value;
}
```

 - **Z**: "Woah!"

# You are not done yet...

 - **Y**: "Have you seen the for/if? It does mean we can do filter/map! The += is a hint for a collector/reducer."

```java
String fizzBuzz(int value) {
  String answer = multiples.entrySet().stream()
                .filter(e -> isMultiple(value, e.getKey()))
                .map(Map.Entry::getValue)
                .collect(Collectors.joining());
  if( !"".equals(answer) )
    return answer;
  return "" + value;
}
```

 - **Y**: "Now, it is done... really?

# Some takeaways

 * Your IDE should help you to refactor, not get in the way
 * Living documentation
 * Emergent design
 * Explore domains
 * Make implicit explicit
 * Baby steps
 * Three strikes and you refactor






