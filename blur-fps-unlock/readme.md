# Blur fps unlocker [DRAFT]

23/08/25 - unfinished draft, since project is not finished yet.

---

Sometimes we pick up battles which are not just uphill but are more in "completely perpendicular" territory. A wall, if you must. 

Blur is a gourges game. Way ahead of its time lightning system, wounderful effects and "dark and gritty Frutiger Aero with flat iconography" look are doing wonders for this game. But sadly it can be only enjoyed properly in 30 fps. Technically you do have 60 fps on PC, but there is a catch - it mess up physics.

If you never played Blur online religiously you are not aware about 2 major things in Blur competitive scene (yes it exists).

### 1 - Drifter

One of in-game ability you can select in online mode is called "Drifter". They are called "Mods" in the game, so don't get confused with actuall game modding. 

This ability reward you with a speed boost power-up every time you do a long enough drift. But there are no cooldowns or other restrictions on it, other than treveled distance in "drifting" state. Which, as you can guess, makes it possible to "chain drift", when you combine Drifter with speed boost powerups (called Nitros), making it possible to earn a VERY concerning number of speed boosts.

It's somewhat similiar to Mario Kart in that department. Just imagine that you can use mushrooms during drifting, which, if done correctly, gives you another mushroom instantly.

**This technic is extremely fps dependant and much much harder on 60 fps**

### 2 - FPS locking

"Nitro dirfting" resulted in pro players locking down fps to 30 on PC, to achieve the same PS3 physics. Some even go as low as 23, claiming that it feels "better".

---

Those 2 things are gonna be really important later on, so I'm laying them down here to set up a scene for a number of rediculous bugs we will encounter later on.

## But what would happen if just increase the FPS?

If you are somewhat familiar with older games or making them in general, you already have started picking up clues with previous part. But let's continue and check what is going on if we set fps to something like 300.

Things are gonna break. **A lot!**

1. Menu elements scrolling speed will become too high to use
2. Holding down a key makes it to repeat actions too fast. Not too much of an issue in general, but trying to use backspace become a nightmare
3. Droped power-ups (on-map pickups) sprite rotation speed will become too high
4. Car bahavior is unresponsive and feels like you're driving a truck
5. After nitro usage your car gets strainghtened much faster/stronger
6. Car gets pulled to the center of the road by some mystical force
7. Sound breaks down and become corrupted
8. If you join other people sessions you will be seens as lagging a lot
9. Steering animation is broken and almost instant
10. You can do a 360 when doing a "Reverse Nitro" (I'll explain it later, don't worry)
11. Menu background light trails effects are being played at increased speed
12. Car gain speed much faster

It's the moment when you tell yourself "Yep, that's an FPS-tied game engine" and you would be damn right.

### How tied are we?

Oh, we are **very** tied.

Back in 2010 Bizarre Creations decided that "if hardware can't push beyond 30 fps, we will target 30 fps and just hardcode everything for this exact fps"

I can't blame them that much due to circumstances, but I curse them and their decisions every time I dive into their code, due to 2 reasons:

1. They outsourced PC port to a UK studio "Spiral House", which targets 60 fps, but without any changes to physics calculations
2. They have a slow-motion capture mode, which does a thing akin to untieing physics to fps. But it only affects 2-3 FPS-tied calculations, from 50-100 existing in this game.

Yeah, so, everything is FPS based, unless it's not, which is mostly not the case. And when I say "everything" I mean "everything". The only notable thing that **not** fps-tied is effects playback system. Which was most likely done to make slow-motion captures possible.

This results in the most *bizarre* ~~crap~~ programming wonders I've witnessed so far. Given, I'm a more of backend programmer and when people come up with protocols they usually put a lot of thought about overral design of said protocol. And there we have a game that was waaay behind it's schedule to hit the market, so a lot of corners were cut.

Or nobody out of 300 people working on Blur thought that "hey maybe this game will be played in more than 30 fps one day".

In either way, let's just go though each known bug one by one and try to fix them.

## 1 + 2 | "Menu elements scrolling speed will become too high to use" and "Holding down a key makes it to repeat actions too fast"

Alright, time to check your knowledge. What is your first idea for this bug to happen?

<details>
  <summary>Click to see the answer!</summary>

If your answer is `actions are executed each frame` you are 100% correct!

</details>
<br>

Back when I first started working on this bug I was only aware about this first one, with menu scrolling speed being way too high on high fps. Which I tried to fix by overriding menu values for scrolling and speed and whatnot. It was a rather disasterous approach, but I didn't have another idea for the fix back then. Untill I was told that backspace is making text input unbearable on high fps and then it all fall into place.

Blur internally is using pub-sub approach a lot between its components. There are producers, there are consumers, and events with unique ids being send back and forth.

To support many input options in their game (xbox 360 controller, ps3 controller and a keyboard) code internally is separated into 3 main conponents

- Controller Processor
- Buttons controller
- UI Actions controller

Controller Processor is linking physical buttons with virtual buttons. It can be any supported input on any platform.

Buttons controller tracking virtual buttons for stuff like Up, Down, Confirm, Back, etc and generate input events for UI to act on.

UI Actions controller take UI-related actions and perform things like "scroll menu left", "open friends list", "show system-based virtual keyboard input" and so and so.

And all of those 3 are executed each frame. Yep.

This creates a horrible, horrible situation where everything goes wrong.

### How it works for 30 fps

