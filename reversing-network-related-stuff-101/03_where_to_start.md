# Analysis of unknown protocols in games. Chapter 3: Where to start?

And so, one random day a bright idea of a dead game back online struck you like a lightning. This brings us a question of "Where to start?". This question, while seemingly trivial, is actually has more than one answer and is deeply rooted in extremely poorly documented field of "History Of Online Network Middlewares".

So in this chapter we'll talk about how to approach this task and how to start like a pro, increasing your chances for success by a a lot.

## How to start if you're a complete beginner and dooming yourself to fail as a result

Well, let's imagine that one particular day you decided to bring back your favorite racing game "Blur" back to life. 

You obviously grab a copy of the game, download a something you've heard about called "decompiler" in form of Ghidra. Maybe even a Wireshark, to grab packets.

In your novice mind you imagine that whole structure looks something like

```
Client <---> Server
```

And that the whole task is rather simple - game sends packet, server replies.

So you launch the game and...what now? Well, of course, you might eventually get to the point where you also launch Wireshark in parallel and able to dump DNS requests for domains of `amax-pc.auth.mmp3.demonware.net` and `amax-pс-prod.lsg.mmp3.demonware.net`. You also see a request on UDP port of 3074 to `stun.demonware.net`. This tells us next to nothing. So we try to load game in decompiler, which result with us being present with a screen of

%Screenshot here%

Cool.

What now?

At this point you realize that you understand nothing. This is all way too complex, way too overwhelming and clearly require a skillset that you lack.

Let's imagine that you're slightly smarter than 99.9% of people attempting such task, and you figured out that Blur, after resolving `amax-pc.auth.mmp3.demonware.net` and `amax-pс-prod.lsg.mmp3.demonware.net` attempts to open a TCP socket at port 3074. You add dns records to `hosts` file pointing at local server, make a dummy TCP server, and finally got your first piece of data

```
14000000000a51a9caad13daa40000c084d65e10248d0100
```

Well, what now? Where to go from there? How to make sense from those bytes and most importantly, how to reply to it? 

And that's where 99.8% of people give up and move on with their life, since the basic knowledge of IT stuff end here. But this is the exact place, where what we call "Fun" begins.

## Why the approach above fails for most people

You see, the approach above technically works, but it works only for people who are dedicated enough to keep digging until there is nothing to dig. You need to be a very dedicated person to achieve results from the above approach and there some cases where the approach above is the only way. But due to nature of the field, it's the hardest one here and there are numerous other approaches to making this task easier.

By "easier" I mean, that you won't be able to skip learning how to use decompiler, write code, learning what TCP and UDP is and how encryption works, but it will cut your work time from 3000-5000 hours to something closer to 200-500 as a result.

To paint a whole picture we'll need to do a big detour, but it do matter. A lot!

### Unknown world of Online Network Middlewares

If you have read mine story about it (which should be available in this repo) game developers were reusing a lot of stuff. But to save us time, I'll just tell a short version of it.

Back in 2000-2010 3 companies dominated the market of online solutions for videogames:

- GameSpy
- Demonware
- Quazal

Those were used in 1000+ games combined, and that affected the reverse engineering field of the games as a whole. Basically speaking if you add support for 1 title you're able to revive 10-100 more. It's a very crude statement, but will do for simplicity here.

This if an often overlooked fact, which affects how RE of online in those is done. There is also a second, even less known fact.

## (not very) Underground market of game prototypes

Name of this sensational on purpose, but it's on purpose.

Due to nature of game development, a lot of stuff is left out during development. Games are a curious form of art, where intermediate product is often more interesting than the final one.

And so, since forever, collectors of oddities have been interested in game prototypes. Those were dumped from dev cartriges, devkits of consoles being sold by workers of various facilities and other questionable sources and collected by said collectors. And as you could guess from the title, it created a rather unknown market where people trade stuff like that.

But not all game prototypes are posing such value, so they have been made public here and there. The biggest source of information regarding such public game protos is https://hiddenpalace.org/ 

