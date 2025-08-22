# FESL

As mention earlier, FESL plays a lot of roles, some of which are duplicating Theater functionality on paper. But in reality Theater only fills a role of a "Game Coordinator", which keeps track of sessions and players in them. The rest is done by FESL.
There are 18 known FESL services:

 1. **fsys** //FeslSystem    
 2. **acct** //FeslAccount    
 3. **achi** //FeslAchievments
 4. **gsum** //FeslGameSummary
 5. **pnow** //FeslPlayNow
 6.  **rank** //FeslRanking
 7. **asso** //FeslAssociation
 8. **blob** //FeslBlob
 9. **dobj** //FeslObject?
 10. **lddr** //FeslLadder
 11. **fdbk** //FeslFeedback
 12. **recp** //FeslRecorder
 13. **club** //FeslClub
 14. **fcms** //FeslContent?
 15. **fltr** //FeslFilter
 16. **fpla** //FeslFindPlayer
 17. **mtrx** //FeslMetrix
 18. **pres** //FeslPresence

Each service feature anywhere from 1 to 20+ tasks, refereed to as **TXN**. 

## FESL and ProtoSSL

The very first obstacle any researcher will encounter is a simple fact that FESL, unlike other services, will communicate with the server via TLS protected socket. In the modern day and age its 3 supported ciphers (`ASN_OBJ_RSA_PKCS_MD5`,`ASN_OBJ_RSA_PKCS_MD2` and `ASN_OBJ_RSA_PKCS_SHA1`) are extremely outdated and highly vulnerable to side-channel attacks, which resulted that no existing ssl/tls libraries supports them well. Technically, OpenSSL can be overridden to utilize those, but this is a huge headache for any researcher, trying to utilize any modern language and up-to-date libs. 

Lucky for us, Rust coders, we have https://crates.io/crates/blaze-ssl which, despite a name, is just Ssl3 socket library, with those 2 (no md2 for us!) ciphers implemented and ProtoSSL bug bundled in. 