1. Since we're polling controller **every frame** we have a default delay of 33 miliseconds between checks "if button pressed"
2. We also have the same delay in "Buttons controller" that adds time of current frame (33 ms) to a value of `TimeHeld` if button is pressed **every frame**
3. If button is being held for more than 1 second this set button state to "AutoRepeat" which results in sending events for Up, Down, Left or Right inputs **every frame** with good old 33 ms delay

This approach works great for 30 fps, producing a smooth menu scroll and responsive controls with a nice delay.

*I will still a catchphrase from ReasonsTV from YouTube here.*

**Sounds like a great idea! With the best of intentions! What could possibly go wrong?**

### How it works for 300 fps

1. Controls are now pulled each 3.3 miliseconds to check if button is pressed
2. Each 3.3 ms a value of 3.3 ms is added to `TimeHeld` variable
3. If button is held for more than 1 second we generating an "AutoRepeat" event each 3.3 ms.

 So instead of 30 events per second we're now sending 300. Those 300 events per second breaks all UI scrolling and text inputs (if button is being held) entirely, resulting in a big mess.

### How it was fixed?

To go around the issue I hooked "UI Actions Processor" and added a internal timer, that checks for "AutoRepeat" events. In case we get than 1 such event in last 60 ms (worked better than 33 ms, for some reason)  those are just dropped and not executed at all. 

```rust

static LAST_UPDATE: AtomicU64 = AtomicU64::new(0);

pub fn set_value(val: u64) {
    LAST_UPDATE.store(val, Ordering::Relaxed)
}

pub fn get_value() -> u64 {
    LAST_UPDATE.load(Ordering::Relaxed)
}

#[repr(C)]
#[derive(Debug, Copy, Clone)]
pub struct Message {
    pub message_type: u32,
    pub size: u32,
}

fn fake_ui_broadcast_message(_self: *mut c_void, message_ptr: *mut Message, mask: u32) {
    let message = unsafe { *message_ptr };

    #[cfg(debug_assertions)]
    if ![
        0xA2193004, 0xA2193005, 0xA2193006, 0xA2193008, 0xA2193035, 0xA2193036,
    ]
    .contains(&message.message_type)
    {
        debug!("message type :{:X}", message.message_type);
    }

    if !AUTO_FIRE_EVENTS.contains(&message.message_type) {
        return unsafe { UIBroadcastMessage.call(_self, message_ptr, mask) };
    };

    //This should be called only each 0.0166 seconds, or 0.033 for true expirience.
    //^ Didn't allign with actual tests

    let now = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .expect("Time went backwards") //Unwrap default?
        .as_millis() as u64;

    let last = get_value();

    debug!("delta {}", now - last);

    if now - last >= 60 {
        //60ms feels the best, but we're dangerously in 50+ ms territory
        set_value(now);
        debug!("Brodcasting the message");
        unsafe { UIBroadcastMessage.call(_self, message_ptr, mask) }
    } else {
        debug!("Skipping broadcasting");
    }
}
```

This allowed me not to mess with controller or in-game inputs, which are outside of UIs are affecting car controls, and just produce a nice, not chaotically scrolled UI. And properly working "backspace being held" behavior.

2 birds with 1 stone, hooray!

# 3. Droped power-ups (on-map pickups) sprite rotation speed will become too high

So, as you already know we're fps tied. So developers though it would be a good idea to tie some animations to a timer and some of them to a frame rate (or physics rate, but more on that later).

Power-ups rotation is one of them. 

Blur architecture is a curious one, which has a lot of common with Call Of Duty architecture and many other titles - C++ based engine with a layer of lua controlling it.

And instead of infamous COD dvars game has "Lua debug vars", exposing a lot of inner values and helping with finding interesting functions.

And so using this method I was able to find a variable, labeled `Perks.PickupActor.RotationSpeed`, and added a fps-tied calculation for a value of it. And this chapter would end here, but things are not that simple.

### How it was fixed?

The solution above worked, unless fps of the game would suddenly change - turns out this speed is defined on actor spawn, and is static though it's lifetime. Great for most cases, not great when you fps drops to 30 on level loading, and staying above it during the gameplay.

Lucky for me, inside `PickupActor::Update` rotation is calculated in a separate function, which I was able to hook. And another thanks to the `PickupActor::Update` accepting frame time, which is not used, frame time is easily avalible to us with a simple offset. Sure, I could take it from a known `GameInstance` static ptr, but hey, why not do it easier for me.

```rust
fn fake_actor_rotator(_self: *mut c_void, rotation: *mut f32, unk1: *mut c_void) -> *mut c_void {
    //Rotating Actor Bug fix

    //Problem: rotating things are rotating based on fps
    //Core issue: rotating is calculated as "pi/2 - rotating_value" each frame
    //This code fix this issue with a time_step or "Frame Duration" value neatly located at offset 0x5c from rotating value.
    //Could be potentially disasterous for any other rotating things. In this case - use code in comments

    //This is defined by global debug Perks.PickupActor.RotationSpeed and set for each powerup actor on actor creation
    let amount_of_rotation_to_do = unsafe { *rotation };
    debug!("Amount : {}", amount_of_rotation_to_do);

    let frame_duration_ptr = rotation.clone().wrapping_byte_add(0x5C);

    let time_step = unsafe { *frame_duration_ptr }; //I'll call it time step, since it's called that in the code
    debug!("Time step in rotator: {}", time_step);

    unsafe { *rotation = amount_of_rotation_to_do * (60.0 / (1.0 / time_step)) }; //adjusting amount of rotation for our timestep

    #[cfg(debug_assertions)]
    {
        let time_step_1 = unsafe { *rotation };
        debug!("Time step in rotator: {}", time_step_1);
    }

    debug!("Rotator called");
    unsafe { ActorRotator.call(_self, rotation, unk1) }
}
```

