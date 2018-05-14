---
layout: post
title:  "Writing better BDD scenario"
date:   2018-05-13 08:42:27 +0100
categories: agile testing methodology craftconf bdd
---

From May 8 to May 11, I have been at the CraftConf in Budapest for the third time and that was a really good experience.

Among all presentations, [Seb Rose](https://twitter.com/sebrose) and [Gáspár Nagy](https://twitter.com/gasparnagy) gave a great talk named **"Writing Better BDD scenario"**. I highly recommend their book "BDD Books: Discovery - Explore behaviour using Examples" in case you look for a reference on the subject. The talk started with the analysis of a badly written BDD scenario and followed by an exchange on what make a scenario bad.

First of all, they defined what is BDD by a good question:
" should we say :

    Behaviour Driven Development

or:

    Behavior Driven Development

You might know that Dan North coined as the one who defined "BDD". Born in UK, he used the word **"Behaviour"** with a U.

A brief definition of BDD: "BDD is about **understanding** & **validating** business requirements through illustrative examples".

<blockquote class="twitter-tweet" data-lang="fr"><p lang="en" dir="ltr">Yes, &quot;BDD is about *understanding* &amp; *validating* business requirements through illustrative examples&quot; Period <a href="https://twitter.com/sebrose?ref_src=twsrc%5Etfw">@sebrose</a> &amp; <a href="https://twitter.com/gasparnagy?ref_src=twsrc%5Etfw">@gasparnagy</a>  at <a href="https://twitter.com/hashtag/Craftconf?src=hash&amp;ref_src=twsrc%5Etfw">#Craftconf</a></p>&mdash; PHELIZOT Yvan (@yoda044) <a href="https://twitter.com/yoda044/status/994857018061131777?ref_src=twsrc%5Etfw">11 mai 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>


We had an interesting debrief about the smells in BDD. You can find smells, best practices and advices in the next table.

| Smells                                     | Best practices                                        | Advices/Examples/...                                             |
|--------------------------------------------|-------------------------------------------------------|------------------------------------------------------------------|
| Some parts contain Shall/Must/AssertEquals | Use Should                                            | The word "Should" might be used to keep the discussion opened. It can be, at least, used in **Then** part. |
| Different concerns are mixed (UI-oriented & DB, ...) |  One concern per scenario                   | Actually, you can have the UI scenarios in your UI domain without reference to DB. So you should probably do the same with business scenario. |
| Use of text from IHM in order to validate the condition | Don't rely on moving parts               | Since the text can change so your scenario can be brittle.       |
| More than 5 steps                          | Limit to 5 steps max (Given, When, Then)              | Beyond that, people won't read them.                             |
| Imperative style (Orders like "Click on this button") | Declarative style ("Buy a pizza")          | Orders may change. For example, instead of clicking on a button for fulffilling the request, you may need to tick a checkbox. You should write scenario with a declarative style, using business words (think of "buy a pizza" for a pizza-store application) |
| "I" (**Then** I should ...)                | Personnas                                             | Avoid the presence of emotions in scenario                       |
| Unclear/misleading name of scenarios       | Correct name for a scenario                           |                                                                  |
| No concrete example                        | Precise example                                       | **When I change the address** : where is "address"? not concrete |
| Imprecise and indecisive language          | Ubiquituous language                                  | **Then I should received a confirmation** : what was confirmed?  | 
| Untestable condition                       | Testable condition                                    | **Then I should received a notification** : how to test?         |
| Too many **then**                          | One **Then**                                          | one failed condition can break the entire test and it is not clear why it breaks |

In order to improve your scenarios, you can use orientation questions :
- Is it clear what each scenario is describing?
- Are there too many/too few details?
- Does it use a language that everyone understand?
- How likely is it that this scenario will change when the application changes?

They also talked about the **"example mapping"** as a way to find new scenarios.

This session was really interesting and helped me to have a better understanding of how to build BDD scenarios and which smells I should look for.


