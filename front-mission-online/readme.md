# Front Mission Online field notes

This is Work-In-Progress report/personal notes of my ongoing attempts of poking around Front Mission Online. Pardon the dust.

## Intro - What FMO is?

As you might know Front Mission is a series of mecha turn-based strategy/rpg game, which initially happen back in 90s for PS1. It's a decent game, especially considering its sequential re-releases for DS and now remasters for PC and other platform. And in early 00s FMO like many games back then got a MMO treatment, resulting in Front Mission Online, which was a Japan exclusive MMO in Front Mission Universe, which was released on PC and PS2.

I personally not that big of a fan of Front Mission Series, and never had played FMO back in a day, but in my humble opinion a restoration of an MMO is sort of "ascension to godhood" in the world of online video games preservation, if you excuse my love for both esoteric and pretentiousness. And so I thought "why not?" back in 2024, and since then it's been an on/off project for me, which I tends to forgot about more than I don't. 

This games turns out to be a **very** educational experience so far, and recently (in March of 2026) I realized that I never made a write-up about it. So one random monday I decided that it's a good time to talk about this game, in hope that this will give me some motivation to continue and maybe someday (in next 5-10 years if I'm being generous) finish the project. Or give somebody else a guidance and saving them some time figuring out some initial details.

## Chapter 1 - Collecting all the info

Lucky for us game is preserved on web archive, where you can get installation disks for PC or PS2 copy, if you're into that. After installing it you're getting a lot of start up menu entries in Japanese, which I don't know. Aaand that's it.

![alt text](img/pol.png)

You also get something called `PlayOnline`. Launching PS2 copy also results in a lot of Japanese and that `PlayOnline` thing and that's the very first blocker you'll encounter.

![alt text](img/pol_ps2.png)

Lucky for you I spend 2 weeks reading about everything related to `PlayOnline` and will be happy to share with you!

In 2000 Square Enix launched a platform called `PlayOnline`, which was a launcher, storefront and a backend for the games, distributed though this platform. Those games included:

- Final Fantasy XI
- Front Mission Online
- Tetra Master

And some less notable others.

While Tetra Master and Final Fantasy XI were released in US version of PlayOnline, Front Mission Online never left Japan and was a very short lived title - from 2005 to 2008. This makes Front Mission Online a very obscure title with next to none publicly available research and data. I was thinking that that's pretty much it - pc version doesn't launch, ps2 version doesn't login, but I keep googling and whenever I searched for `PlayOnline` I stumbled into 1000 of pages about Final Fantasy XI. And checking pages related to it made me quickly realize - it's for better or for worse is very popular. So popular there are custom servers and a lot of tools. And custom servers means somebody knows something about `PlayOnline`.

And I also had a feeling that those games were using a very similar backend, due to them being released at the same time by the same studio and looking very similar.
![alt text](img/fmo_ingame.png)

![alt text](img/ffxi_ingame.png)

So excited that the answer is that close and that ffxi custom server can be probably repurposed, I begging digging some more, and I uncovered a very curious situation legal-wise.

Due to FFXI being popular, there are custom servers, but to launch game players still use `PlayOnline`with a patcher (more on that later) . There are some parties who have fully reversed `PlayOnline` and made a replacement for it, but since Square Enix is still running `PlayOnline` and official servers, they can't (or won't, but no judgment here) release the files. 

But Front Mission Series can't launch in that patched state, since its `PlayOnline` is unusable, so it require a full replacement, which is somewhat legal to do, since the game is dead.

And so there are 2 games released at the same time using the same tech, but 20 years later one of them is not dead while other is, which blocks preservation of the dead one.

Well, on paper my plan looked simple enough:

- Bypass launch of the game, by restoring `PlayOnline` functionality
- Use FFXI custom server to make one for FMO
- FMO is playable and I'm happy

Which as you could guess wasn't anywhere near simple, since it's not done yet. But we'll get to that in the next chapter.

## Chapter 2 - Game (and not only) architecture

After the installation you're present with a handful of links on you desktop:

- Play Online Viewer
- Configure Front Mission Online
- Play Front Mission Online

And files

![alt text](img/fmo_folder.png)

![alt text](img/pol_folder.png)

Booting pol.exe resulted in nothing, but I feel like I must show you this funky mess of 00s programming anyway.

![alt text](img/pol_viewer.png)

Anyway, `what do`? Launcher not launching. I have several dlls and everything is in Japanese. Adding those dlls and exes into a debugger reveal a grim picture - they're packed with something custom!

![alt text](img/pol_sector.png)

And in entry function `POL1` is getting decrypted and copied to `.text`. Well, not a big deal, I loaded dll via x32dbg and dumping dll via scylla and we now have a decrypted dll, ready to be decompiled. After brief analysis I now have a lot of code to work with, but dll export is strange.

```
DllCanUnloadNow
DllGetClassObject
DllRegisterServer
DllUnregisterServer
```

...what could it be?

Well, lucky for me as I previously talked there are tools for custom servers. And one of said tools is a launcher and a redirector, which is precisely is what I need. I open its code and...

...what the hell is `CoCreateInstance`?

### COM hell

Despite being in my 30 I'm still a rather young developer. As win sysadmin early in my IT career I encountered com libs maybe twice and remembered nothing, except that I had to register them. I suspect that most people have even less experience with COM stuff.

To save time I'll just explain the whole thing.

COM objects are system wide registrable libs with a common interface, which are added to system registry, after that we can call them via their unique uuid v4 called `CLSID`. This is used by Windows to this day for a low-level stuff. And Office and Explorer extension, go figure.

![alt text](img/oleview1.png)

For one reason or another somebody back in 00s decided that splitting `PlayOnline` and its games in several COM objects is the way to go, and it resulted in a mess that `PlayOnline` is. During installation of `PlayOnline` infused title installer register `PlayOnline`COM object and game COM object.

%Screenshot from OleView here%

Under normal conditions a game start happen as

- User launch `PlayOnline Viewer`
- User log in
- User start a game
- `PlayOnline Viewer` create an instance of `IPOL` Object from `polcore.dll`
- `PlayOnline Viewer` initialize it and setup the data by calling its functions
- `PlayOnline Viewer` create an instance of game COM object
- `PlayOnline Viewer` calls `GameStart` function of game COM object and passing `IPOL` COM object ptr into it

The launcher I talked about before (https://github.com/zircon-tpl/xiloader in case you're curious) is doing roughly the same, just doing the whole setup by hand with some patching for redirection.

Now we have a minimally required amount of knowledge to work with COM objects, so, let's try to do it in Rust!