And so after some more math rotation actor was finally scaling with fps properly.

# 4. Car bahavior is unresponsive and feels like you're driving a truck

Oh, this one is nasty.

So, at this point in time we're finally at the point where you're somewhat aware of what developers had choosen to did with their approach.

For me, it was the very first thing I tried to do, and at this point at time I've redone it at least a dozen of times. But without getting ahead of myself, let's start with how it begin, what I've learned along the way and how it was fixed.

### PCGamingWiki being great

So at PCGamingWiki there is a curious piece of info

```
High frame rate

    This method can only fully restore some of the broken post-processing effects like motion blur while others (like lens flare) may still exhibit broken and inconsistent behaviors. That is if the new internal limit matches the desired external value.

    Simply running at a higher rate - regardless of this fix being applied - speeds up camera movement and sensitivity. This fix helps the sped up camera to render at higher internal values.

    High frame rate may be somewhat difficult to consistently achieve on some setups due to limited multi-threading. This can be alleviated by using dxvk to wrap D3D9 API calls to Vulkan. This reduces overall CPU usage and overhead but increases GPU usage.

Modify with hex editor

    Go to <path-to-game>.
    Open Blur.exe with a hex editor such as HxD.
    Replace all instances of 89 88 88 3C (60 FPS) with one of the following,
        89 88 08 3C for 120 FPS.
        39 8E E3 3B for 144 FPS.
        89 88 88 3B for 240 FPS.
        61 0B 36 3B for 360 FPS.
    Replace the first instance of 00 00 70 42 (60 FPS) with one of the following,
        00 00 F0 42 for 120 FPS.
        00 00 10 43 for 144 FPS.
        00 00 70 43 for 240 FPS.
        00 00 B4 43 for 360 FPS.

If gameplay speed becomes slow change other instances of 00 00 70 42; however, if not possible, all instances can be replaced at once without any noticeable changes to gameplay.
```

Okay, what do we have here. There are 2 known floats which controls physics. Let's take a look at the floats itselfs.

- 89 88 88 3C = 0.016666668
- 00 00 70 42 = 60.0

Remember, this is a guide for PC version of a game, which, by default, runs at 60 fps. So checking them like this you won't suspect a thing. They seems like a proper values for something running at 60 fps.

### The awful truth

But what if we double check those values in PS3 version of the game?

And then, we learn the dirty, dirty Blur PC port secret.

**Both PC and PS3 versions are using the same values for physics**

Which means, that PC version, targeting 60 fps, are using the very same values from PS3 version, targeting 30 fps. Which means, that PC port of the game, from its release was utilizing wrong values, which makes vehicle physics behaves 2 times less prominent on PC. **Which is fixed by setting game FPS on PC to 30.**

This tiny mistake from the developers doing PC port went unnoticed during testing and got into the game release. As we dwell deeper it become more apparent, that this was probably intentional, since fixing Blur code to properly support 60 fps is a monumental task, since whole physics engine is tied to FPS in one way or another. But I once again getting ahead of myself. Let's get back to me back then, being all naive, thinking that changing those 2 values will miracally fix the game.

### Attempt #1

Via search I found where value of `89 88 88 3C` was utilized. Let's check the whole code of the decompiled function, to get a better understanding of what we're dealing with.

```C++
void __fastcall .Game::C_GameWorld::UpdatePhysics(int GameWorld)

{
  int iVar1;
  int iVar2;
  float FrameTimeStep;
  float local_8;
  float fStack_4;
  
  iVar1 = *(int *)(GameWorld + 0x3b4);
  if (((iVar1 != 0) && (*(char *)(GameWorld + 0x1cd0) != '\0')) &&
     (*(int *)(*(int *)(pInstance->field3204_0xc90 + 0x30) + 0xa18) != 6)) {
    UpdateGroundPlane(GameWorld);
    local_8 = -1.0;
    FUN_00918de0(iVar1,&local_8);
    if (MakeSlowmoCaptureWork != 0) {
      _TimeStep = pInstance->time_scale_2 * pInstance->time_step_fps_tied;
      local_8 = _TimeStep;
      AIAmax::AmaxAIR::Update(0x1134d88,'\x01');
      FUN_00bad640();
      DAT_010d9f68 = DAT_010d9f68 + 1;
      UpdatePhysics(iVar1,&local_8);
      AmaxPerfLogging::AccumulateDuration(kPhysicsUpdate,0.0);
      DAT_011a9492 = 0;
      return;
    }
    FrameTimeStep = 0.01666667;
    if ((TimeStep50hz != 0) && (*(int *)(DAT_01143798 + 4) == 0)) {
      FrameTimeStep = 0.02;
    }
    FrameTimeStepCalculated = *(float *)&pInstance->field_0xcb0 * _DAT_0111ca98 * FrameTimeStep;
    iVar2 = 0;
    _TimeStep = FrameTimeStepCalculated;
    local_8 = FrameTimeStepCalculated;
    if (0 < *(int *)(GameWorld + 0x1ca4)) {
      do {
        if (iVar2 < MaxPhysicsUpdate) {
          AIAmax::AmaxAIR::Update(0x1134d88,~(byte)iVar2 & 1);
          AmaxPerfLogging::AccumulateDuration(kAiUpdate,0.0);
          FUN_00bad640();
          FUN_007c0ae0();
          DAT_010d9f68 = DAT_010d9f68 + 1;
          fStack_4 = local_8;
          UpdatePhysics(iVar1,&fStack_4);
          AmaxPerfLogging::AccumulateDuration(kPhysicsUpdate,0.0);
        }
        else {
          DAT_010d9f68 = DAT_010d9f68 + 1;
        }
        iVar2 = iVar2 + 1;
      } while (iVar2 < *(int *)(GameWorld + 0x1ca4));
    }
    DAT_011a9492 = 0;
  }
  return;
}
```

