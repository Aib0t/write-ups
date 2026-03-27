# Analysis of unknown protocols in games. Chapter 2: What to know?

To even start such an adventure, you first must understand what the task in front of you takes to complete. I in no way discourage you from trying, making mistakes and learning from them, but you must understand that this is a task for no mere mortal.

You see, reverse engineering is tough. It's the closest thing in the world, which can make you feel like mad scientist and Indiana Jones at the same time, but it require a tremendous dedication to the cause. At this moment in time you probably feel like you can make it, but trust me - you will be thinking about giving up and moving on more time than you can count.

Proceeding further will require you to accumulate a lot of knowledge from various fields. It's sadly not optional. You're now a one man army, filling all the roles in the project and nobody will help you, but yourself. 

## Programming skills

I will be bold here, saying that knowing how to code is a must. Doesn't matter what language, doesn't matter if you code with LLM or by hand like our ancestors, but it's not an option, it's a must. 

> While I would probably get mean looks for mentioning LLMs here, I fully realize that coding field had changed. The wall to enter programming is now as small as it ever was and LLMs are here to stay, despite us like that fact or not. Yes, LLM generated code sucks. We all hate it. But at the of the day it's a tool like many others before it, and it all depends on how you use it.

For all the people who decide to vibe-code a server emulator for a game my advice is: **Don't**. The main issue is not even in the code quality, but the problem of "You don't learn shit while using LLMs to code". And reverse engineering is fully build on you doing "educated guesses" which you later implement in the code. Without learning how something works you might get as far as making game connect, but it won't allow you to make a good product or get far enough.

## Networking

I don't expect you to be able to name all 7 layers of OSI model from memory, but you would be dealing with networking protocols, so you need to at the very least to know:

- TCP vs UDP
- DNS
- UDP broadcast packets
- NAT, NAT punching
- STUN, TURN
- TCP and how it works. ACK, RST and all others
- Wireshark as a tool
- Learning how webrtc protocol functions will help you by a lot, trust me

Without this knowledge it's possible to proceed to a particular point, but more times than not not knowing those things will lead you to wrong conclusions, and this will hurt your progress.

## Decompilers, Reverse Engineering, CPU architectures, OS architectures, assembler and reading C code

This one is a never ending story. 

You see, due to a nature of this skill I don't think there is a feasible "endgame" here. In programming, we can realistically measure one person skill by giving them task and observing how they solve it. There are rules to the game, better ways to do something, libraries, frameworks and language abstractions. In reverse engineering there are no such things, generally speaking.

Yes, of cause, there are skills to get. Like, if I ask to people with 0 years of experience and 5 years of it to do a task of "find me a network packet builder" their required time will be something around 2-3 weeks for the first, and 15 minutes for the second. But let me explain from the very begging.

### Assembler and binaries

It all starts with the binary code. The one inside your `program.exe`. But here is the first catch - Windows is not the only OS in the world.

You see, computers varies. Your trusty Windows PC is working with x86 CPU. Your phone is using ARM CPU. PS3 and Xbox 360 are using PowerPC CPU. Those CPU are using "instructions". Those instructions looks something like

`0xAA 0xBB 0xCC`

Let's imagine for simplicity that `0xAA 0xBB 0xCC` translates to `Copy value of 0xCC to 0xBB`. x86 CPU uses one instructions, PowerPC use another. Those instructions combines into what is know as "assembler code". You, in theory, can program in raw assembler, but such endeavour require a tremendous amount of skill, and usually not done in this day and age. 

And so each CPU has its own set of instructions which it run on the CPU for everything, from calculus to displaying images. But at the end of the day it's just a different amount of platform specific assembler code.

### Decompilers and reading C code

Now decompilers comes into play. Their task is to convert this raw assembler to its approximately correct C counterpart. Now the tricky part - this is all based on an assumptions. 

For explanation let's take this C code

```C
int add_and_check(int a, int b) {
    int sum = a + b;

    if (sum > 0)
        return 1;
    else
        return 0;
}
```

when compiled into platform specific assembler we will get this code for x86

```C
; int add_and_check(int a, int b)
; arguments:
;   a -> [esp+4]
;   b -> [esp+8]
; return value in eax

add_and_check:
    mov eax, [esp+4]     ; load a
    add eax, [esp+8]     ; eax = a + b

    cmp eax, 0           ; compare sum with 0
    jle not_positive     ; jump if <= 0

    mov eax, 1           ; return 1
    ret

not_positive:
    mov eax, 0           ; return 0
    ret
```

and in PowerPC