Another (and more doable approach) is utilizing a [patch](https://aluigi.altervista.org/patches/fesl.lpatch) and the lpacher itself [here](https://aluigi.altervista.org/mytoolz/lpatch.zip). After that just patch the binary, and it will bypass the ssl check. After that just though in yet another **Luigi Auriemma** tool called [stcppipe](https://aluigi.altervista.org/mytoolz.htm#stcppipe) and launch it like

```
./stcppipe.exe -Y 1 -D 127.0.0.1 8080 18210
```
This will ensure that non-encrypted traffic is reaching your server, and so you can proceed with the rest.

## FESL protocol details

Without SSL protection in place the rest of FESL protocol is extremely easy to understand and work with. There are several caveats to FESL (and the rest of the Plazma), primary "out-of-order" packets, which complicates logic and makes research without references a pain. Lucky for us references exists, so almost all of out-of-order packets which plays major role in main flows are known.

A basic FESL packet will look like:

`fsys` - 4 bytes utf8 service name
`0x80000001` - 4 bytes of packet sequential id (and type, more on that later)
`0x255` - byte indicating payload size
`field=value` - any number of tag-value pairs, separated with `0xA` (newline)
`0x00` - final null byte

Sequential/Type id is important to let the client know what type of a packet we're sending.

Packets with `0x80XXXXXX` are a "sequential" packets. Initially, when implementing FESL we opted for keeping the counter as a connection property, and later simplified it to "reply with incoming id". So far it works without any issues.

Packet with an id of `0x80000000` are out-of-order packets, which comes exclusively from the server itself.

## Initial FESL flow

In this section I'll briefly describe steps client and server takes to get to "standby" connection state, meaning that all of this data will be send after client clicks "Log in", without getting much into details about other stuff. For a full list of known calls consult `known_fesl_tasks.md`. As an example a NFS: Carbon for PC will be used, since there are a handful of preserved logs made by https://github.com/xan1242/NFSC_MasterServer as well as an ability to dump said logs, which helps tremendously.

In general, initial flow looks like:
```
->N res fsys 0x80000001 {TXN=Hello, clientString=nfs-pc, sku=America, locale=en_US, clientPlatform=PC, clientVersion=1.0, SDKVersion=2.9.7.1.0, protocolVersion=2.0, fragmentSize=2048, clientType=client}
<-N res fsys 0x80000001 {domainPartition.domain=eagames, messengerIp=messaging.ea.com, messengerPort=13505, domainPartition.subDomain=NFS-2007, TXN=Hello, activityTimeoutSecs=0, curTime="Mar-17-2021 20%3a51%3a54 UTC", theaterIp=nfs-pc.theater.ea.com, theaterPort=18215}
<-A res fsys 0x80000000 {TXN=MemCheck, memcheck.[]=0, type=0, salt=458711633}
->R res fsys 0x80000000 {TXN=MemCheck, result=}
->N res acct 0x80000002 {TXN=Login, returnEncryptedInfo=0, name=xan1242, password=REDACTED, machineId=, macAddr=$f04e2abfa102}
<-N res acct 0x80000002 {lkey=W5NyZxHqi4vzoYp9Cki6GQAAKDw., profileId=907370134, TXN=Login, userId=907370134, displayName=xan1242}
->N res fsys 0x80000003 {TXN=GetPingSites}
<-N res fsys 0x80000003 {pingSite.0.name=nfs-wc, pingSite.1.type=0, pingSite.1.addr=159.153.72.181, minPingSitesToPing=0, pingSite.1.name=nfs-eu, pingSite.0.addr=159.153.70.181, pingSite.0.type=0, pingSite.[]=3, pingSite.2.addr=159.153.93.213, pingSite.2.name=nfs-ec, pingSite.2.type=0, TXN=GetPingSites}
```
There is a lot to unpack here, so let's go 1 by 1.

### Step 1 - Hello and MemCheck

```
fsys 0x80000001 {TXN=Hello, clientString=nfs-pc, sku=America, locale=en_US, clientPlatform=PC, clientVersion=1.0, SDKVersion=2.9.7.1.0, protocolVersion=2.0, fragmentSize=2048, clientType=client}
```
This is a `fsys/Hello` packet, which describes client properties to the server. While all of this data is highly valuable in real implementation, only `fragmentSize` plays major role, since it defines the size of packets payload. Which is rather silly, considering whole communication is done via TCP, but probably there were some reasons for this decision from EAs programmers.

To this packet server replies with not 1 but 2 packets. 1 a reply to `Hello` and 1 out-of-order packet of `MemoryCheck`
```
<-N res fsys 0x80000001 {domainPartition.domain=eagames, messengerIp=messaging.ea.com, messengerPort=13505, domainPartition.subDomain=NFS-2007, TXN=Hello, activityTimeoutSecs=0, curTime="Mar-17-2021 20%3a51%3a54 UTC", theaterIp=nfs-pc.theater.ea.com, theaterPort=18215}
<-A res fsys 0x80000000 {TXN=MemCheck, memcheck.[]=0, type=0, salt=458711633}
```
As you can see, `Hello` reply contains `messenger` and `theater` domains (or ips) and ports. Another thing which plays a significant role is a `domainPartition`, since FESL operates on "federation basis". At this point in time we haven't found any hardcoded checks for what values `domain` and `subDomain` can take, so we're free to utilize whatever values we want.

`MemCheck` is a anti-cheat/anti-piracy feature, which can check any memory available to the process. So far we have no examples of it, but whole parsing flow can be easily found in the code of clients.

### Step 2 - Login

To authorize client will send
```
->N res acct 0x80000002 {TXN=Login, returnEncryptedInfo=0, name=xan1242, password=REDACTED, machineId=, macAddr=$f04e2abfa102}
```
This is an example for PC login, for PS3 login a `TXN` of `LoginPS3` will be used, that sends a PSN ticket towards the client. Password is send in plain text (bad EA!).

In reply we get
```
<-N res acct 0x80000002 {lkey=W5NyZxHqi4vzoYp9Cki6GQAAKDw., profileId=907370134, TXN=Login, userId=907370134, displayName=xan1242}
```
Both `lkey` and `userId` are utilized down the road with Theater and Messenger, so those are should be unique in any real implementation. For research purposes we can neglect those requirements.

### Step 3 - the rest

"The rest" varies from title to title and to environment to environment. For example in some configurations NFS: Carbon will skip sending `GetPingSites`. 

```
->N res fsys 0x80000003 {TXN=GetPingSites}
<-N res fsys 0x80000003 {pingSite.0.name=nfs-wc, pingSite.1.type=0, pingSite.1.addr=159.153.72.181, minPingSitesToPing=0, pingSite.1.name=nfs-eu, pingSite.0.addr=159.153.70.181, pingSite.0.type=0, pingSite.[]=3, pingSite.2.addr=159.153.93.213, pingSite.2.name=nfs-ec, pingSite.2.type=0, TXN=GetPingSites}
```  
For NFS: Carbon here game sends ping sites to determine the best server location, which will later be used in PlayNow packets. Those IPs are just pinged with ICMP, nothing custom or out of ordinary.

## What happen next?
That's a rather tough question. If you're curious the best next stop is https://github.com/xan1242/NFSC_MasterServer/blob/master/EXAMPLE%20PACKETS/OnlineLog%20-%20game%20sess%20start2.txt which contains each packet. 

In short, after calls above client authenticate to Theater and Messenger, set stats and profile, get list of active sessions and finally gets into standby mode, where no new packets are send, without user input. For the majority of next FESL incoming packets a simplistic reply of
```
<-N res xxxx 0x800000XX {TXN=TaskName}
```
can be utilized, to report to the client that its data was accepted.

Since in the most cases this stats and telemetry bloat can be ignored, we can skip directly to matchmaking.

It's a good idea to stop reading the doc now, and come back to it later, since Theater will be playing a major role. So go read though `theater.md`, to get familiar with it.

## FESL PlayNow

Talking about PlayNow can't be done without Theater in the picture, so we'll assume that you read though Theater doc. 

Us, the authors, has been snooping though a lot of online middlewares and titles, but none of them gets close to what is happening in FESL matchmaking. To put it into perspective, the biggest payload for "get me appropriate sessions" seen before was containing 50+ parameters. And how exactly to matchmake a player is done entirely on the server side. But it's different for FESL.

In FESL case, the client will state how exactly to match itself to the server. To better understand it, let's take a look at packets.

```
->Q res pnow 0xb0000019 {size=18728, data=VFhOPVN0YXJ0CnBhcnRpdGlvbi5wYXJ0aXRpb249L0VBR0FNRVMvTkZTLTIwMDcKZGVidWdMZXZlbD1vZmYKcGxheWVycy5bXT0xCnBsYXllcnMuMC5vd25lcklkPTkwNzM3MDEzNApwbGF5ZXJzLjAub3duZXJUeXBlPTEKcGxheWVycy4wLnByb3BzLntzZXNzaW9uVHlwZX09ZmluZFNlcnZlcgpwbGF5ZXJzLjAucHJvcHMue25hbWV9PXhhbjEyNDIKcGxheWVycy4wLnByb3BzLntmaXJld2FsbFR5cGV9PXVua25vd24KcGxheWVycy4wLnByb3BzLntwb29sTWF4UGxheWVyc309OApwbGF5ZXJzLjAucHJvcHMue3Bvb2xUaW1lb3V0fT0yMApwbGF5ZXJzLjAucHJvcHMue3Bvb2xUYXJnZXRQbGF5ZXJzfT0wOjEKcGxheWVycy4wLnByb3BzLntmaXRUaHJlc2hvbGR9PTA6NTQwMApwbGF5ZXJzLjAucHJvcHMue3Bpbmd9PW5mcy13YzoyMDN8bmZzLWV1OjQ3fG5mcy1lYzotMwpwbGF5ZXJzLjAucHJvcHMue2ZpbHRlci12ZXJzaW9ufT0yOThfcHJvZF9zZXJ2ZXIrMjIwMTJiMTgKcGxheWVycy4wLnByb3BzLntmaWx0ZXJUb0dhbWUtdmVyc2lvbn09dmVyc2lvbgpwbGF5ZXJzLjAucHJvcHMue2ZpbHRlci1nYW1lX3R5cGV9PTB8MgpwbGF5ZXJzLjAucHJvcHMue2ZpbHRlclRvR2FtZS1nYW1lX3R5cGV9PVUtZ2FtZV90eXBlCnBsYXllcnMuMC5wcm9wcy57ZmlsdGVyLW1hdGNobWFraW5nX3N0YXRlfT0xCnBsYXllcnMuMC5wcm9wcy57ZmlsdGVyVG9HYW1lLW1hdGNobWFraW5nX3N0YXRlfT1VLW1hdGNobWFraW5nX3N0YXRlCnBsYXllcnMuMC5wcm9wcy57cHJlZi1jYXJfdGllcn09MwpwbGF5ZXJzLjAucHJvcHMue3ByZWZWb3RpbmdNZXRob2QtY2FyX3RpZXJ9PXBsdXJhbGl0eQpwbGF5ZXJzLjAucHJvcHMue2ZpdFZhbHVlcy1jYXJfdGllcn09IjEsMiwzLEFCU1RBSU4iCnBsYXllcnMuMC5wcm9wcy57Zml0VGFibGUtY2FyX3RpZXJ9PTE7MC41OzA7MXwwLjU7MTswLjU7MXwwOzAuNTsxOzF8MTsxOzE7MQpwbGF5ZXJzLjAucHJvcHMue2ZpdFdlaWdodC1jYXJfdGllcn09MTUwCnBsYXllcnMuMC5wcm9wcy57cHJlZlRvR2FtZS1jYXJfdGllcn09VS1jYXJfdGllcgpwbGF5ZXJzLjAucHJvcHMue3ByZWYtY29sbGlzaW9uX2RldGVjdGlvbn09MQpwbGF5ZXJzLjAucHJvcHMue3ByZWZWb3RpbmdNZXRob2QtY29sbGlzaW9uX2RldGVjdGlvbn09cGx1cmFsaXR5CnBsYXllcnMuMC5wcm9wcy57Zml0VmFsdWVzLWNvbGxpc2lvbl9kZXRlY3Rpb259PSIwLDEsQUJTVEFJTiIKcGxheWVycy4wLnByb3BzLntmaXRUYWJsZS1jb2xsaXNpb25fZGV0ZWN0aW9ufT0xOzA7MXwwOzE7MXwxOzE7MQpwbGF5ZXJzLjAucHJvcHMue2ZpdFdlaWdodC1jb2xsaXNpb25fZGV0ZWN0aW9ufT0yNQpwbGF5ZXJzLjAucHJvcHMue3ByZWZUb0dhbWUtY29sbGlzaW9uX2RldGVjdGlvbn09VS1jb2xsaXNpb25fZGV0ZWN0aW9uCnBsYXllcnMuMC5wcm9wcy57cHJlZi1nYW1lX21vZGV9PTEKcGxheWVycy4wLnByb3BzLntwcmVmVm90aW5nTWV0aG9kLWdhbWVfbW9kZX09cGx1cmFsaXR5CnBsYXllcnMu}
->Q res pnow 0xb0000019 {size=18728, data=MC5wcm9wcy57Zml0VmFsdWVzLWdhbWVfbW9kZX09IjEsMCw1LDEzLDE1LDE0LEFCU1RBSU4iCnBsYXllcnMuMC5wcm9wcy57Zml0VGFibGUtZ2FtZV9tb2RlfT0xOzAuNjswLjY7MC40OzAuNDswOzF8MC42OzE7MC42OzAuNDswLjQ7MDsxfDAuNjswLjY7MTswLjQ7MC40OzA7MXwwLjQ7MC40OzAuNDsxOzAuMjswOzF8MC40OzAuNDswLjQ7MC4yOzE7MC42OzF8MDswOzA7MDswLjY7MTsxfDE7MTsxOzE7MTsxOzEKcGxheWVycy4wLnByb3BzLntmaXRXZWlnaHQtZ2FtZV9tb2RlfT01MDAKcGxheWVycy4wLnByb3BzLntwcmVmVG9HYW1lLWdhbWVfbW9kZX09VS1nYW1lX21vZGUKcGxheWVycy4wLnByb3BzLntwcmVmLWhlbHBfdHlwZX09MgpwbGF5ZXJzLjAucHJvcHMue3ByZWZWb3RpbmdNZXRob2QtaGVscF90eXBlfT1wbHVyYWxpdHkKcGxheWVycy4wLnByb3BzLntmaXRWYWx1ZXMtaGVscF90eXBlfT0iMCwxLDIsMyIKcGxheWVycy4wLnByb3BzLntmaXRUYWJsZS1oZWxwX3R5cGV9PTAuODstMTstMTsxfDE7MC45Oy0xOzAuOXwxOy0xOzE7MXwxOzAuOTsxOzEKcGxheWVycy4wLnByb3BzLntmaXRXZWlnaHQtaGVscF90eXBlfT0xNDAwMApwbGF5ZXJzLjAucHJvcHMue3ByZWZUb0dhbWUtaGVscF90eXBlfT1VLWhlbHBfdHlwZQpwbGF5ZXJzLjAucHJvcHMue3ByZWYtbGVuZ3RofT0yCnBsYXllcnMuMC5wcm9wcy57cHJlZlZvdGluZ01ldGhvZC1sZW5ndGh9PXBsdXJhbGl0eQpwbGF5ZXJzLjAucHJvcHMue2ZpdFZhbHVlcy1sZW5ndGh9PSIxLDIsMyw0LEFCU1RBSU4iCnBsYXllcnMuMC5wcm9wcy57Zml0VGFibGUtbGVuZ3RofT0xOzAuNzU7MC4zOzA7MXwwLjc1OzE7MC43NTswLjM7MXwwLjM7MC43NTsxOzAuNzU7MXwwOzAuMzswLjc1OzE7MXwxOzE7MTsxOzEKcGxheWVycy4wLnByb3BzLntmaXRXZWlnaHQtbGVuZ3RofT0xMDAKcGxheWVycy4wLnByb3BzLntwcmVmVG9HYW1lLWxlbmd0aH09VS1sZW5ndGgKcGxheWVycy4wLnByb3BzLntwcmVmLWxvY2F0aW9ufT1uZnMtZWMKcGxheWVycy4wLnByb3BzLntwcmVmVm90aW5nTWV0aG9kLWxvY2F0aW9ufT1wbHVyYWxpdHkKcGxheWVycy4wLnByb3BzLntmaXRWYWx1ZXMtbG9jYXRpb259PSJuZnMtd2MsbmZzLWVjLG5mcy1ldSIKcGxheWVycy4wLnByb3BzLntmaXRUYWJsZS1sb2NhdGlvbn09MTswLjg7MC4yfDAuODsxOzAuNnwwLjI7MC42OzEKcGxheWVycy4wLnByb3BzLntmaXRXZWlnaHQtbG9jYXRpb259PTEwMDAKcGxheWVycy4wLnByb3BzLntwcmVmVG9HYW1lLWxvY2F0aW9ufT1VLWxvY2F0aW9uCnBsYXllcnMuMC5wcm9wcy57cHJlZi1uMm99PTEKcGxheWVycy4wLnByb3BzLntwcmVmVm90aW5nTWV0aG9kLW4yb309cGx1cmFsaXR5CnBsYXllcnMuMC5wcm9wcy57Zml0VmFsdWVzLW4yb309IjAsMSxBQlNUQUlOIgpwbGF5ZXJzLjAucHJvcHMue2ZpdFRhYmxlLW4yb309MTswOzF8MDsxOzF8MTsxOzEKcGxheWVycy4wLnByb3BzLntmaXRXZWlnaHQtbjJvfT0yNQpwbGF5ZXJzLjAucHJvcHMue3ByZWZU}
->Q res pnow 0xb0000019 {size=18728, data=b0dhbWUtbjJvfT1VLW4ybwpwbGF5ZXJzLjAucHJvcHMue3ByZWYtcmFjZV90eXBlX2Nhbnlvbl9kdWV9PUFCU1RBSU4KcGxheWVycy4wLnByb3BzLntwcmVmVm90aW5nTWV0aG9kLXJhY2VfdHlwZV9jYW55b25fZHVlfT1wbHVyYWxpdHkKcGxheWVycy4wLnByb3BzLntmaXRWYWx1ZXMtcmFjZV90eXBlX2Nhbnlvbl9kdWV9PSIyLmR1ZWwsMy5kdWVsLHFyLjMuMyxxci4zLjEsNS5kdWVsLDQuZHVlbCxxci4zLjIsQUJTVEFJTiIKcGxheWVycy4wLnByb3BzLntmaXRUYWJsZS1yYWNlX3R5cGVfY2FueW9uX2R1ZX09MTswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODsxOzAuODswLjg7MC44OzAuODswLjg7MXwwLjg7MC44OzE7MC44OzAuODswLjg7MC44OzF8MC44OzAuODswLjg7MTswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODsxOzAuODswLjg7MXwwLjg7MC44OzAuODswLjg7MC44OzE7MC44OzF8MC44OzAuODswLjg7MC44OzAuODswLjg7MTsxfDE7MTsxOzE7MTsxOzE7MQpwbGF5ZXJzLjAucHJvcHMue2ZpdFdlaWdodC1yYWNlX3R5cGVfY2FueW9uX2R1ZX09MTI1CnBsYXllcnMuMC5wcm9wcy57cHJlZlRvR2FtZS1yYWNlX3R5cGVfY2FueW9uX2R1ZX09VS1yYWNlX3R5cGVfY2FueW9uX2R1ZQpwbGF5ZXJzLjAucHJvcHMue3ByZWYtcmFjZV90eXBlX2NpcmN1aXR9PWV4LjUuMQpwbGF5ZXJzLjAucHJvcHMue3ByZWZWb3RpbmdNZXRob2QtcmFjZV90eXBlX2NpcmN1aXR9PXBsdXJhbGl0eQpwbGF5ZXJzLjAucHJvcHMue2ZpdFZhbHVlcy1yYWNlX3R5cGVfY2lyY3VpdH09ImV4LjUuMSxzZi4yLjEsdG4uMy4xLG11LjIuMyxjdC4xLjEsY2UuMy4zLHRuLjIuMSxjcy44LjEsY3QuMy4xLGV4LjIuMSxjZS4zLjIsY3MuOC4zLGNzLjguMixjdC4yLjEsZXguMS4xLHRuLjEuMSxleC40LjEsb25saW5lLG11LjMuMSxjdC43LjMsY3QuNS4zLHRuLjUuMSxjZS4zLjEsbXUuMS4xLEFCU1RBSU4iCnBsYXllcnMuMC5wcm9wcy57Zml0VGFibGUtcmFjZV90eXBlX2NpcmN1aXR9PTE7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODsxOzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MTswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzE7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODsxOzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MTsw}
->Q res pnow 0xb0000019 {size=18728, data=Ljg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzE7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxOzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MTswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzE7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxOzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MTswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzE7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxOzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MTswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzE7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxOzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MTswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzE7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxOzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MTswLjg7MC44OzAuODsx}
->Q res pnow 0xb0000019 {size=18728, data=fDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzE7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxOzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MTsxfDE7MTsxOzE7MTsxOzE7MTsxOzE7MTsxOzE7MTsxOzE7MTsxOzE7MTsxOzE7MTsxOzEKcGxheWVycy4wLnByb3BzLntmaXRXZWlnaHQtcmFjZV90eXBlX2NpcmN1aXR9PTEyNQpwbGF5ZXJzLjAucHJvcHMue3ByZWZUb0dhbWUtcmFjZV90eXBlX2NpcmN1aXR9PVUtcmFjZV90eXBlX2NpcmN1aXQKcGxheWVycy4wLnByb3BzLntwcmVmLXJhY2VfdHlwZV9rbm9ja291dH09QUJTVEFJTgpwbGF5ZXJzLjAucHJvcHMue3ByZWZWb3RpbmdNZXRob2QtcmFjZV90eXBlX2tub2Nrb3V0fT1wbHVyYWxpdHkKcGxheWVycy4wLnByb3BzLntmaXRWYWx1ZXMtcmFjZV90eXBlX2tub2Nrb3V0fT0icXIuNS4zLHFyLjUuNCxxci41LjIsa28ucHJvdG8scXIuNS41LHFyLjUuMSxBQlNUQUlOIgpwbGF5ZXJzLjAucHJvcHMue2ZpdFRhYmxlLXJhY2VfdHlwZV9rbm9ja291dH09MTswLjg7MC44OzAuODswLjg7MC44OzF8MC44OzE7MC44OzAuODswLjg7MC44OzF8MC44OzAuODsxOzAuODswLjg7MC44OzF8MC44OzAuODswLjg7MTswLjg7MC44OzF8MC44OzAuODswLjg7MC44OzE7MC44OzF8MC44OzAuODswLjg7MC44OzAuODsxOzF8MTsxOzE7MTsxOzE7MQpwbGF5ZXJzLjAucHJvcHMue2ZpdFdlaWdodC1yYWNlX3R5cGVfa25vY2tvdXR9PTEyNQpwbGF5ZXJzLjAucHJvcHMue3ByZWZUb0dhbWUtcmFjZV90eXBlX2tub2Nrb3V0fT1VLXJhY2VfdHlwZV9rbm9ja291dApwbGF5ZXJzLjAucHJvcHMue3ByZWYtcmFjZV90eXBlX3B1cnN1aXRfdGFnfT1BQlNUQUlOCnBsYXllcnMuMC5wcm9wcy57cHJlZlZvdGluZ01ldGhvZC1yYWNlX3R5cGVfcHVyc3VpdF90YWd9PXBsdXJhbGl0eQpwbGF5ZXJzLjAucHJvcHMue2ZpdFZhbHVlcy1yYWNlX3R5cGVfcHVyc3VpdF90YWd9PSJxci42LjIscXIuNi40LHFyLjYuNSxxci42LjEsdGFnLnByb3RvLHFyLjYuMyxBQlNUQUlOIgpwbGF5ZXJzLjAucHJvcHMue2ZpdFRhYmxlLXJhY2VfdHlwZV9wdXJzdWl0X3RhZ309MTswLjg7MC44OzAuODswLjg7MC44OzF8MC44OzE7MC44OzAuODswLjg7MC44OzF8MC44OzAuODsxOzAuODswLjg7MC44OzF8MC44OzAuODswLjg7MTswLjg7MC44OzF8MC44OzAuODswLjg7MC44OzE7MC44OzF8MC44OzAuODswLjg7MC44OzAuODsxOzF8MTsxOzE7MTsxOzE7MQpwbGF5ZXJzLjAucHJvcHMue2ZpdFdlaWdodC1yYWNlX3R5cGVfcHVyc3VpdF90YWd9PTEyNQpwbGF5ZXJzLjAucHJvcHMue3ByZWZUb0dhbWUtcmFj}
->Q res pnow 0xb0000019 {size=18728, data=ZV90eXBlX3B1cnN1aXRfdGFnfT1VLXJhY2VfdHlwZV9wdXJzdWl0X3RhZwpwbGF5ZXJzLjAucHJvcHMue3ByZWYtcmFjZV90eXBlX3NwZWVkdHJhcH09QUJTVEFJTgpwbGF5ZXJzLjAucHJvcHMue3ByZWZWb3RpbmdNZXRob2QtcmFjZV90eXBlX3NwZWVkdHJhcH09cGx1cmFsaXR5CnBsYXllcnMuMC5wcm9wcy57Zml0VmFsdWVzLXJhY2VfdHlwZV9zcGVlZHRyYXB9PSJtdS4zLjMsY3QuNC4zLGNzLjExLjIsZXguMi4zLG11LjUuMSxjdC41LjEsdG4uNC4yLGN0LjMuMixtdS4yLjIsdG4uMy4zLGNzLjExLjMsY3MuMTEuMSxtdS40LjIsQUJTVEFJTiIKcGxheWVycy4wLnByb3BzLntmaXRUYWJsZS1yYWNlX3R5cGVfc3BlZWR0cmFwfT0xOzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzF8MC44OzE7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MTswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MXwwLjg7MC44OzAuODsxOzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzF8MC44OzAuODswLjg7MC44OzE7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MTswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MXwwLjg7MC44OzAuODswLjg7MC44OzAuODsxOzAuODswLjg7MC44OzAuODswLjg7MC44OzF8MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzE7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MTswLjg7MC44OzAuODswLjg7MXwwLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxOzAuODswLjg7MC44OzF8MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzE7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MTswLjg7MXwwLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxOzF8MTsxOzE7MTsxOzE7MTsxOzE7MTsxOzE7MTsxCnBsYXllcnMuMC5wcm9wcy57Zml0V2VpZ2h0LXJhY2VfdHlwZV9zcGVlZHRyYXB9PTEyNQpwbGF5ZXJzLjAucHJvcHMue3ByZWZUb0dhbWUtcmFjZV90eXBlX3NwZWVkdHJhcH09VS1yYWNlX3R5cGVfc3BlZWR0cmFwCnBsYXllcnMuMC5wcm9wcy57cHJlZi1yYWNlX3R5cGVfc3ByaW50fT1BQlNUQUlOCnBsYXllcnMuMC5wcm9wcy57cHJlZlZvdGluZ01ldGhvZC1yYWNlX3R5cGVfc3ByaW50fT1wbHVyYWxpdHkKcGxheWVycy4wLnByb3BzLntmaXRWYWx1ZXMtcmFjZV90eXBlX3NwcmludH09ImV4LjQuMixjdC41LjIsY3MuOS4xLG11LjUuMixleC4zLjMsdG4uMi4zLHFyLjQuMSxtdS4yLjEsY2UuMi4xLGNzLjIuMixjZS4yLjIscXIuNC42LHFyLjQuNCxjcy4yLjEscXIuNC4zLGN0LjEuMyxjcy4yLjMsY3QuNi4xLGNzLjkuMyxleC41LjIsY3QuMi4zLGN0LjQuMixleC4yLjIsdG4uNC4xLHRu}
->Q res pnow 0xb0000019 {size=18728, data=LjEuMyxleC4zLjEscXIuNC4yLG11LjEuMix0bi4zLjIsY3MuOS4yLG11LjMuMixBQlNUQUlOIgpwbGF5ZXJzLjAucHJvcHMue2ZpdFRhYmxlLXJhY2VfdHlwZV9zcHJpbnR9PTE7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MXwwLjg7MTswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzF8MC44OzAuODsxOzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzE7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MXwwLjg7MC44OzAuODswLjg7MTswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzF8MC44OzAuODswLjg7MC44OzAuODsxOzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzE7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MXwwLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MTswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzF8MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxOzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzE7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MXwwLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MTswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzF8MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxOzAu}
->Q res pnow 0xb0000019 {size=18728, data=ODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzE7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MXwwLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MTswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzF8MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxOzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzE7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MXwwLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MTswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzF8MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxOzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzE7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MXwwLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MTswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzF8MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxOzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzE7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MXwwLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MTswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzF8MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7}
->Q res pnow 0xb0000019 {size=18728, data=MC44OzAuODswLjg7MC44OzAuODsxOzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzE7MC44OzAuODswLjg7MC44OzAuODswLjg7MXwwLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MTswLjg7MC44OzAuODswLjg7MC44OzF8MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxOzAuODswLjg7MC44OzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzE7MC44OzAuODswLjg7MXwwLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MTswLjg7MC44OzF8MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODsxOzAuODsxfDAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzAuODswLjg7MC44OzE7MXwxOzE7MTsxOzE7MTsxOzE7MTsxOzE7MTsxOzE7MTsxOzE7MTsxOzE7MTsxOzE7MTsxOzE7MTsxOzE7MTsxOzEKcGxheWVycy4wLnByb3BzLntmaXRXZWlnaHQtcmFjZV90eXBlX3NwcmludH09MTI1CnBsYXllcnMuMC5wcm9wcy57cHJlZlRvR2FtZS1yYWNlX3R5cGVfc3ByaW50fT1VLXJhY2VfdHlwZV9zcHJpbnQKcGxheWVycy4wLnByb3BzLntwcmVmLXRlYW1fcGxheX09MQpwbGF5ZXJzLjAucHJvcHMue3ByZWZWb3RpbmdNZXRob2QtdGVhbV9wbGF5fT1wbHVyYWxpdHkKcGxheWVycy4wLnByb3BzLntmaXRWYWx1ZXMtdGVhbV9wbGF5fT0iMCwxLEFCU1RBSU4iCnBsYXllcnMuMC5wcm9wcy57Zml0VGFibGUtdGVhbV9wbGF5fT0tMTstMTstMXwtMTsxOzF8LTE7MTsxCnBsYXllcnMuMC5wcm9wcy57Zml0V2VpZ2h0LXRlYW1fcGxheX09NTAwCnBsYXllcnMuMC5wcm9wcy57cHJlZlRvR2FtZS10ZWFtX3BsYXl9PVUtdGVhbV9wbGF5CnBsYXllcnMuMC5wcm9wcy57cHJlZi1tYXhfb25saW5lX3BsYXllcn09OApwbGF5ZXJzLjAucHJvcHMue2ZpdFNjYWxlLW1heF9vbmxpbmVfcGxheWVyfT01CnBsYXllcnMuMC5wcm9wcy57Zml0V2VpZ2h0LW1h}
->Q res pnow 0xb0000019 {size=18728, data=eF9vbmxpbmVfcGxheWVyfT03NQpwbGF5ZXJzLjAucHJvcHMue3ByZWZUb0dhbWUtbWF4X29ubGluZV9wbGF5ZXJ9PVUtbWF4X29ubGluZV9wbGF5ZXIKcGxheWVycy4wLnByb3BzLntwcmVmLXBsYXllcl9kbmZ9PTI1CnBsYXllcnMuMC5wcm9wcy57Zml0U2NhbGUtcGxheWVyX2RuZn09NApwbGF5ZXJzLjAucHJvcHMue2ZpdFdlaWdodC1wbGF5ZXJfZG5mfT01MDAKcGxheWVycy4wLnByb3BzLntwcmVmVG9HYW1lLXBsYXllcl9kbmZ9PVUtcGxheWVyX2RuZgpwbGF5ZXJzLjAucHJvcHMue3ByZWYtc2tpbGx9PTEwMDAKcGxheWVycy4wLnByb3BzLntmaXRTY2FsZS1za2lsbH09NzAwCnBsYXllcnMuMC5wcm9wcy57Zml0V2VpZ2h0LXNraWxsfT0zMDAwCnBsYXllcnMuMC5wcm9wcy57cHJlZlRvR2FtZS1za2lsbH09VS1za2lsbApwbGF5ZXJzLjAucHJvcHMue309MTEwCg%3d%3d}
```
Right, I forgot to mention that in case with big packets FESL protocol will only state the size and then send the payload as base64 packed data, my bad! Let's de-base64 the data, shall we?

```
TXN=Start
partition.partition=/EAGAMES/NFS-2007
debugLevel=off
players.[]=1
players.0.ownerId=907370134
players.0.ownerType=1
players.0.props.{sessionType}=findServer
players.0.props.{name}=xan1242
players.0.props.{firewallType}=unknown
players.0.props.{poolMaxPlayers}=8
players.0.props.{poolTimeout}=20
players.0.props.{poolTargetPlayers}=0:1
players.0.props.{fitThreshold}=0:5400
players.0.props.{ping}=nfs-wc:203|nfs-eu:47|nfs-ec:-3
players.0.props.{filter-version}=298_prod_server+22012b18
players.0.props.{filterToGame-version}=version
players.0.props.{filter-game_type}=0|2
players.0.props.{filterToGame-game_type}=U-game_type
players.0.props.{filter-matchmaking_state}=1
players.0.props.{filterToGame-matchmaking_state}=U-matchmaking_state
players.0.props.{pref-car_tier}=3
players.0.props.{prefVotingMethod-car_tier}=plurality
players.0.props.{fitValues-car_tier}="1,2,3,ABSTAIN"
players.0.props.{fitTable-car_tier}=1;0.5;0;1|0.5;1;0.5;1|0;0.5;1;1|1;1;1;1
players.0.props.{fitWeight-car_tier}=150
players.0.props.{prefToGame-car_tier}=U-car_tier
players.0.props.{pref-collision_detection}=1
players.0.props.{prefVotingMethod-collision_detection}=plurality
players.0.props.{fitValues-collision_detection}="0,1,ABSTAIN"
players.0.props.{fitTable-collision_detection}=1;0;1|0;1;1|1;1;1
players.0.props.{fitWeight-collision_detection}=25
players.0.props.{prefToGame-collision_detection}=U-collision_detection
players.0.props.{pref-game_mode}=1
players.0.props.{prefVotingMethod-game_mode}=plurality
players.0.props.{fitValues-game_mode}="1,0,5,13,15,14,ABSTAIN"
players.0.props.{fitTable-game_mode}=1;0.6;0.6;0.4;0.4;0;1|0.6;1;0.6;0.4;0.4;0;1|0.6;0.6;1;0.4;0.4;0;1|0.4;0.4;0.4;1;0.2;0;1|0.4;0.4;0.4;0.2;1;0.6;1|0;0;0;0;0.6;1;1|1;1;1;1;1;1;1
players.0.props.{fitWeight-game_mode}=500
players.0.props.{prefToGame-game_mode}=U-game_mode
players.0.props.{pref-help_type}=2
players.0.props.{prefVotingMethod-help_type}=plurality
players.0.props.{fitValues-help_type}="0,1,2,3"
players.0.props.{fitTable-help_type}=0.8;-1;-1;1|1;0.9;-1;0.9|1;-1;1;1|1;0.9;1;1
players.0.props.{fitWeight-help_type}=14000
players.0.props.{prefToGame-help_type}=U-help_type
players.0.props.{pref-length}=2
players.0.props.{prefVotingMethod-length}=plurality
players.0.props.{fitValues-length}="1,2,3,4,ABSTAIN"
players.0.props.{fitTable-length}=1;0.75;0.3;0;1|0.75;1;0.75;0.3;1|0.3;0.75;1;0.75;1|0;0.3;0.75;1;1|1;1;1;1;1
players.0.props.{fitWeight-length}=100
players.0.props.{prefToGame-length}=U-length
players.0.props.{pref-location}=nfs-ec
players.0.props.{prefVotingMethod-location}=plurality
players.0.props.{fitValues-location}="nfs-wc,nfs-ec,nfs-eu"
players.0.props.{fitTable-location}=1;0.8;0.2|0.8;1;0.6|0.2;0.6;1
players.0.props.{fitWeight-location}=1000
players.0.props.{prefToGame-location}=U-location
players.0.props.{pref-n2o}=1
players.0.props.{prefVotingMethod-n2o}=plurality
players.0.props.{fitValues-n2o}="0,1,ABSTAIN"
players.0.props.{fitTable-n2o}=1;0;1|0;1;1|1;1;1
players.0.props.{fitWeight-n2o}=25
players.0.props.{prefToGame-n2o}=U-n2o
players.0.props.{pref-race_type_canyon_due}=ABSTAIN
players.0.props.{prefVotingMethod-race_type_canyon_due}=plurality
players.0.props.{fitValues-race_type_canyon_due}="2.duel,3.duel,qr.3.3,qr.3.1,5.duel,4.duel,qr.3.2,ABSTAIN"
players.0.props.{fitTable-race_type_canyon_due}=1;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;1;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;1;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;1;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;1;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;1;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;1;1|1;1;1;1;1;1;1;1
players.0.props.{fitWeight-race_type_canyon_due}=125
players.0.props.{prefToGame-race_type_canyon_due}=U-race_type_canyon_due
players.0.props.{pref-race_type_circuit}=ex.5.1
players.0.props.{prefVotingMethod-race_type_circuit}=plurality
players.0.props.{fitValues-race_type_circuit}="ex.5.1,sf.2.1,tn.3.1,mu.2.3,ct.1.1,ce.3.3,tn.2.1,cs.8.1,ct.3.1,ex.2.1,ce.3.2,cs.8.3,cs.8.2,ct.2.1,ex.1.1,tn.1.1,ex.4.1,online,mu.3.1,ct.7.3,ct.5.3,tn.5.1,ce.3.1,mu.1.1,ABSTAIN"
players.0.props.{fitTable-race_type_circuit}=1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;1|1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1
players.0.props.{fitWeight-race_type_circuit}=125
players.0.props.{prefToGame-race_type_circuit}=U-race_type_circuit
players.0.props.{pref-race_type_knockout}=ABSTAIN
players.0.props.{prefVotingMethod-race_type_knockout}=plurality
players.0.props.{fitValues-race_type_knockout}="qr.5.3,qr.5.4,qr.5.2,ko.proto,qr.5.5,qr.5.1,ABSTAIN"
players.0.props.{fitTable-race_type_knockout}=1;0.8;0.8;0.8;0.8;0.8;1|0.8;1;0.8;0.8;0.8;0.8;1|0.8;0.8;1;0.8;0.8;0.8;1|0.8;0.8;0.8;1;0.8;0.8;1|0.8;0.8;0.8;0.8;1;0.8;1|0.8;0.8;0.8;0.8;0.8;1;1|1;1;1;1;1;1;1
players.0.props.{fitWeight-race_type_knockout}=125
players.0.props.{prefToGame-race_type_knockout}=U-race_type_knockout
players.0.props.{pref-race_type_pursuit_tag}=ABSTAIN
players.0.props.{prefVotingMethod-race_type_pursuit_tag}=plurality
players.0.props.{fitValues-race_type_pursuit_tag}="qr.6.2,qr.6.4,qr.6.5,qr.6.1,tag.proto,qr.6.3,ABSTAIN"
players.0.props.{fitTable-race_type_pursuit_tag}=1;0.8;0.8;0.8;0.8;0.8;1|0.8;1;0.8;0.8;0.8;0.8;1|0.8;0.8;1;0.8;0.8;0.8;1|0.8;0.8;0.8;1;0.8;0.8;1|0.8;0.8;0.8;0.8;1;0.8;1|0.8;0.8;0.8;0.8;0.8;1;1|1;1;1;1;1;1;1
players.0.props.{fitWeight-race_type_pursuit_tag}=125
players.0.props.{prefToGame-race_type_pursuit_tag}=U-race_type_pursuit_tag
players.0.props.{pref-race_type_speedtrap}=ABSTAIN
players.0.props.{prefVotingMethod-race_type_speedtrap}=plurality
players.0.props.{fitValues-race_type_speedtrap}="mu.3.3,ct.4.3,cs.11.2,ex.2.3,mu.5.1,ct.5.1,tn.4.2,ct.3.2,mu.2.2,tn.3.3,cs.11.3,cs.11.1,mu.4.2,ABSTAIN"
players.0.props.{fitTable-race_type_speedtrap}=1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;1|1;1;1;1;1;1;1;1;1;1;1;1;1;1
players.0.props.{fitWeight-race_type_speedtrap}=125
players.0.props.{prefToGame-race_type_speedtrap}=U-race_type_speedtrap
players.0.props.{pref-race_type_sprint}=ABSTAIN
players.0.props.{prefVotingMethod-race_type_sprint}=plurality
players.0.props.{fitValues-race_type_sprint}="ex.4.2,ct.5.2,cs.9.1,mu.5.2,ex.3.3,tn.2.3,qr.4.1,mu.2.1,ce.2.1,cs.2.2,ce.2.2,qr.4.6,qr.4.4,cs.2.1,qr.4.3,ct.1.3,cs.2.3,ct.6.1,cs.9.3,ex.5.2,ct.2.3,ct.4.2,ex.2.2,tn.4.1,tn.1.3,ex.3.1,qr.4.2,mu.1.2,tn.3.2,cs.9.2,mu.3.2,ABSTAIN"
players.0.props.{fitTable-race_type_sprint}=1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;0.8;1|0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;0.8;1;1|1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1;1
players.0.props.{fitWeight-race_type_sprint}=125
players.0.props.{prefToGame-race_type_sprint}=U-race_type_sprint
players.0.props.{pref-team_play}=1
players.0.props.{prefVotingMethod-team_play}=plurality
players.0.props.{fitValues-team_play}="0,1,ABSTAIN"
players.0.props.{fitTable-team_play}=-1;-1;-1|-1;1;1|-1;1;1
players.0.props.{fitWeight-team_play}=500
players.0.props.{prefToGame-team_play}=U-team_play
players.0.props.{pref-max_online_player}=8
players.0.props.{fitScale-max_online_player}=5
players.0.props.{fitWeight-max_online_player}=75
players.0.props.{prefToGame-max_online_player}=U-max_online_player
players.0.props.{pref-player_dnf}=25
players.0.props.{fitScale-player_dnf}=4
players.0.props.{fitWeight-player_dnf}=500
players.0.props.{prefToGame-player_dnf}=U-player_dnf
players.0.props.{pref-skill}=1000
players.0.props.{fitScale-skill}=700
players.0.props.{fitWeight-skill}=3000
players.0.props.{prefToGame-skill}=U-skill
players.0.props.{}=110
```
110 parameters to define how to match 1 player. Amazing!

Anyway, the PlayNow takes this monstrosity of a request and match a player to a server based on this request. Or set an existing dedicated server according to the request and connect player there. Since this task took a while back then, this request is "async". First, server reply with:
```
<-N res pnow 0x80000019 {TXN=Start, id.id=2516, id.partition=/eagames/NFS-2007}
```
and when sessions is found it sends out-of-order packet

```
<-A res pnow 0x80000000 {sessionState=COMPLETE, props.{avgFit}=0.8642424242424243, props.{games}.[]=1, props.{games}.0.gid=235234, TXN=Status, props.{}=3, props.{games}.0.lid=257, props.{resultType}=JOIN, id.id=2516, id.partition=/eagames/NFS-2007}
```
While several result types are supported (`CREATE`, `CANCEL`, etc) mostly none of them, other than `JOIN` is supported by the client. 

After client get a "GameId" (`gid`) it makes a request to Theater to get all the necessary information to perform the connection to the session

```
->N GDAT {LID=257, GID=235234, TID=6}
<-N GDAT {JP=1, HN=nfsdevserver, B-U-matchmaking_state=0, N=NA1CarbonPC-02-024, B-U-team_play=1, I=159.153.43.197, J=O, HU=799270239, B-U-car_tier=3, V=1.0, P=19023, B-U-game_mode=1, TYPE=G, LID=257, B-U-help_type=0, B-U-player_dnf=25, B-version=298_prod_server+22012b18, B-U-max_online_player=8, B-U-n2o=1, B-U-track=, QP=0, MP=8, B-U-collision_detection=1, B-U-version=298_prod_server+22012b18, B-U-race_type_sprint=mu.5.2, B-U-race_type_pursuit_tag=qr.6.2, GID=235234, PL=PC, PW=0, TID=6, B-U-race_type_speedtrap=ct.4.3, B-U-skill=1000, B-U-game_type=1, B-U-race_type_canyon_due=2.duel, B-U-race_type_circuit=cs.8.1, AP=0, B-U-race_type_knockout=qr.5.5, B-U-length=2}
->N EGAM {PORT=1042, R-INT-PORT=1042, R-INT-IP=192.168.5.113, LID=257, GID=235234, TID=7}
<-U GDET {TID=6, UGID=775a990f-1cef-4230-8e75-63de83b18396, LID=257, GID=235234}
<-N EGAM {TID=7, LID=257, GID=235234}
<-A EGEG {PL=pc, TICKET=1172681271, PID=19, P=19023, HUID=799270239, INT-PORT=19023, EKEY=Fpj/Ob0gskZs8P2b8SsWaw%3d%3d, INT-IP=159.153.43.197, UGID=775a990f-1cef-4230-8e75-63de83b18396, I=159.153.43.197, LID=257, GID=235234}

```
After that the connection happening and everyone is happy and can enjoy their favorite game online.

## Afterword

Beside the described above, FESL also does a handful of other things, like, serving content, keep track of rank and others. But for a revival and preservation of the games, most of those could be neglected.

If we were asked to rate FESL we would give it a solid 3/5. Not good, not terrible. Approachable protocol with too much bloat on the sides. And no P2P mode in most titles. 