It's a lot of digest here, and I'm putting it here for sake of giving the most curious of you the whole picture. This is not by any means a 100% correct decompilation, since I'm very, very far away from it.

For the most curious people out there, asking "How did you get functions name?" I will reveal a secret. There is a leak of Blur PS3 certification build, which contain a 200mb DWARF file. While this is invaluable source of info when working with its PC counterpart, but due to some issues with decompilation, half of the functions are broken, some names don't correspond properly and it's still, a huge mess to work with.

So as we can see `FrameTimeStep = 0.01666667;` is a hardcoded value, which is being used by this function in the main loop:

```C++
FrameTimeStepCalculated = *(float *)&pInstance->field_0xcb0 * _DAT_0111ca98 * FrameTimeStep;
    iVar2 = 0;
    _TimeStep = FrameTimeStepCalculated;
    local_8 = FrameTimeStepCalculated;
    if (0 < *(int *)(GameWorld + 0x1ca4)) {
      do {
        if (iVar2 < MaxPhysicsUpdate) {
          AIAmax::AmaxAIR::Update(0x1134d88,~(byte)iVar2 & 1);
          AmaxPerfLogging::AccumulateDuration(kAiUpdate,0.0);
          FUN_00bad640();
          FUN_007c0ae0();
          DAT_010d9f68 = DAT_010d9f68 + 1;
          fStack_4 = local_8;
          UpdatePhysics(iVar1,&fStack_4);
          AmaxPerfLogging::AccumulateDuration(kPhysicsUpdate,0.0);
        }
        else {
          DAT_010d9f68 = DAT_010d9f68 + 1;
        }
        iVar2 = iVar2 + 1;
      } while (iVar2 < *(int *)(GameWorld + 0x1ca4));
    }
```

Amazing! So we can update this value before each call, to a value of `0.5/current_fps` and this will solve all of our problems this instant!

And that's exactly what I did. And it did fixed vehicle physics! Case closed, this chapter ends here.

**I wish so.**

While the above approach does fix basic vehicle physics,  there are so many other factors affecting car behavior that I still don't know if I'm aware of all of them, or if most of them are fixed. 

So after tests I got reports stating that physics is "not how it behaves with 30 fps" and got back to work.

### Attempt #2 - `MakeSlowmoCaptureWork`

If we look a little above our main loop, we will see a secondary, more early reurn happening:

```C++
    if (MakeSlowmoCaptureWork != 0) {
      _TimeStep = pInstance->time_scale_2 * pInstance->time_step_fps_tied;
      local_8 = _TimeStep;
      AIAmax::AmaxAIR::Update(0x1134d88,'\x01');
      FUN_00bad640();
      DAT_010d9f68 = DAT_010d9f68 + 1;
      UpdatePhysics(iVar1,&local_8);
      AmaxPerfLogging::AccumulateDuration(kPhysicsUpdate,0.0);
      DAT_011a9492 = 0;
      return;
    }
```

And there we can see that physics frame time step is equal 

```C++
      _TimeStep = pInstance->time_scale_2 * pInstance->time_step_fps_tied;
```

Back then I thought that it could solve other issues the game is having at other places, not realizing that problem is much deeper than I initially thought.

So enabling this value and shipping the build to the testers once again gave negative results.

Back to the drawing board I go!

### Attempt #3 - the missing 60.0

If you memory exceeds goldfish capabilities you remember that PCGamingWiki telling us to alter not 1, but 2 values, second one being 60.

After a week of trial end error and a lot of cross referencing I finally found another very important function - `.Game::C_GameWorld::Update`

To save your scroll wheel, I will hide it in a spoiler

<details>
  <summary>.Game::C_GameWorld::Update</summary>

