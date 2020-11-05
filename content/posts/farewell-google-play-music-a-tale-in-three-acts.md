---
layout: blog
title: Farewell Google Play Music - A tale in three acts...
date: '2020-11-04T22:23:20-05:00'
cover: /images/google-play-music-could-be-dead-by-2019-replaced-by-youtube-remix.jpg
categories:
  - news
---
# Prologue: The death of a beloved friend...

I've been an avid fan of Google Play Music for several years now, and when I heard it was being shutdown I was quite sad. It did exactly what I wanted, namely provide a way for me to easily stream my music collection and buy new tunes.

I grew up buying physical things (first cassettes, then CDs), and the idea of renting my music collection never sat well with me.

So I figured I go ahead and whip up a replacement.

My requirements were pretty simple. First, it has to be inexpensive. GPM was free, some something as close to free as possible would be ideal.

Second, I wanted it to be under my control. Never again, will some company be able to deprive me of my music.

Third, I wanted to be able to play it from anywhere.

Finally my fourth requirement was that it has to be simple. By this I mean that I didn't want to have to manage any infrastructure. It had to be the simplest solution that wouldn't require me worrying about things like servers, patching systems, etc...

After some looking around and various projects, I couldn't find anything that felt good. Sure, I could install a server in AWS and run something like Ampache, but then I'd have to worry about patching it, securing it, and so on. Not good enough.

Let's build something!

# Act 1: Trials and tribulations in browserland

First stab: Build a progressive web app, backed by serverless infrastructure in AWS.

On the surface, this bad-boy checks all the marks.

* No servers to manage, everything is serverless inside AWS: S3 for storing files, Cognito for authorizing users
* Works from any modern browser... ....
* All the basics pretty easy to build
* Was able to find a reason to dig into things like service workers, and have implemented a custom browser cache for caching media files asynchronously.
* Finally got around to dig a bit deeper into React (hooks, etc...)

After polishing it up a bit, this is what I ended up with:

![null](/images/musakbox.jpg)

Code here: <https://github.com/marksteele/musakbox>

Phew! Looks not bad. Works great on my desktop and even installs itself as a chrome app. Neat, almost as good as a native app. I also did some testing with Electron but frankly once I had the chrome app done I figured I was done.

Works on Windows!

Works on Mac!

Works on my iPhone... almost!

This is when I discovered a sad truth. Not spending any significant amount of time writing for the front-end I didn't realize that in it's infinite wisdumb, Apple forces apps that render web stuff on iOS to use the somewhat backwards Webkit rendering engine instead of the blink engine. Also, Apple places a bunch of restrictions on background tasks, service workers don't work, etc.... So the neat shiny features I wrote (custom caching of my media files, etc... are now useless on iOS). 

Neat stuff from Chrome crippled by Apple. Bleh. App works on my phone, but doesn't feel good. Does not spark joy.

Fine.

# Act 2: A journey to live with the natives...

Be that way. Stupid Apple. Guess I have to go native. React Native.

Once again, same requirements as the first round, except this time we need to build a native app. 

There's a first time for everything, so I dipped my toe in the React Native world to see how the water feels. Water feels a bit clunky from a developer experience side to be honest.

I had to work around several kinks in the slightly weird ecosystem that exists to allow React Native to function. Nevertheless, I managed to get feature parity with my web version.

![](/images/home.png)

Code can be found here: <https://github.com/marksteele/MusakBoxNative>

# Act 3: Reflections

This was a fun little project because it was scratching an itch, and I'm quite happy with the results. I now have an app on my phone that will be my forever music app. Also have an app on my desktop/laptop to play music.

Apps and website sourcing my playlist from the same cloud storage. Very cheap to run (pennies a month), but definitely not something for beginners (need to know your way around AWS a bit, copy files to s3, etc...). I think I've documented things in the repo sufficiently for folks to be able to reproduce my success on their own.

It would have been great if iOS didn't cripple Chrome, I feel that the web approach would have been an elegant solution (write once, run everywhere).
