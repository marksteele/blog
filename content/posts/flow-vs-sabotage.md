---
layout: blog
title: Flow vs. Sabotage
date: '2021-05-31T10:20:16-04:00'
cover: /images/sabotage.jpg
categories:
  - random
---
I've been thinking about lean in the context of software engineering and analyzing processes and communication patterns lately, and how everything fits together.

The goal: How to increase flow, or the velocity of work getting done. In any non-trivial organization, there are a ton of things to consider. The ones I've been focusing on are communication, processes, and ceremonies.

Observations about communication:

* Organizations which design systems are constrained to produce designs which are copies of the communication structures of these organizations. This is known as [Conway’s law](http://www.melconway.com/research/committees.html). Worded a bit differently: “Organizations which design processes are constrained to produce a process flow which is a copy of the communication structures between departments”
* Conway also gives us the idea homomorphic force: “This kind of structure-preserving relationship between two sets of things is called a homomorphism.” 
* Software architectures are difficult to change, as are organizations. We can reason that the two are mutually supporting. Once software takes on the characteristics of the wider organization it will serve to reinforce the organizational structures, which in turn will strengthen the software architecture.

As organizations grow from a single engineering team to multiple teams, there appears to be a natural tendency to want to start to apply local optimizations by creating specialist teams that focus on specific problem areas. This specialization usually emerges as a result of the larger organization needing to manage priorities coming from multiple sources (customer support, legal, finance, and so on). The reaction: create engineering teams that only focus on small problem areas in the hopes that increases in domain knowledge leads to less context switching and increases in delivery velocity.

As we explore moving away from this model and instead shifting the focus to a holistic feature driven (or customer centric) view of development, we’re feeling the discomfort and friction of the impedance mismatch between our traditional thinking/communication patterns and optimizing for end-to-end process efficiency. 



In 1944, the Office of Strategic Services published a field manual on how sympathizers could assist the war efforts by sabotaging organizations. This handy manual covered several different industries, and interestingly there is a ton of insight into how to make organizations fail and falter.

You can view the whole doc [here](https://www.gutenberg.org/files/26184/page-images/26184-images.pdf).

Here are some gems:

* Insist on doing everything through “channels.” Never permit short-cuts to be taken in order to expedite decisions.
* Make “speeches.” Talk as frequently as possible and at great length. Illustrate your “points” by long anecdotes and accounts of personal experiences.
* When possible, refer all matters to committees, for “further study and consideration.” Attempt to make the committee as large as possible — never less than five.
* Bring up irrelevant issues as frequently as possible.
* Haggle over precise wordings of communications, minutes, resolutions.
* Refer back to matters decided upon at the last meeting and attempt to re-open the question of the advisability of that decision.
* Advocate “caution.” Be “reasonable” and urge your fellow-conferees to be “reasonable” and avoid haste which might result in embarrassments or difficulties later on.
* In making work assignments, always sign out the unimportant jobs first. See that important jobs are assigned to inefficient workers.
* Insist on perfect work in relatively unimportant products; send back for refinishing those which have the least flaw.
* To lower morale and with it, production, be pleasant to inefficient workers; give them undeserved promotions.
* Hold conferences when there is more critical work to be done.
* Multiply the procedures and clearances involved in issuing instructions, pay checks, and so on. See that three people have to approve everything where one would do.
* Work slowly
* Contrive as many interruptions to your work as you can.
* Do your work poorly and blame it on bad tools, machinery, or equipment. Complain that these things are preventing you from doing your job right.
* Never pass on your skill and experience to a new or less skillful worker.

These are amazing markers and hints for identifying patterns of unintentional sabotage and a great list of things to be on the lookout for when trying to optimize organizations.