```C++
void __fastcall .Game::C_GameWorld::Update(astruct_4 *C_GameWorld)

{
  int iVar1;
  uint IsPaused;
  undefined *puVar2;
  int *piVar3;
  int iVar4;
  uint uVar5;
  float fVar6;
  float TimeStep;
  uint *puVar7;
  float TimeStepSize;
  uint uStack_30;
  undefined4 uStack_2c;
  float fStack_28;
  uint uStack_24;
  undefined4 uStack_20;
  float fStack_1c;
  uint uStack_18;
  undefined4 uStack_14;
  float fStack_10;
  uint uStack_c;
  undefined4 uStack_8;
  float fStack_4;
  
  TimeStep = 60.0;
  fVar6 = pInstance->time_scale_2 * pInstance->time_step_fps_tied + C_GameWorld[5].field1097_0x4 ac;
  C_GameWorld[5].field1097_0x4ac = fVar6;
  if ((TimeStep50hz != '\0') && (*(int *)(DAT_01143798 + 4) == 0)) {
    TimeStep = 50.0;
  }
  C_GameWorld[5].field1098_0x4b0 = (float)((int)C_GameWorld[5].field1098_0x4b0 + 1);
  iVar1 = (int)(fVar6 * TimeStep + 0.5);
  *(int *)&C_GameWorld[5].field_0x4a8 = iVar1;
  C_GameWorld[5].field1097_0x4ac = C_GameWorld[5].field1097_0x4ac - (float)iVar1 / TimeStep;
  Scheduler::Flush(*(int *)&C_GameWorld[5].field_0x2d8);
  AmaxPerfLogging::AccumulateDuration(7,0.0);
  if (C_GameWorld[6].field_0x8 != '\0') {
    BeginCarEffectTasks((int)C_GameWorld);
  }
  ResetEnvironmentSettings((int)C_GameWorld);
  fVar6 = pInstance->field3229_0xcac;
  IsPaused = Framework::C_ApplicationBase::IsPaused(pInstance);
  if ((char)IsPaused == 0) {
    FUN_00ba99b0(0x1222380);
    FUN_00491250(DAT_01132cd4,*(int *)&C_GameWorld[6].field_0x1fc);
    FUN_0041a4b0((int)C_GameWorld);
    TimeStepSize = fVar6;
    FUN_00858930(C_GameWorld->field92_0x5c,&TimeStepSize,C_GameWorld[5].field1098_0x4b0);
    if (DAT_0111ca70 != '\0') {
      FUN_00b46d20(0x11b4a38);
    }
    if ((pInstance->field3204_0xc90 != 0) &&
       (iVar1 = *(int *)(pInstance->field3204_0xc90 + 0x50), iVar1 != 0)) {
      *(undefined4 *)(iVar1 + 0x1278) = 0;
    }
    UpdatePhysics((int)C_GameWorld);
    FUN_00b47390((int *)&DAT_011b4a38);
    if ((pInstance->field3204_0xc90 != 0) &&
       (iVar1 = *(int *)(pInstance->field3204_0xc90 + 0x50), iVar1 != 0)) {
      if (DAT_0111cce8 == '\0') {
        FUN_00b94b40(iVar1);
      }
      else {
        FUN_004fddf0(&PTR_LAB_00f6ce68,0,0,*(undefined4 *)&C_GameWorld[5].field_0x2d4,0,(void *)0 x0,
                     0);
      }
      FUN_00b96a70(*(int *)(pInstance->field3204_0xc90 + 0x50));
    }
    FUN_00467860();
    FUN_005f09c0(C_GameWorld->field108_0x78,fVar6,1.0 / fVar6);
    FUN_00ba7950(0x1222380);
  }
  GlobalUpdate((astruct_3 *)C_GameWorld);
  IsPaused = Framework::C_ApplicationBase::IsPaused(pInstance);
  fStack_28 = fVar6;
  if ((char)IsPaused == '\0') {
    TimeStepSize = 0.01666667;
    if ((TimeStep50hz != 0) && (TimeStepSize = 0.01666667, *(int *)(DAT_01143798 + 4) == 0)) {
      TimeStepSize = 0.02;
    }
    TimeStepSize = (float)*(int *)&C_GameWorld[5].field_0x4a8 * TimeStepSize;
    if (MakeSlowmoCaptureWork != 0) {
      TimeStepSize = pInstance->time_scale_2 * pInstance->time_step_fps_tied;
    }
    puVar2 = AmaxCam::CameraSystem();
    AmaxCam::C_CameraSystem::CameraChangeUpdate((int)puVar2);
    Scheduler::Flush(*(int *)&C_GameWorld[5].field_0x2d4);
    piVar3 = Animation::C_TaskManager::Instance();
    Animation::C_TaskManager::Flush2(piVar3);
    piVar3 = Animation::C_TaskManager::Instance();
    Animation::C_TaskManager::ClearStats(piVar3);
    piVar3 = AnimatedModel::C_TaskManager::Instance();
    AnimatedModel::C_TaskManager::Flush(piVar3);
    AmaxPerfLogging::AccumulateDuration(7,0.0);
    Game::C_GameActorList::Update((int)C_GameWorld->field92_0x5c,fVar6);
    FUN_00523ca0((int *)C_GameWorld->field92_0x5c[8],0x30cc003,0xffffffff);
    uStack_30 = 0x30cc001;
    uStack_2c = 0xc;
    .Messaging::C_MessageSubscriberSystem::BroadcastMessage
              ((int *)C_GameWorld->field92_0x5c[8],&uStack_30,0xffffffff);
    FUN_00523ca0((int *)C_GameWorld->field92_0x5c[8],0x6186e011,0xffffffff);
    uStack_24 = 0x648cc003;
    uStack_20 = 0xc;
    fStack_1c = fVar6;
    .Messaging::C_MessageSubscriberSystem::BroadcastMessage
              ((int *)C_GameWorld->field92_0x5c[8],&uStack_24,0xffffffff);
    AmaxPerfLogging::AccumulateDuration(3,0.0);
    FUN_00b22150();
    FUN_00492fc0(C_GameWorld->field950_0x3e0,fVar6);
    if ((C_GameWorld->field944_0x3d4 != (int *)0x0) && (C_GameWorld[6].field_0x8 != '\0')) {
      uStack_24 = 0;
      uStack_20 = 0;
      fStack_1c = 0.0;
      FUN_0086f240(C_GameWorld->field944_0x3d4,&uStack_24,fVar6);
    }
    iVar1 = *(int *)(pInstance->field3204_0xc90 + 0x30);
    if ((iVar1 != 0) && (iVar1 = FUN_00b824a0(iVar1), iVar1 != 0)) {
      iVar1 = FUN_00b824a0(*(int *)(pInstance->field3204_0xc90 + 0x30));
      IsPaused = 0;
      iVar4 = FUN_0049ae20(iVar1);
      if (iVar4 != 0) {
        do {
          iVar4 = FUN_0049b130(iVar1,(byte)IsPaused);
          if ((iVar4 != 0) && (*(int *)(iVar4 + 0x274) != -1)) {
            Scheduler::Flush(*(int *)(iVar4 + 0x274));
          }
          IsPaused = IsPaused + 1;
          uVar5 = FUN_0049ae20(iVar1);
        } while (IsPaused < uVar5);
      }
    }
    AmaxPerfLogging::AccumulateDuration(7,0.0);
    IsPaused = Framework::C_ApplicationBase::IsPaused(pInstance);
    if ((char)IsPaused == '\0') {
      FUN_00523ca0((int *)C_GameWorld->field92_0x5c[8],0x1992005,0xffffffff);
    }
    Scheduler::Flush(DAT_01124080);
    FUN_005f1310(C_GameWorld->field108_0x78);
    FUN_005f2d10(C_GameWorld->field108_0x78);
    Scheduler::Flush(DAT_01247518);
    FUN_005f54b0(C_GameWorld->field103_0x70,pInstance->field3238_0xcb8,fVar6,pInstance->field16_ 0x10
                );
    AmaxPerfLogging::AccumulateDuration(5,0.0);
    FUN_005f13b0(C_GameWorld->field108_0x78);
    FUN_004bb530(C_GameWorld->field135_0x9c,pInstance->field3238_0xcb8);
    TimeStep = TimeStepSize;
    piVar3 = (int *)AmaxCam::CameraSystem();
    FUN_0040a360(piVar3,TimeStep);
    TimeStep = TimeStepSize;
    puVar2 = AmaxCam::CameraSystem();
    FUN_00409710((int)puVar2,TimeStep);
    fStack_10 = (float)MaxPhysicsUpdate;
    if ((float)*(int *)&C_GameWorld[5].field_0x4a8 < (float)MaxPhysicsUpdate) {
      fStack_10 = (float)*(int *)&C_GameWorld[5].field_0x4a8;
    }
    fStack_10 = fStack_10 * 0.01666667;
    uStack_18 = 0x6186e001;
    uStack_14 = 0xc;
    .Messaging::C_MessageSubscriberSystem::BroadcastMessage
              ((int *)C_GameWorld->field92_0x5c[8],&uStack_18,0xffffffff);
    if (C_GameWorld->field97_0x64 != (int *)0x0) {
      (**(code **)(*C_GameWorld->field97_0x64 + 4))();
    }
    if (C_GameWorld->field945_0x3d8 != 0) {
      FUN_008709a0(C_GameWorld->field945_0x3d8,(undefined4 *)&DAT_011580e8);
      FUN_008709c0(C_GameWorld->field945_0x3d8,(undefined4 *)&DAT_011580f8);
    }
    FUN_005f2af0(C_GameWorld->field108_0x78);
    FUN_00b27a60(C_GameWorld->field140_0xa4);
  }
  else {
    uStack_30 = 0x648cc008;
    uStack_2c = 0xc;
    .Messaging::C_MessageSubscriberSystem::BroadcastMessage
              ((int *)C_GameWorld->field92_0x5c[8],&uStack_30,0xffffffff);
    puVar7 = &uStack_30;
    IsPaused = 0xffffffff;
    iVar1 = Game::C_Game::GetInstance();
    piVar3 = (int *)GetMessageSubscriberSystemFromCGame(iVar1);
    .Messaging::C_MessageSubscriberSystem::BroadcastMessage(piVar3,puVar7,IsPaused);
  }
  puVar7 = &uStack_c;
  IsPaused = 0xffffffff;
  uStack_c = 0x648cc004;
  uStack_8 = 0xc;
  fStack_4 = fVar6;
  iVar1 = Game::C_Game::GetInstance();
  piVar3 = (int *)GetMessageSubscriberSystemFromCGame(iVar1);
  .Messaging::C_MessageSubscriberSystem::BroadcastMessage(piVar3,puVar7,IsPaused);
  return;
}
```

