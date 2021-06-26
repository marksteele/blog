---
layout: blog
title: Flow vs. Impedance vs. Sabotage
date: '2021-05-31T10:20:16-04:00'
cover: /images/sabotage.jpg
categories:
  - random
---
How to increase flow (or the velocity of work getting done) is the topic of today's ramblings. 

Flow is impacted by many different factors, but the ones I'd like to pontificate on are: communication, process, and culture.

Observations about communication and organizational structures:

* In 1968, [Conway observed](http://www.melconway.com/research/committees.html): "Organizations which design systems are constrained to produce designs which are copies of the communication structures of these organizations." This is known as Conway’s law. Worded a bit differently: Organizations which design processes are constrained to produce a process flow which is a copy of the communication structures between departments.
* Conway also gives us the idea homomorphic force: “This kind of structure-preserving relationship between two sets of things is called a homomorphism.” 
* Software architectures are difficult to change, and so are organizations. It's not a very large stretch to suppose that these two things are self-supporting. Once software takes on the characteristics of the wider organization it will serve to reinforce the organizational structures, which in turn will strengthen the software architecture.

As organizations grow from a single engineering team to multiple teams, there appears to be a natural tendency to want to start to apply local optimizations by creating specialist teams that focus on specific problem areas. This specialization usually emerges as a result of the larger organization needing to manage priorities coming from multiple sources (customer support, legal, finance, and so on). Organizational lines are drawn, and engineering team structure often follows.

Here's an example on what that might look like

![null](/images/org.png)

This type of organization probably looks familiar. You have a series of teams inside an organization with their own lists of priorities attempting to influence a series of product managers to get their asks prioritized across a set of compartmentalized teams. 

To get a feature done, you need to talk to several PMs and influence the roadmap of several teams, and hope that all the items which touch your ask across functional component teams get done (hopefully close together in time) as well as worrying about all the inter-operability and integration points between the component teams. Then there's the whole release train that involves integrating the work, packaging it up, testing it somehow, and releasing to production.

![null](/images/tincantelephone_7099.jpeg)

Your engineering teams are 2-3 levels away from the consumers of the application, the chances of information loss in the telephone game is very high, which effectively means that you will often end up building solutions that aren't quite right. 

Each group has competing objectives and incentives. Your marketing team wants a website update pushed out in time for a conference, your sales team is trying to close a big deal that includes a SoW that was a slight over-promise on a timeline, your support team is worried about a bug that has been opened a while back, etc... Each of these actors is pushing on PMs and teams that themselves have different priorities.

As the number of fingers in the pie increases, you might start to notice an increasingly fragmented set of experiences.

Component teams being pushed in different directions might start letting quality slip, throwaway code never gets refactored (if it ain't broke...) and some parts of the app start getting harder to change (testing cycle time increases, simple asks appear to take disproportionate amounts of time).

As more and more things are getting done in parallel (in an attempt to get more stuff done in our waterfalls), we start to encounter another uncomfortable truth: The more we try to do in parallel, the slower velocity becomes. This is known as [Little's law](https://en.wikipedia.org/wiki/Little%27s_law), and can be paraphrased as: If we have a consistent speed of things getting completed, the amount of time it takes to complete each item is directly proportional to the number of things we are working on. This concept applies at all types of work effort and all teams inside an organization (individual teams, feature level, organizational initiatives, etc...). From this lens, an impediment to flow is starting too many things simultaneously. Less (in progress) is more (in delivery).

None of this is necessarily pre-ordained, however I believe this to be a common pattern among early stage companies that have managed to achieve a certain scale without having been intentional about how they think about building their products and the organizational and process patterns to apply to guard against some of these pitfalls.



As with anything else in the universe, counteracting entropic forces requires an investment of energy.

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
