---
layout: post
title:  "Parcoursup, a long way to be graduated"
date:   2018-05-27 08:42:27 +0100
categories: parcoursup craft review quality
---

In France, the last step before going to advance studies is to define your wishes with an application called "Parcoursup". There, you wait for getting the position you are looking for. [According to the minister of Education](https://www.laprovence.com/actu/en-direct/4993400/parcoursup-65-des-eleves-ont-au-moins-un-oui-assure-le-ministre-de-leducation.html), around 65% got a proposition and only 20% of students say yes.

At the same time, the French government has published the [source code of Parcoursup](https://framagit.org/parcoursup/algorithmes-de-parcoursup). Some people have issued criticisms. In this article, I will review the code and provide a set of tips.

# A review

## Let's received the code

The code is hosted on Framagit and is released under the license GPL. It is supplied with a lot of information:
- The implementation of two algorithms: call order and propositions to send
- Some examples and documentation which help to clarify business choices
- A pom.xml
- Scripts for creating the database

I would say that I'm really surprised by the fact we have a better release than the [source code of tax calculation](https://github.com/GouvernementFR/calculette-impots-m-source-code) (it has been developed with a custom language: M, without any documentation and on a non-French website). Here, we can build and test the application. Even in the professional world, you can't expect to have something that easy. It has been developed with Java 8, a modern and common language. You can build it with Maven. 

However, you need an external dependency (Oracle JDBC driver) before.

## Let's travel around the code

When you open the project, the first thing you notice is the lack of **test** folder. Surprisingly, a lot of examples is available with a function **main**. You can easily move everything. Tests (and TDD) are still not widely used.

When I start or continue a project, I'm looking for different metrics in order to have a big picture of the project. Thanks to maven, you can do :
```
$ mvn sonar:sonar
```
I have added PI-Test in order to check the quality of the example. Here are the results
 - Duplication : < 1% (really good point)
- Code coverage: Global 80%, Core algorithms: 100%
- Pitest: KO (mainly due to timeout). 
Sonar has found 12 bugs (mainly try-with-resources), 14 vulnerabilities and 123 smells. 

A vulnerability is related to the access to the database. [For example](https://framagit.org/parcoursup/algorithmes-de-parcoursup/blob/master/java/parcoursup/propositions/donnees/ConnecteurDonneesPropositionsOracle.java#L137):
```java
        try (PreparedStatement ps = conn.prepareStatement(
                "INSERT INTO A_ADM_PROP "
                + "(G_CN_COD,G_TA_COD,I_RH_COD,C_GP_COD,G_TI_COD,C_GI_COD,NB_JRS)"
                + " VALUES (?,?,?,?,?,?," + GroupeInternat.nbJoursCampagne + ")")) {
```
I don't really understand why a parameter is added to this prepared statement.

Between the smells, we have 11 critical and 45 major problems. Critical smells are related to the bad naming of variables and the cognitive complexity of methods. Those methods are the one related to the loading of data and the algorithms.

Sonar indicates us that the code could have been improved. As a free and opensource software, Sonar should be considered as a default tool. It helps you to understand how "**sick**" your code feels.

## Let's find the craft

The project is well organized. It is separated by the two mains algorithms, each main package has a package for each concern (algorithm, data, ...). This is a good way for organizing your packages. We can easily ignore some parts at the beginning (like data reading or writing) and concentrate on the algorithms.

Looking at the function **equals**  (GroupeAffectationUID), I have the feeling that it has been written by hand. The [contract of equals](https://docs.oracle.com/javase/8/docs/api/java/lang/Object.html#equals-java.lang.Object-) is really important and it is hard to write a good **equals** by hand. You can ask your IDE to do it or use a library like **Lombok**.

I'm an IntelliJ user. It gives me more advises on how to improve the code. We should rely on our tools. When we are tired, we do mistakes.

When I read the classes [AlgoPropositions](https://framagit.org/parcoursup/algorithmes-de-parcoursup/blob/master/java/parcoursup/propositions/algo/AlgoPropositions.java) and [GroupeClassement](https://framagit.org/parcoursup/algorithmes-de-parcoursup/blob/master/java/parcoursup/ordreappel/algo/GroupeClassement.java), I found it really complicated to understand it, even with the huge amount of comments. You should know that comments should be considered as a smell. The code should be self-explanatory. If you need comments to make it readable, clearly you need to improve it.

You can notice that the code has been reviewed before it has been published. This is a good practice. Another even better practice is pair programming. With such technics, you can talk about the code and improve it. Other developers can help you to have a different vision and find technical or business mistakes. Again, a professional technic, not so widely used.

There is a function **log**. Sometimes, it is used, sometimes not. The problem is that you can't have an easy way to define levels of messages (error, warning, ...).

Final point : the complexity. A lot of classes (like **VerificationsResultats** or data access classes)
 have too much complexity. Thay can be improved with better use of patterns (GoF patterns, immutability,
 ...) or others approachs (functionnal, ...).

## Let's find the knowledge

The only source of knowledge is the documentation and the source code. We can notice that a documentation and example are available and versioned with the code. So, you can track changes to requirements and code together. The code tries to reuse as much as it can the words of the language. However, I have the feeling that some terms (**ordre d'appel**, **admission**) are mixed. I'm not a specialist in the domain so I can't really decide.

Sometimes, you have shorted named variables:
```java
    /* initialise la position d'admission à son maximum
    Bmax dans le document de référence */
    public void initialiserPositionAdmission() {

        /* on calcule le nombre de candidats éligibles à une admission
        dans l'internat aujourd'hui, stocké dans la variable assietteAdmission.
        On colle aux notations du document de référence */
        int M = candidatsEnAttente.size() + candidatsAffectes.size();
        int L = capacite;
        int t = nbJoursCampagne;
        int p = pourcentageOuverture;
```

One of the problems is the lack of expressivity of the code (like in the algorithms classes), making hard to link requirements to the code. Moreover, I have not found the requirements really easy to read. I need to implement it myself in order to understand it.

You can improve the situation by writing unit tests which are related to the rules. Another technique is BDD. You can explore and understand your domain by illustrating it with examples. 

You can find a good example there :
<blockquote class="twitter-tweet" data-lang="fr"><p lang="fr" dir="ltr">Intéressé par les algos de <a href="https://twitter.com/hashtag/ParcoursSup?src=hash&amp;ref_src=twsrc%5Etfw">#ParcoursSup</a> ? Vous pouvez retrouver les tests Cucumber de ceux de la section 4 sur mon  GitHub. 400 tests pour comprendre et valider l&#39;implémentation publiée sur Framagit <a href="https://t.co/wVV81sT7k0">https://t.co/wVV81sT7k0</a> La section 5 va suivre ! <a href="https://twitter.com/hashtag/TDD?src=hash&amp;ref_src=twsrc%5Etfw">#TDD</a> <a href="https://twitter.com/hashtag/BDD?src=hash&amp;ref_src=twsrc%5Etfw">#BDD</a></p>&mdash; José Paumard (@JosePaumard) <a href="https://twitter.com/JosePaumard/status/1000988954584387584?ref_src=twsrc%5Etfw">28 mai 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

One last tool for better domain distillation in your code: property-based testing. Some of the examples are implemented with some randomness. The documentation clearly states some properties (about the rank, ...). Finding good properties can be hard. You can use a library like [junit-quickcheck](https://github.com/pholser/junit-quickcheck/) to do it.

# Final thoughts

Considering this review, I would say that the project has good points (documentation, organization, naming, packaging, review, ...) and bad points (lack of tests, no living documentation, most important methods are not readable, ...).

According to my experience, I would say that it is what we use to find in a lot of companies. It could be better but I'm sure that a lot of developers doesn't reach the quality of this code.

In order to improve its maintainability, some practices could be applied:
 - TDD
 - Use of Sonar (to track and control technical debt)
 - Simple Design (to maximize clarity and minimize complexity)
 - "object calisthenics" constraints (to improve expressivity)