</details>

I'm still yet to understand everything about the function, but in short, that's where a lot of things related to the game simulation is processed. **But not all of them. Of cause not.**

We see the 60.0

```C++
  TimeStep = 60.0;
  fVar6 = pInstance->time_scale_2 * pInstance->time_step_fps_tied + C_GameWorld[5].field1097_0x4 ac;
  C_GameWorld[5].field1097_0x4ac = fVar6;
  if ((TimeStep50hz != '\0') && (*(int *)(DAT_01143798 + 4) == 0)) {
    TimeStep = 50.0;
  }
```

And familiar block of

```C++
    TimeStepSize = 0.01666667;
    if ((TimeStep50hz != 0) && (TimeStepSize = 0.01666667, *(int *)(DAT_01143798 + 4) == 0)) {
      TimeStepSize = 0.02;
    }
    TimeStepSize = (float)*(int *)&C_GameWorld[5].field_0x4a8 * TimeStepSize;
    if (MakeSlowmoCaptureWork != 0) {
      TimeStepSize = pInstance->time_scale_2 * pInstance->time_step_fps_tied;
    }
```

Altering both of those values each frame accordingly to fps resulted in positive feedback from the testers. They said it wasn't perfect, but was def an improvment and car was controlled similiar to 30 fps - sharp and swift.

