---
layout: post
title:  "A road to be a secure dev - From 0 to Kevin! 2nd Edition!"
date:   2018-11-21 08:42:27 +0100
categories: secure coding
---

In the [first post](https://cotonne.github.io/agile/testing/methodology/2018/11/10/secure-dev.html), I was talking about the steps to become a "secure dev".
When I started to develop, I asked myself: "Is there a way to write code without a bug?".
Later, when I discovered security and how it impacts development, I was like:
"Can I write code that no one can hack or exploit?".
Long story made short: No. From the initial idea to the final deployment, you can have so many ways
to introduce a weakness. But, this is not a problem.

With the infrastructure, we don't say any more: "No part of my system should fail" but
"eventually, something will fail, but I don't care, because I can recover".
We should think in terms of resilient systems, not unbreakable.
And when we develop, we should not think with: "No one will hack me" but "Someone will find one weakness, but I don't care because my system is designed to deal with it".

This article will explain different techniques that you can use to develop a resilient system for a developer. 
This article will focus on how to write a quality code.

# Level 0: Secure software is by definition quality software

This title is a quote from the book [The Security Development Lifecycle](https://blogs.msdn.microsoft.com/microsoft_press/2016/04/19/free-ebook-the-security-development-lifecycle/).

The first step to write a secure software is to write a software with quality inside.
The benefits of writing clean code are important: easier to read, understand, modify, test, ...
You will work on reducing complexity, improve the clarity of code, reduce duplication ...
You will try to minimize the amount of code.
And to do quality code, you need to use practices from the craftsmanship community: Clean Code, TDD, BDD, DDD, ...

This is a basic level. Writing quality code doesn't mean writing secure code.
However, it is a mandatory activity because it facilitates the use of other practices.

# Level 1: RTFM

First thing before using the new microwave you just bought: you read the manual and see if there
is no security issue [to dry your cat in it](https://www.snopes.com/fact-check/the-microwaved-pet/).
Same story when you write software: you start by reading security guidelines to see if you use it.
You can find a lot of security best practices which help you for writing a secure code.

For example:
 * [Tomcat](https://tomcat.apache.org/tomcat-7.0-doc/security-howto.html)
 * [MySQL](https://www.mysql.com/fr/why-mysql/presentations/mysql-security-best-practices/)

# Level 2: Secure Coding

Here is a good definition from [Wikipedia, _"Secure Coding"_](https://en.wikipedia.org/wiki/Secure_coding):

    Securing coding is the practice of developing computer software in a way that guards against the accidental introduction of security vulnerabilities.

We develop thinking about what could be wrong.

There are two main axes:
 * Some components can be internally "flawed" or dangerous enough to be forbidden. Just have the look to what [Oracle say about serialization](https://www.oracle.com/technetwork/java/seccodeguide-139067.html#8). Security guidelines will state what we can use and how we should such components. You have problems linked to your language.

Some examples of secure coding guidelines for Java:
 * [Secure Coding Guidelines for Java SE](https://www.oracle.com/technetwork/java/seccodeguide-139067.html)
 * [SECURE CODING GUIDELINES FOR JAVA SE, VERSION 5.0](https://wiki.sei.cmu.edu/confluence/display/java/SECURE+CODING+GUIDELINES+FOR+JAVA+SE%2C+VERSION+5.0)

 * Incorrectly trusted data: you are using data without thinking of the impact of them on the rest of the world. You received a string and send it directly to the database (which can lead to SQLi) or the browser (Reflected-XSS). Simply enough, you should treat such data with caution. Those problems are linked to the way you've coded.

You can find checklists that you can keep in mind when you develop:
 - [Top 10 Secure Coding Practices](https://wiki.sei.cmu.edu/confluence/display/seccode/Top+10+Secure+Coding+Practices)

# Level 3: Crafting Secure Software

We talk about how crafting quality software is necessary. Here, we will see how we can make those techniques evolved
to make code secured.

## Test-Driven Security

You can use TDD practices with security.

You start by writing a test for security.

For example, you want to assure that an URL can only be accessible by an authenticated user.
Let's say that we are developing a Spring application. So, you write a test:

```java
@RunWith(SpringRunner.class)
@SpringBootTest()
public class HelloControllerTest {
    @WithMockUser(username = "myUser", roles = {"READER"})
    @Test
    public void only_authenticated_user_with_role_READER_should_be_able_access_this_service() throws Exception {
        mvc.perform(get("/private/hello"))
                .andExpect(status().isOk());
    }

    @Test
    public void unauthenticated_user_should_not_be_able_access_this_service() throws Exception {
        mvc.perform(get("/private/hello"))
                .andExpect(status().isUnauthorized());
    }

    @WithMockUser(username = "myUser", roles = {})
    @Test
    public void authenticated_user_without_the_role_READER_should_not_be_able_access_this_service() throws Exception {
        mvc.perform(get("/private/hello"))
                .andExpect(status().isForbidden());
    }
}
```

You run the test and check that it fails. If not, your system is already secured and you don't know why!
Then, you make it pass with the minimum amount of code. Here, we want to reduce the risk of introducing
weaknesses by over-engineering our solution.
You can implement a simple solution:

```java
@Configuration
public class WebSecurityConfigurer extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/private/**")
                .hasRole("READER")
                .and()
                .httpBasic();
    }
}
```

Then, you refactor. For example, you can introduce a more generic solution for authenticated users.
And, with the test-driven approach, you should test corner cases.
So you need another test for the unauthenticated case.

The more specific your tests are, the more generic your solution is.
And you can stay assured that you have not broken any functionality.
Tests become your safety net.

You can find an [article](https://freecontent.manning.com/where-security-meets-devops-test-driven-security/) on this subject by Julien Vehent, author of "Securing DevOps".
Another reference is the [OWASP Secure TDD Project](https://www.owasp.org/index.php/OWASP_Secure_TDD_Project).

## Strongly Typed

Another thing that I really like: relying on the compiler to warn you in case of mistakes.
Let's imagine that you have a function **booking**:
```java
private static void booking(String userId, int quantity, String vhsId) {
    ...
}
```

You use it in your code with this:
```java
booking(vhsId, qty, userId);
```

Have you noticed the error? Yes, you have mix **vhsId** and **userId**.
We can add type to our function to prevent us from mixing this two parameters.

```java
private static void booking(UserId userId, int quantity, VhsId vhsId) {
    ...
}
```
In case of error, the compiler will generate an alarm. So, we **can't** do mistake thanks to our compiler.

No all developers like strongly-typed languages. ANSSI, the French Agency in charge of security of IT system, indicates:

    Utiliser un langage de programmation fortement typé, qui entre autres prévient les
    débordements de tableau, d’entier, ou l’utilisation de pointeurs invalides, est fortement
    souhaitable pour le développement des composants de confiance.

From [RECOMMANDATIONS POUR LA MISE EN PLACE DE CLOISONNEMENT SYSTÈME](https://www.ssi.gouv.fr/uploads/2017/12/guide_cloisonnement_systeme_anssi_pg_040_v1.pdf)

# Level 4: Secure by (DD-)Design

DDD aims to discover and understand a business domain in order to distill it in the code.
It proposes a set of patterns at code and software levels (aka Tactic and Strategic patterns).

For the first phase (the discovery phase), you look for clarifying terms used.
A user is not a customer and is not principal.
DDD highlights the fact that you should not try to over-reuse components from different models.
You need to be extremely specific.

Another great pattern is **Value Object**. They are designed to be immutable.
**State** is one of the pernicious kingdoms of software security errors.
With value objects, you try to minimize moving states.
Moreover, value objects are consistent objects. They represent an element from your domain.
When you create a new one, you always check that they respect the intrinsic properties,
meaning that you can't create value object in an invalid state.

More on this subject in the book ["Secure by Design"](https://www.manning.com/books/secure-by-design).

# Level 5: Secure by Design

Last but not least: use higher-level principles.
The checklist listed in part "Level 2: Securing coding" is inspired by the [Salzer & Schoeder design principles](https://en.wikipedia.org/wiki/Saltzer_and_Schroeder%27s_design_principles).
Those principles have not evolved so much since 1979. You can use it from day 1, from the first idea, during the design phase, development (see Secure Coding), ...

With those principles, you are able to make smart and safe decisions when you face an unknown problem.

Some other key practices I want to highly:
 - Threat Modeling. I advise you the book by Adam Shostack on this subject: [Threat Modeling: Designing for Security](https://www.amazon.fr/Threat-Modeling-Designing-Adam-Shostack/dp/1118809998/)
 - [Secure design patterns](Secure Software patterns)

# Conclusion

In this article, we have seen different levels of the mindset that we can use to produce better code.

This table will summarize which security error you can expect to address with every practice.
I will use the taxonomy defined by in the paper ["Seven Pernicious Kingdoms"](https://cwe.mitre.org/documents/sources/SevenPerniciousKingdoms.pdf).

| Kingdoms                            | Quality | RTFM | Secure Coding | CSS | Secure by (DD-)Design | Secure by Design |
|-------------------------------------|:-------:|:----:|:-------------:|:---:|:---------------------:|:----------------:|
| Input Validation and Representation | X       |      | X             | X   | X                     | X                |
| API Abuse                           | X       | X    | X             | X   |                       | X                |
| Security Features                   |         |      | X             | X   | X                     | X                |
| Time and State                      | X       |      |               | X   | X                     |                  |
| Errors                              | X       | X    | X             | X   | X                     | X                |
| Code Quality                        | X       |      |               | X   | X                     |                  |
| Encapsulation                       | X       |      |               | X   | X                     |                  |
| Environment                         |         | X    |               | X   |                       | X                |
