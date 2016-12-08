---
layout: post
title:  "Hexagonal architecture is dead… Long live to hexagonal architecture!"
date:   2016-12-08 08:42:27 +0100
categories: architecture craft
---


Some days ago, i did the exercise “Birthday Greetings” during an internship 
about TDD given by Mathieu P., a coach. In this exercise, you have to isolate 
your core domain from implementation details. When we were comparing each others 
our refactoring, I realized that the core shape of the application was heavily 
influenced by the first type of services you will request. The day after, 
I was talking with my customer about the reason of using the suffix Impl with 
a lot of classes in their codebase. Both of those experiences leads me to think 
that hexagonal architecture has not been well implemented since years.


If you use “Impl”, I think it is a good smell that you don’t have comply to the 
definition of hexagonal architecture. Solving the “Birthday Greetings” Kata will 
lead us to think about abstractions. Impl is just something wrong which should make 
you think about the fact that “Do you really need this interface?”

## Hexagonal architecture

This pattern is from Alistair Cockburn (http://alistair.cockburn.us/Hexagonal+architecture). 
The principle is that the heart of your application should not directly depend on the outside world. 
If something changes outside, the core should not be impacted. In order to achieve this protection, 
you need to create an abstraction layer which will adapt the outside world to your world.


This abstraction, called adapter, defines a contract that external services should follow. You will 
have to manipulate the service in order to comply to the contract.

![A basic hexagonal architecture](/images/2016-12-08-hexagonal-architecture-is-dead-1.png){:class="img-responsive"}


In this way, your core will not change due to a modification of the service. However, the question 
is how can you define the contract that in a way it will not have to be changed when the outside 
service evolves? 


## An example: “Birthday greetings” Kata

While doing this Kata during the internship, two points emerged. First, the abstraction can leak 
in your core. Second, your design is shaped by the way you receive your data.


The application is simple: you parse a file which contains employees and you send an e-mail to 
the ones whose their birthday is today. 


In order to refactor the application, you go to the main class and you look for a way to 
externalize dependencies.


For the repository part, you would probably do a refactor close to something like this:


{% highlight java %}
public void sendGreetings(XDate today) {
   Collection<Employee> employees = employeesRepository.getEmployees();
   Collection<Employee> employeesBornedToday = findEmployeesBornedToday(today, employees);
   sendGreetings(employeesBornedToday);
}
{% endhighlight %}

But, you can also reach to something like this:
{% highlight java %}
public void sendGreetings(XDate today) {
   Collection<Employee> employeesBornedToday = employeesRepository.findEmployeesBorned(today);
   sendGreetings(employeesBornedToday);
}
{% endhighlight %}


Do you notice the difference? In the first case, you know that you are manipulating a file, 
so you don’t apply a constraint to your repository. In the second case, you are ignoring 
this fact and you just define the contract in a way that it would fit well with your application core.


For the message part, you could also do this :
{% highlight java %}
private void sendGreetings(Employee employee) throws MessagingException {
   String recipient = employee.getEmail();
   String body = "Happy Birthday, dear %NAME%!".replace("%NAME%", employee.getFirstName());
   String subject = "Happy Birthday!";
   messageSender.sendMessage("sender@here.com", subject, body, recipient);
}
{% endhighlight %}

Or that:
{% highlight java %}
private void sendGreetings(Employee employee) {
   messageSender.sendMessage(employee);
}
{% endhighlight %}

Once again, you have decided that your service must accept a subject and a body. What will happen 
if you decide to sent an SMS? Moreover, exceptions like MessagingException can be indicated in the 
interface, so you need to catch them. Creating an interface will constrain you to do strange things 
in the implementation.


This example gives us a glitch of the process which will in the end make an interface emerges: 
from the inside to the outside. We saw that we should not let us influenced by the underlying 
implementation such as file-based repositories, messaging systems, ...


## Shaping the core
When you build a new application, you oftenly start from the information you have and you derive 
your application from them. You analyze the answer given by services and you define the way your 
application will work:

![A basic hexagonal architecture](/images/2016-12-08-hexagonal-architecture-is-dead-2.png){:class="img-responsive"}


In order to test your application, you would probably introduce interfaces and dependency injection :
![A basic hexagonal architecture](/images/2016-12-08-hexagonal-architecture-is-dead-3.png){:class="img-responsive"}


Well, the fact is that the actual context are driving the design. If we think about the kata, maybe we should tell what we want.
![A basic hexagonal architecture](/images/2016-12-08-hexagonal-architecture-is-dead-4.png){:class="img-responsive"}


We end up using the same data but, we have first thought on how we want it design.  It is like you bullying the outside world to accept your requirement instead of you, obeying it.


So what is the problem of having Impl in class names which implement the interface ?


## What “Impl” is the sign of and why is it wrong?

Impl comes from J2EE world. It was used when interface and implementation were along in the same package. Impl shows that you are applying this rule.


When we develop, most of the time, we start by developing the implementation. Then, we look for a way to introduce dependency injection. We create an interface which is the copy of the implementation functions. 


Some points need to be highlighted:
We start with the implementation instead of defining what we really need. 
Based on one example, we define a strong contract that will spread everywhere in our application and will constrain future services.  
A name has to be given to the  interface. So let’s called the interface “InterfaceName” and the implementation “InterfaceNameImpl”. 


On this point, let’s have a look on how an implementation is defined in Java. Given an interface called Toto, following the rule Impl, we get:
{% highlight java %}
public class TotoImpl implements Toto
{% endhighlight %}

Let’s read it: the class TotoImpl implements Toto. We clearly have a duplication here. It does not give you more information.


One told me that Impl is used to indicate that it is the only implementation in the system. I advise to use “Default”. 
{% highlight java %}
public class DefaultToto implements Toto
{% endhighlight %}


It can also be an implicit convention. Since old times, such classes were named with “Impl”. As in the fable 
[“The five monkeys and the ladder”][five-monkeys], 
we should remember that rules must exist for a reason and we must know why.


Whenever, I don’t find this information useful. Most of time, I’m more interested on the way the implementation is done:
{% highlight java %}
public class DatabaseToto implements Toto
{% endhighlight %}


The interface is created more because we are used. But do we really need it?


## Do you really need this interface?
Let me quote Robert C. Martin : *“As the tests get more specific, the code get's more generic”*.
And Martin Fowler: *“The three strikes and you refactor”*


Trying to do something generic without much more information is like us saying 
“Yes, let’s me do something generic which will answer all my future needs that i don’t know”.


Why do we need to create an interface so early in our development? 
 + Dependency injection? You can do dependency injection even without interface. 
You can extend or mock some part. Just provide the concrete class. as soon as you 
will have to do change, you can start to create an interface.
 + For testing purpose? In Java, you can use Mockito to mock a concrete class. 
When you will introduce the interface, you will just need to replace it in your test.


The main point here is to make you aware to carefully defined your interface. Just using 
dependency injection without protecting your application core for implementation details
avaoids benefits from hexagonal architecture.
Impl is an old convention. It is an habit which doesn’t make sense anymore. It comes from 
the need to create an hexagonal architecture. Just applying this convention without 
understand and applying the design process behind this pattern is just a “cargo cult”.
We have to think before coding, mainly on the constraint we are giving ourselves. 
When implementing the hexagonal architecture, we should better choose to design from 
the inside to the outside, delaying as much as we can constraints introduced by the adapters.


After all, do we really need this? :)

[five-monkeys]: http://www.wisdompills.com/2014/05/28/the-famous-social-experiment-5-monkeys-a-ladder/  