```rust
fn fake_physics_update(_self: *mut c_void) {
    //Physics fix

    //Problem: physics updates are tied to fps
    //Core issue: physics is updated each frame, but with fps increase it's still calculated with 30 fps as a target
    //With this hook we recalculate correct physics timesteps and values based on fps before physics update is performed

    let p_instance = match get_p_instance() {
        Some(p_instance) => p_instance,
        None => {
            //Should never happen, but just in case
            debug!("Instance is null!");
            return unsafe { PhysicsUpdate.call(_self) };
        }
    };

    let ptr_base: *mut std::ffi::c_void =
        unsafe { GetModuleHandleA(PCSTR::null()) }.unwrap().0 as _;

    let timestep: f32 = (p_instance.fps1 * 2.0).round(); //in base ps3 code equal 60 with 30 fps target
    let timestep_size: f32 = 1.0 / timestep; //in base ps3 code equal 0.01666667
    let physics_updates = (0.07 * timestep).round() as u32; //in base ps3 code equal 4. It's effect is rather unknown, but I update it anyway

    //Horrible, replace with direct ptrs for optimization
    let ingame_timestep = ptr_base.wrapping_byte_offset(TIMESTEP_PTR - EXE_BASE_ADDR) as *mut f32;
    let ingame_timestep_size =
        ptr_base.wrapping_byte_offset(TIMESTEP_SIZE_PTR - EXE_BASE_ADDR) as *mut f32;
    let ingame_max_physics_updates =
        ptr_base.wrapping_byte_offset(MAX_PHYSICS_UPDATE_PTR - EXE_BASE_ADDR) as *mut u32;

    unsafe {
        ingame_timestep.write(timestep);
        ingame_timestep_size.write(timestep_size);
        ingame_max_physics_updates.write(physics_updates); //still not entirely sure this one is required
    }

    [cfg(debug_assertions)]
    unsafe {
        debug!(
            "Timestep: {}, size: {}, physics updates: {}",
            *ingame_timestep, *ingame_timestep_size, *ingame_max_physics_updates
        );
    }

    return unsafe { PhysicsUpdate.call(_self) };
}
```

Down the line those 2 changes also fixed some other bugs related to high fps.

But I was (and still is) far from being done. Yes, we do update world and physics at the correct rate. But what about `FakePhysics`?

# 5. After nitro usage your car gets strainghtened much faster/stronger

I taked about Drifter and Nitro Drifting at the very beggining. The bread and butter of Blur competitive scene. The very core of it. *The very thing I need to get right, or I will be in a grave danger.*

So while the fix above fixed how car behaves under normal condition, high fps broke "Nitro Drifting Chains", since after using a Nitro, you can would get straightened by the unknown force, which breaks your drift state, which breaks "Nitro Drifting Chains", which also could get my legs broken, in case I won't get it right.

There is a subset of things in Blur, called `FakePhysics`, which apply some "magical" forces to your car to do something not very ordinary. Like a 360 spin if mine is hit. Or a backflip. Or a, you guessed it, strighten your car after a Nitro.

And if you're made it this far you're already know it's tied to FPS, because **of cause it is**.

## How it was fixed?

Calculations for `FakePhysics` are not a part of the physics calculations and done separately in their own functions **every frame**, if `VehicleActor` is under any of them. Lucky for me, for "Nitro" it's done in `Game::Actors::FakeVehicleSpeedState::Update` in a rather simplistic manner of

```C++
    if (Perks.FakePhysics.Speed.AccelerateStabilise * 0.4469444 < fVar3) {
      fVar2 = (float)((uint)(Perks.FakePhysics.Speed.AccelerateStabiliseRate * fVar3 * param_4[0 x4f]
                            ) ^ 0x80000000);
      param_4[0x4a] = param_4[0x4a];
      param_4[0x49] = *param_4 * fVar2 + param_4[0x49];
      fVar2 = param_4[0x4b] + param_4[2] * fVar2;
      goto LAB_00488e00;
    }
```

With `Perks.FakePhysics.Speed.AccelerateStabilise` and `Perks.FakePhysics.Speed.AccelerateStabiliseRate` being Lua debug variables, the fix was rather straightforward - update those before execution to reflect current FPS.

```rust
fn fake_fake_speed_update(_self: *mut c_void, vehicle: *mut c_void, time_step: f32, unk1: *mut c_void ) {    
    
    debug!("_self: {:?} vehicle: {:?}, time_step: {:?}, unk1: {:?}", _self,vehicle,time_step, unk1 );

    let ptr_base: *mut std::ffi::c_void =
        unsafe { GetModuleHandleA(PCSTR::null()) }.unwrap().0 as _;

    let fake_physics_stabilization_rate =
    ptr_base.wrapping_byte_offset(FAKE_PHYSICS_ACCELERATE_STABILIZE_RATE - EXE_BASE_ADDR) as *mut f32;

    let fake_physics_stabilization =
    ptr_base.wrapping_byte_offset(FAKE_PHYSICS_ACCELERATE_STABILIZE - EXE_BASE_ADDR) as *mut f32;

    let new_rate: f32 = 10.0 * (time_step / 0.03333);
    let new_base: f32 = 5.0 * (time_step / 0.03333);
    unsafe {
        fake_physics_stabilization_rate.write(new_rate);
        fake_physics_stabilization.write(new_base);
    }

    unsafe { FakeVehicleSpeedStateUpdate.call(_self,  vehicle, time_step,unk1) }
}
```

Potentially, this fix can be applied to the hardcoded value of `0.44694445`, since it's only used in this function, but my tests resulted in a broken mess, so, I will just update those 2 for now on.


# 6. Car gets pulled to the center of the road by some mystical force

TODO

# 7. Sound breaks down and become corrupted

