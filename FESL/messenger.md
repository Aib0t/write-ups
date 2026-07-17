# Messenger

As was stated in readme.md, fesl controls 4 entities:

- Roster (aka friend list)
- Invites
- Presence
- Messages

Those are mostly separated, but has their own quirks in inner working.

## Messenger internals

As of now, it's really clear that all 3 components of FESL were done either by 3 separate teams at the same time, or separately across several years. Plus, based on packets structure and the fact that messenger was existing as early as 2002, I suspect the second option, but it doesn't eliminates first option. Go figure.

So while all of those are using FESL protocol of fields and keys, Messenger does it in its own, as I call it "legacy" way. Plus code structure is very different from theater and fesl, so figuring out packets require more dynamic analysis then other 2.

While theater and fesl has mentions of task or TXN in their decompiled functions, making searching for writers/readers of packets a manageable task, messenger is using dynamic registration.

The main issue with messenger were hard to find handlers for numerous out-of-order packets (not direct replies to the calls). 

```C++
void __fastcall CreateMSGReplyHandlers(int param_1)

{
  int *piVar1;
  
  piVar1 = (int *)(**(code **)(**(int **)(param_1 + 0x10) + 4))();
  (**(code **)(*piVar1 + 0x1c))(&DAT_00bcbb84,ADMN_msg,param_1);
  piVar1 = (int *)(**(code **)(**(int **)(param_1 + 0x10) + 4))();
  (**(code **)(*piVar1 + 0x1c))(&DAT_00bcbb3c,GNOT,param_1);
  piVar1 = (int *)(**(code **)(**(int **)(param_1 + 0x10) + 4))();
  (**(code **)(*piVar1 + 0x1c))(&DAT_00bcbba8,RNOT_wrapper,param_1);
  piVar1 = (int *)(**(code **)(**(int **)(param_1 + 0x10) + 4))();
  (**(code **)(*piVar1 + 0x1c))(&DAT_00bcbb24,FUN_00914b20,param_1);
  piVar1 = (int *)(**(code **)(**(int **)(param_1 + 0x10) + 4))();
  (**(code **)(*piVar1 + 0x1c))(&DAT_00bcbb9c,PGET,param_1);
  piVar1 = (int *)(**(code **)(**(int **)(param_1 + 0x10) + 4))();
  (**(code **)(*piVar1 + 0x1c))(&DAT_00bcbbf0,FUN_00916e80,param_1);
  piVar1 = (int *)(**(code **)(**(int **)(param_1 + 0x10) + 4))();
  (**(code **)(*piVar1 + 0x1c))(&DAT_00bcbbcc,&LAB_0091acc0,param_1);
  piVar1 = (int *)(**(code **)(**(int **)(param_1 + 0x10) + 4))();
  (**(code **)(*piVar1 + 0x1c))(&DAT_00bcbb18,FUN_00915ca0,param_1);
  return;
}
```

Exhibit A is the registration function for all the handlers. The main issue here is that while functions are here and we can guess what functions are doing what, but we can't do it with 100% certainty. My best guess was that those `DAT_` should contain task names, but those are empty. 

So to get the concrete answer we need to dump those `DAT_` content from program memory, which as a result show us what functions is doing what.

```
ADMN  0x00BCBB84  -> RADM_msg_wrapper
GNOT  0x00BCBB3C  -> FUN_00915c90
RNOT  0x00BCBBA8  -> ROST_WRAPPER2
PING  0x00BCBB24  -> FUN_00914b20
PGET  0x00BCBB9C  -> FUN_00914b30
RECV  0x00BCBBF0  -> FUN_00916e80
ROST  0x00BCBBCC  -> LAB_0091acc0
USER  0x00BCBB18  -> FUN_00915ca0
```

After that it's a relatively simple task of "putting a lot of dedication to make it work". But let's talk about each component separately.

# Friends (Roster)

Roster nightmare begin with a "simple" packet of `RGET`.