So, why is this important? Well, it's a combination of 2 factors.

In previous chapter we talked about symbols, which makes RE 10 times easier. And those protos often contain them.
Factor number two is that games engines and tooling evolving slowly. For example, there is no symbols for Need For Speed Carbon. But there are for Need For Speed Prostreet, the next game in the series, which share roughly 60% of its codebase with Carbon. And in network part of the game this value is around 90%.

So by using symbols from PS3 version of Prostreet we can make reversing of Carbon network stuff much more approachable.

There is another resources of https://debugging.games/ where people share game binaries with symbols, and not whole protos, which is am extremely useful resource in this field of work.

## Summing it up

So, in bad case scenario you will:

1) Get the game
2) Try to run it
3) Try to decompile it
4) Fail miserably since it's hard

While instead you should be doing:

1) Gather information about the game, developer, publisher and release year
2) Gather information about used tech in the game
3) Grab samples of binaries
4) Grab siblings, protos and games sharing the same tooling
5) Search github for opensource solutions
6) Decompile the game and all samples
7) Searching for clues in decompiler
8) Search for packet captures for the game or related games
9) Running the game
10) Capturing traffic
11) Writing a server stub
12) Implementing packet handling
13) Writing a server

We'll talk about each step in greater detail in next chapters, but let's take a look at Blur with "How it should be done" approach.

## How to start if you're doing it for 100th time and know everything there is to know

> This chapter is written like that on purpose. Me being unable to properly express it means that I'm not too good as a writer. Oh well.

So, Blur!

Developer by Bizarre Creations, with Activision as a publisher and released in 2010. Previous games are Project Gotham Racing, which are Xbox 360 exclusives, so, probably not relevant.

So we grab a sample for Blur binary from a website with no-cds for educational purposes only, since downloading a whole game takes too much time, and we're still busy bidding for a licensed DVD on ebay.

Activision as a publisher in 2010 means that game with 99% certainty is using Demonware as its backend for online multiplayer. And since it's a game from 2010, it's most likely a Demonware 2.0. Since it's Demonware, we need Demonware samples. There are a lot, but the most notable is "Call Of Duty Blackops Dedicated Server with symbols", which we find on the very first page of google.

We also checks hiddenpalace and debugging.games, and surprise surprise - there are prototypes with symbols for us to use. So we also grab those

So now we have

- Blur binary
- CoD Server binary with symbols
- Blur PS3 version with symbols

We load them into decompiler, and enjoy a lot of nice symbols and begin cross referencing functions with names `auth`, `connect`, `packet` with PC binary.

While those are loading, we search github for anything `Demonware`, and lucky for us we find several repos with various implementations of it.

And very conveniently our DVD of Blur from ebay arrives, so we install the game, start wireshark, and begin capturing the traffic. We also redirect it locally.

While we're doing all of that we're making a server stub, based on existing repos, decompiled code and general sense. 

After several weeks we made game to proceed from 1st packet to a next one. Than to the next one.

After a month we can host sessions, and connect players.

After another month we have a server with persistent data, stats and everything is working like it was!

Great job us! It was easy, wasn't it?

## Putting jokes aside

What I described above is a exaggeration, of cause, but it has a lot of true to it. Not every game would be like that, since each game is unique, but generally this is the approach you want to follow.

Sure, more often than not, there would be nothing to help you. No symbols, no data, no captures, nothing. Such games pose a lot of challenge, but they're much more interesting to work on, in a sense. But information gathering plays a huge role in reversing any title and can save you a lot of time.

To put things into perspective, my initial Blur attempt took me 6 months of working 16 hours a day and I did it with "bad" approach. Now I estimate that it could be done in 2-3 weeks, with a proper approach.

Another example is work of my friend halvors, who spend 2 years reviving Far Cry 3, due to protocol complexity alone. And it took him 2 years with 5 years of prior RE experience.

But the main thing that made us achieve the end goal is `determination`. And even if starting is hard, don't let it discourage you. It gets easier. After 200 hours or so.