This strange bug never happen to me directly, but there are a lot of report from other people. 

To my personal suprise, this one was fixed together with physics fix, the culpit being the 60.0 value. Together with world simulation, this also affected audio subsystem, making it work as it should on higher fps.

I didn't say everything on that list is a complicated mess! Sometimes things fix themselves along the way.

# 8. If you join other people sessions you will be seens as lagging a lot

TODO

# 9. Steering animation is broken and almost instant

Just like with `PickupActor` rotation, this is yet another pi based calculation, which is done every frame, instead of being played based in a timer based manner. Go figure.

## How it was fixed?

TODO


# 10. You can do a 360 when doing a "Reverse Nitro"

This one is my personal favorite.

So in Blur pick-ups could be shot forward or backwards. This works more or less how you expect them to. Mines spawns behind your car when shot backwards or fly forward a limited distance when shot forward.

Except for the Nitro.

When shot forward, you get a speed boost. Basic stuff. But when shot backwards it does 2 things:

1. You freeze in place, with a limited steering control, around ~20 degree left and right
2. After a short delay you're shot forward with a lot of speed

As you could figure out from the name of this part, with increased fps you can do a 360 degree (or even higher) rotation, which looks amazing.

Yep, another rotation related bug. Pi be damned.

## How it was fixed?

TODO

# 11. Menu background light trails effects are being played at increased speed

Pretty light trails in the menu is yet another actor with fps tied animation. And this one once again was fixed via physics fix!

I'm honestly don't understand this game sometimes. But maybe since those don't rotate it's working as intended.

# 12. Car gain speed much faster

If you're familiar with Blur, you know it looks (and plays, for the most part) like a silly arcade racing game.

But we should not forget that Bizarre Creations were the makers of Project Gotham Racing. Which while having nothing related to Batman, was a semi-realistic racing sim.

And Blur is using a game engine from it. Which resulted in cars in this game to be a **very** complex entity. How complex you ask?


<details>
  <summary>An example of 1 vehicle</summary>

```json
{
        "index": 12,
        "data_type_name": "gears",
        "structure_offset": 360,
        "structure_type": 17,
        "data_read_type": 3,
        "data_structure_name": "RP_CarGears",
        "value": [
          {
            "index": 91,
            "data_type_name": "gearRatios",
            "structure_offset": 0,
            "structure_type": 7,
            "data_read_type": 256,
            "data_structure_name": "f32",
            "array_flag": 655361,
            "array_size_hex": "06000000",
            "array_size": 6,
            "array": [
              "********",
              "********",
              "********",
              "********",
              "********",
              "********"
            ]
          },
          {
            "index": 92,
            "data_type_name": "reverseGearRatio",
            "structure_offset": 8,
            "structure_type": 7,
            "data_read_type": 0,
            "data_structure_name": "f32",
            "value": 0.0
          },

...
      {
        "index": 13,
        "data_type_name": "tyres",
        "structure_offset": 392,
        "structure_type": 18,
        "data_read_type": 3,
        "data_structure_name": "RP_CarTyres",
        "value": [
          {
            "index": 98,
            "data_type_name": "frontTyres",
            "structure_offset": 0,
            "structure_type": 27,
            "data_read_type": 3,
            "data_structure_name": "RP_CarTyre",
            "value": [
              {
                "index": 131,
                "data_type_name": "radius",
                "structure_offset": 0,
                "structure_type": 7,
                "data_read_type": 0,
                "data_structure_name": "f32",
                "value": 0.326
              },
              {
                "index": 132,
                "data_type_name": "grip",
                "structure_offset": 4,
                "structure_type": 29,
                "data_read_type": 3,
                "data_structure_name": "RP_LinearGraph",
                "value": [

...

 {
        "index": 14,
        "data_type_name": "friction",
        "structure_offset": 608,
        "structure_type": 19,
        "data_read_type": 3,
        "data_structure_name": "RP_CarFriction",
        "value": [
          {
            "index": 100,
            "data_type_name": "airFriction",
            "structure_offset": 0,
            "structure_type": 7,
            "data_read_type": 0,
            "data_structure_name": "f32",
            "value": 0.596
          },
          {
            "index": 101,
            "data_type_name": "rollingFriction",
            "structure_offset": 4,
            "structure_type": 7,
            "data_read_type": 0,
            "data_structure_name": "f32",
            "value": 20.0
          },
          {
            "index": 102,
            "data_type_name": "constantDownforce",
            "structure_offset": 8,
            "structure_type": 7,
            "data_read_type": 0,
            "data_structure_name": "f32",
            "value": 0.51
          },
          {
            "index": 103,
            "data_type_name": "downforce",
            "structure_offset": 12,
            "structure_type": 7,
            "data_read_type": 0,
            "data_structure_name": "f32",
            "value": 1.45
          },
          {
            "index": 104,
            "data_type_name": "downforceOffset",
            "structure_offset": 16,
            "structure_type": 7,
            "data_read_type": 0,
            "data_structure_name": "f32",
            "value": 0.0
          },
          {
            "index": 105,
            "data_type_name": "rotationalAirFriction",
            "structure_offset": 20,
            "structure_type": 7,
            "data_read_type": 0,
            "data_structure_name": "f32",
            "value": 900.0
          }

```

</details>

The most important thing for us here is that there is no "car go this fast" value.

There is a fully simmulated RPM, defined as a graph, geard box simulation and a lot of other params, making the car to behave a certan way.

And higher fps makes car to get max RPM much, much faster.

## How it was fixed?

TODO