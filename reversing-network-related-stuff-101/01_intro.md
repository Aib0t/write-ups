# Analysis of unknown protocols PART 1: Introduction.

While protocols varies (http, grpc, RakNet, DNS, Mysql) their design stays the same across the board. So in this tiny lecture I'll try to explain what to look for, what to ignore and some caveats in protocol analysis.

Analysis of protocols shares a lot with binary file research and if you're experienced with one, you're gonna be good with another. It's all about finding out patterns and making sense out of them. It takes time, but it's a rather approachable task for a person with any skillset. 

Pretty much every protocol operates between levels 4 to 6 of OSI model and usually build on top of TCP or UDP (level 4). Then comes level 5 - session level. This is usually taken care for you by libs. And finally a level 6 - messages itself.

So we have a diagram of

`TCP/UDP packet -> Session -> Message -> Payload`

Pretty much you gonna be analyzing Messages to get payload/produce messages of your own. This mainly consists of 3 parts

- Encryption
- Payload formatting
- Signature

** Encryption**

Usually includes several key exchanges, rngs, ivs and other nasty crypto stuff.

**Signature**
To properly replicate packets (or to verify them) we need to know how to properly generate signature for a message. Usually this is a simple crc32/md5/whatever from unencrypted payload. Like, in 99% of cases.

**Payload formatting**

Understanding and figuring out payload formatting is what will take the most time. There are some hints that will aid you:

Payloads, in 99% of cases processed as a stream. This is the reason why in, 95% of cases first value will be a size. This rather simple fact defines 2 important points:

- reader needs to know how many bytes to process **exactly**
- reader must be able to process payload (no way!)

This is obvious, dumb even, but it helps with approaching the unknown. But let's break second point a little, taking grpc as an example.

In grpc you can define packets payload like

```
message Point {
  int32 latitude = 1;
  int32 longitude = 2;
}
```

in some imaginary language we can code it in binary as

`000B` - size of 11, so we know, how many bytes to process
`01` - packet type, so we know how many variables to read/where to route packet
`01` - data type of int32, so we know how many bytes to read next
`00000000` - int32 data
`01` - data type of int32, so we know how many bytes to read next
`00000000` - int32 data

we can write this data in binary, send over tcp/udp, and reconstruct payload at the receipting side.

__**REAL WORLD**__

And now you can safely forget whatever I said before, because real world has nothing to do with my world of ponies and rainbow I decribed a second before. The list of real world crap you gonna meet is endless. I will just provide some examples from real world

**Bit-sized payloads**

You would think that nobody will do such a thing, but you're very, very naive. Imagine you have not that many data types (string, number, float) so you can use 2 bits like

`0 0 0 0 0 0 1 1` 

to define your 3 types

And then you can use 32 bits to define you payload. Now you, as a researcher has 5 bytes which doesn't makes sense, since data is bitshifted, changes constantly and your naive approach of using full bytes doesn't work anymore.

**Xors and other bitwise operations**

More common than you think. 

 **Complex structures**

This is from blaze from EA. This is an actual packet.

```
BlazeSDK(0): "Main": Info:    = {
BlazeSDK(0): "Main": Info:     CDAT = {
BlazeSDK(0): "Main": Info:       LANG = 1701729619 (0x656E5553)
BlazeSDK(0): "Main": Info:       TYPE = 2 (0x00000002)
BlazeSDK(0): "Main": Info:     }
BlazeSDK(0): "Main": Info:     CINF = {
BlazeSDK(0): "Main": Info:       BSDK = "2.11.5.1"
BlazeSDK(0): "Main": Info:       BTIM = "Jun 14 2010 18:19:47"
BlazeSDK(0): "Main": Info:       CLNT = "moh-python-pc"
BlazeSDK(0): "Main": Info:       CSKU = "PC"
BlazeSDK(0): "Main": Info:       CVER = "PYTHONBETA550504"
BlazeSDK(0): "Main": Info:       DSDK = "7.7.0.0"
BlazeSDK(0): "Main": Info:       ENV = "prod"
BlazeSDK(0): "Main": Info:       LOC = 1701729619 (0x656E5553)
BlazeSDK(0): "Main": Info:       MAC = "00:00:00:00:00:00"
BlazeSDK(0): "Main": Info:       PLAT = "Windows"
BlazeSDK(0): "Main": Info:     }
BlazeSDK(0): "Main": Info:     FCCR = {
BlazeSDK(0): "Main": Info:       CFID = "BlazeSDK"
BlazeSDK(0): "Main": Info:     }
BlazeSDK(0): "Main": Info:   }
```

Yes, it has maps and unions.

```
public enum TdfType
{
    // TDF Type  /  C# Type
    Struct = 0x0,   //class
    String = 0x1,   //string
    Int8 = 0x2,     //bool
    UInt8 = 0x3,    //byte
    Int16 = 0x4,    //short
    UInt16 = 0x5,   //ushort
    Int32 = 0x6,    //int
    UInt32 = 0x7,   //uint
    Int64 = 0x8,    //long
    UInt64 = 0x9,   //ulong
    Array = 0xa,    //List<T>
    Blob = 0xb,     //byte[]
    Map = 0xc,      //Dictionary<T,T>
    Union = 0xd,    //KeyValuePair<T,T>
}
```

And now some technical info:

- Those 4 characters `FCCR` keys are bitshifted into 3 bytes
- after it comes 1 bytes of size and type where first 4 bits are type and second 4 bits are size. In case size is `1 1 1 1 0 0 0 0` next byte will have full size
- then comes the payload


__**Okay, what do?**__

After all that scary stuff above we are back to square 1 - look for patterns and similarities. Even without exact values you can figure out patterns. And then AFTER patterns you can get to figuring out values and how to write them. Write some code to read packets. Check it on all availably packets. If it fails - figure out why. The longer you stare into abyss of meaningless bytes the more meaning they have.

There are also many ways to approach it from programmer perspective, but as NTAuthority told me once - "research PDB and write a client".


Writing client is easier since it's a step by step process, rather than writing a full blow server and being overwhelmed when something out of 100 requests from the client breaks and you don't know what caused it.