```C
# int add_and_check(int a, int b)
# r3 = a, r4 = b
# return value in r3

add_and_check:
    add r5, r3, r4      # r5 = a + b

    cmpwi r5, 0         # compare with 0
    ble not_positive    # branch if <= 0

    li r3, 1            # return 1
    blr

not_positive:
    li r3, 0            # return 0
    blr
```

And so decompiler will try to generate the initial C code from those 2, but it's all just a guess. Yes, sometimes guesses are correct. Sometimes they're bogus. And here comes you, with a task to evaluate produced C code. Games consists of 1000s of such functions and your task as a researcher to know what to look for and how to interpreter the result. 

### OS architectures and libraries

Here comes the final part - underlying OS. So, while each CPU and OS is unique, to ease the work of developers, they comes prebundled with libraries for basic stuff. For example in Windows pretty much everything network related will utilize `ws2_32.dll`. And pretty much everything DNS related call in any Windows binary under the sun will use a function of `gethostbyname` from `ws2_32.dll`, which you can track down via imports table, but I'm starting to use big words here. The same can be done for PS3 binaries and its `Np*` lib. Same for Xbox360 and its `xhttp` module.

But this knowledge is obscure, esoteric even. Sure thing, it's possible to find guides explaining that you need to look for `sendto` import from `ws2_32.dll` to hook packets being send, but it's a tiny part of the picture. 

And so, to get this knowledge you have 2 options:

- Do enough reverse engineering to learn it
- Program in low-level platform specific C to learn all default libraries names

Both are not the easiest thing to do.

### So, what do?

I honestly have no definitive answer for that. Sure, you should watch a guide or two about Ghidra (or IDA), learn how to navigate gui and how to search for stuff. But due to a very speculative nature of this skill, it's hard to guide you from zero to hero.

As an example, here is a funny story. 

Halvors (my partner in RE crimes) has been working in Far Cry 3 for 2 years. And for 2 years he couldn't find function responsible for parsing fields of the session payload. He traced code flow from sockets, searched for code in known code regions and was trying to be smart about it. It didn't work. So, he asked me for help.

I checked fields, which had values of

```
    GameModeProgress = 123,
    Version = 124,
    Unknown125 = 125,
    Unknown126 = 126,
```

And so instead of being smart I just searched for u32 values of those fields, since to me it was obvious, that something like 

```
MOV EXX VALUE
```

was used. And I found the function in under 10 minutes, concluding that 2 years long search.

This story servers as an illustration that even if you're as good as Halvors with 5 years of RE experience and close to 10 years of C++ experience, there are still some approaches you didn't think about are doable.

### To sum up

- Watch or read a guide or two
- Familiarize yourself with C++ and what `void` is
- Learn the difference between `stdcall`, `cdecl`, `fastcall` and `thiscall`
- What is `xref` and `polymorphic calls`
- Learn how to search in Ghidra/IDA
- What is `uint`, `int` `CString`, `ptr` and `* *`

Knowing those basics will help you to begin. The rest is up to you.

## Learn how to think as a human, as a machine and how to make educated guesses

This one is less about hard skills and more about "philosophy". But it's a very important skill to have.

Behind every piece of code there is a human (or humans), who designed and implemented said piece of code. Since it was made by human, you, as another human, can understand what another human was trying to achieve and what was their train of thought.

So a protocol has an encryption. This encryption is of type X, because it was the standard back in a day of the game release. It gets encryption keys from something, which was made for reasons. Your job is to unwind this core idea into steps of getting the key, setting the key, setting encryption and made a use of it. I know I'm making not that much sense here, and maybe it will become clear to you later on. But what I'm trying to say, think as a human implementing it, to understand why it was done and how it potentially work. This could give you insights even when there are none.

On another hand, you need to learn how to think as a machine. My personal example for that is from early demonware reversing days, when I was struggling with packets being sent split in different TCP packets, which was causing issues with mine very young code. And one day I got a thought "this is processed as a continuous stream of bytes!". Which made me change the whole approach from "per TCP packet" to "put all data in 1 continuous buffer and then process". Yes, for some of you this is obvious, but it wasn't for me.

Same goes for processing raw tcp streams. You must understand that machine doesn't know what and how to read, and all this info is happening in the code, and code can be reversed and understood.

### Educated guesses

This is a bread and butter of this field. 

Since you can't ask somebody or consult documentation, all you realistically have is what you know. And getting further, you're gonna be stuck in infinity loops of infinity attempts to solve something. Add byte, restart server, restart game, it failed again, another change, another attempt.

This skill is sadly not the one that could be thought normally or explain in words. It's more of a "Write down and remember everything and don't be afraid to test even the wildest theories" style of thinking. 

"If X is working like Y, there is a chance that Z was also implemented as X, since it resembles X partially" - embrace this idea. Use it.
