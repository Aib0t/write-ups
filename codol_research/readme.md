> A doc I made about CODOL back in 2025. Nothing major had happen since. To preserve the doc I decided to put it here.

Call Of Duty Online (or CODOL for short) is a chinese market iw4 spinoff made by tencent and activision. Unlike its big counterpart of mw2, it features cosmetics, unlocks and other Chinese flavored stuff.

### Current state of things and updates
Non-operation as of ~~25/4/2025~~ 02/02/2026.

### Main vectors of research
There are 2 main directions were revival is going:
- Restoration of QA build 3.3.3.3 without tencent protect and VMProtect
- Restoration of 4D1 build, based on OpenIW4

## QA build 3.3.3.3
Thanks to a leak there are 2 publicly available binaries without tenprotect (and VMP virtualization) with their respective files. Build 3.3.3.3 is the most interesting one, being late in CODOL life and the one actually launchable, though a binary called “CODO_OfflineLauncher.exe”.

Despite being launchable and backend being emulate-able the game lacks server binaries, required in initial auth flow. In case of initial auth bypass, the game still can’t be played, due to lack of the servers.

## 4D1 build
Based on OpenIW4, some 4D1 tech and CODOL files, this build was actually playable in the past and featured some game modes and content from the original game. Currently, work is being done on making it playable, to bring at least some features of the game back from the grave.

TODO: add the actual state of the project and roadblocks.

# Technical info

## QA build 3.3.3.3

### Build links

Most of the builds, including 3.3.3.3 are located at rains (thanks rain!) mega https://mega.nz/folder/SfglgLyD#VnmwxC4zAuMZ1dsh9gkRIA/folder/KbIS2LwC

Build 3.3.3.3 - https://mega.nz/folder/SfglgLyD#VnmwxC4zAuMZ1dsh9gkRIA/folder/6aIgFDRC

### Brief analysis - the binary
Build 3.3.3.3 comes with an unprotected QA binary, which is processable with IDA/Ghidra. Some logging/debug strings are available, easing the analysis a bit. Most of the calls are lacking references to them, making static analysis a bit of a pain.

There is minor anti-debugger protection in case of dynamic analysis, but those are bypassable with basic settings of ScyllaHide (a plugin for x32dbg).

Memory location is also shifted, which is a minor issue for a researcher.

### Brief analysis - the backend
COBOL is using what we call a “Demonware 3”, which is internally called version 2.1.0. Activision began using this version starting from around 2014 in its titles. For now I don’t have any concrete evidence if DW3 was made specifically for COBOL and later repurposed for all the titles, or if it was the other way around. But COBOL is certainly one of the earliest titles to feature this version.

DW3 utilizes http(s) auth server and raw tcp master server. Additional to this a stun UDP service is used, but can be mostly neglected due to limited usage in DW configuration with dedicated servers.

There is one public implementation of DW3 server, called s1x, available here - https://github.com/CBServers/s1x-client

There are also other derivatives of this code for later COD games, but those could be found via github search.
Launching the build

To launch the build you either needs “CODO_OfflineLauncher.exe” or call the binary with the arguments extracted from this launcher

`“codoMP_client_shipRetail.exe"  -q 0 -src=tgp -game_id 29 -zone_id 83886082`

After that you would be presented with the game window, which is mostly useless without a working backend.

## Backend
To properly construct a backend for COBOL one would need to take s1x code, move it away from the dll, and make 2 dedicated servers - Auth and LSG.

DW 3 was Demonware's attempt to move away from outdated DW2 crypto and auth, to bring it into the modern age of consoles. Funny enough that’s where the difference ends, since the LSG part is mostly the same as its previous generation. For reference one can use leaked COD BO2 server binaries with PDB from https://mega.nz/file/VUVATCDK#aHbD69iprnTDgvrvwTQD_LZ9fCWUZ06zltZkNhULT1M

Auth
Auth is done via https with json payloads. Client would send an auth task with all clients data, and in return server sends 2 tickets - auth ticket and lsg ticket, named client_ticket and server_ticket in the payload.
```
	// Request
	// {
	// 	"auth_task": "XX",
	// 	"iv_seed": "1170742737",
	// 	"title_id": "1002",
	// 	"extra_data": {
	//     	"version": "32",
	//     	"token": {
	//         	"scope": "openid,openid:duid",
	//         	"id_token": "base64-encoded-payload"
	//     	},
	//     	"online_id": "aibot",
	//     	"extended_data": true
	// 	}
	// }

	// Response
	// {
	// 	"auth_task": "XX",
	// 	"code": "700",
	// 	"iv_seed": "2108419525",
	// 	"client_ticket": "base64-encoded-ticket",
	// 	"server_ticket": "base64-encoded-ticket",
	// 	"client_id": "",
	// 	"account_type": "psn",
	// 	"crossplay_enabled": false,
	// 	"loginqueue_enabled": false,
	// 	"lsg_endpoint": null,
	// 	"extra_data": {
	//     	"extended_data": "%data%"
	// 	}
	// }
```
Code 700 is indicating the success in reply.

In case the client can successfully decrypt 3des encrypted auth ticket (client_ticket) the auth proceeds to LSG auth.

Decryption is happening in 
```
Address=0079C3F0
Module/Label/Exception=<codomp_client_shipretail.exe.3desDecryption>
```
And
```
Address=006C9C01
Module/Label/Exception=<codomp_client_shipretail.exe.TicketDecryptor>
```
Where a known to client key and iv from the auth response are supplied. It’s not known if the key is static or generated from clients data. For simplicity it’s now extracted from this function and applied on the Auth server side.

Some extra functions from the auth flow:
```
Address=006C7CD0
Module/Label/Exception=<codomp_client_shipretail.exe.DWAuthTicketParser>

Address=006C8730
Module/Label/Exception=<codomp_client_shipretail.exe.ReadAuthReplyDW3_1>

Address=006C98B0
Module/Label/Exception=<codomp_client_shipretail.exe.ReadDWAuthTicket_2>
```
### LSG

Client sends towards the client the LSG ticket. The auth flow itself can be studied here https://github.com/CBServers/s1x-client/blob/main/src/client/game/demonware/servers/lobby_server.cpp#L38

Client makes a total of 39 calls to the lsg during initial flow
```
Storage2(Storage2ServiceType::UploadFile10),
Storage2(Storage2ServiceType::GetFile12),
Storage2(Storage2ServiceType::GetPublisherFile21),
Storage2(Storage2ServiceType::ListAllPublisherFiles20),
TitleUtilities2(TitleUtilities2ServiceType::VerifyString),
BandwidthTest,
ContentUnlock2(ContentUnlock2ServiceType::Unknown1),
ContentUnlock2(ContentUnlock2ServiceType::Unknown2),
Twitter(TwitterServiceType::Search),
Twitter(TwitterServiceType::Unk70),
Tencent(TencentServiceType::unknown_3),
Tencent(TencentServiceType::unknown_4),
Tencent(TencentServiceType::unknown_5),
Tencent(TencentServiceType::unknown_7),
Tencent(TencentServiceType::unknown_9),
Tencent(TencentServiceType::unknown_10),
Tencent(TencentServiceType::unknown_11),
Tencent(TencentServiceType::TelemetryCompressed),
Marketplace(MarketplaceServiceType::GetBalance),
Marketplace(MarketplaceServiceType::Unk70),
Marketplace(MarketplaceServiceType::Unk107),
Marketplace(MarketplaceServiceType::Unk173),
Marketplace(MarketplaceServiceType::Unk174),
Marketplace(MarketplaceServiceType::Unk230),
TitleUtilities2(TitleUtilities2ServiceType::GetServerTime),
Messaging(MessagingServiceType::GetNotifications),
Profiles2(Profiles2ServiceType::GetPublicInfos),
Profiles2(Profiles2ServiceType::SetPublicInfo),
Profiles2(Profiles2ServiceType::GetPrivateInfo),
Unk93(Unk93ServiceType::Unk1),
Friends2(Friends2ServiceType::GetIncomingProposals),
Friends2(Friends2ServiceType::GetOutgoingProposals),
Friends2(Friends2ServiceType::SetRichPresence),
Friends2(Friends2ServiceType::GetGroupNames),
Teams(TeamsServiceType::Unk51),
Teams(TeamsServiceType::Unk53),
Teams(TeamsServiceType::Unk46),
Teams(TeamsServiceType::Unk48),
Unk29(Unk29ServiceType::Unk10),
Unk102(Unk102ServiceType::Unk1),
Matchmaking2(Matchmaking2ServiceType::FindSessions),
RichPresence(RichPresenceServiceType::SetInfo)
```
### LSG - GetPublisherFile
While 38 out of 39 of requests can be dummied out (https://github.com/CBServers/s1x-client/blob/main/src/client/game/demonware/services/bdGroups.cpp#L12) this calls needs to return a set of files

```
activities_v1.info
broadcast.info
configs_s.info
market-down.info
mp_playlists_shipRetail_alpha2.info
sku_tags.info
tlogconfig.info
whitelist_tester.info
```

All of them could be empty, but they must be sent towards the client.

Those files and their fields could be reconstructed from the code to some degree, but since they are crucial to the game, without the real files it’s rather hard to say if the game will function, if at all.

### Post initial LSG auth

Jumping through all of those loops you would get to the operator picking screen, and then get stuck on a screen with a button, with no ability to proceed.

This button sends a request “Matchmaking::GetSessions”. Game requests a session, and we, sadly, don't have any to return. Judging by telemetry data, the game tries to find an empty server to put us into an “onboarding mission”. And since there are no servers, and no leaked COBOL server, proceeding further is not possible.


This is where we can take another approach with “reply with players data” to the game, to trick it into thinking we’re not a new player.

But after that we’re met with more calls to the master server. And no ability to play anything, since there are no server binaries.

## Client setup for backend connection
To properly redirect game somewhere you would need several files

```
cacert.pem
dwauth.ini
stun.ini
OpenID.txt (Seem optional, but I used it, so I will include it here)
```

### dwauth.ini

Is the main file for redirection, with a content of
```
127.0.0.1 3074
127.0.0.1 443
0x03EA
```

First record being master server, second being auth server and the last one being game port, which the game needs to check when doing a NAT check via STUN servers.

### stun.ini

This file controls domains for DW specific STUN server. Since official stuns for COBOL are long gone, an official, still functioning STUN will work.
```
stun.us.demonware.net
```

### OpenID.txt

I borrowed this file from another COBOL build and mine has a content of
```
F99BB9DC143845921B548228B2CE8760
```
### cacert.pem

This is an encoded CA cert for auth server file cert.

## 4D1 Build
TODO: fill info about this build and its state, since I personally lack any

## Other notable builds
Another build without tenprotect is Call of Duty Online [PTS Build 1.1.1.14]. Build itself is not compatible with codoMP_client_shipRetail.exe, probably to an incorrect zone_id being passed. 

Retail builds, tenprotect and launcher
The rest of the available builds are protected with tenprotect and virtualized with an old version of VMProtect. While de-virtualization of the binaries is possible to some selected few via enormous effort, tenprotect is playing a big role in making usage of those builds viable.

Potentially, a dummy dll can be created via RE and tools like http://www.rohitab.com/apimonitor

Launcher itself poses some sort of a roadblock, to bring most recent versions of the game online, since the flow of the launcher->game client is unknown, all of those versions are VMProtected and other factors, this approach doesn’t look feasible at the moment.

## Useful tools
Decompilers: IDA, Ghidra
Debugger: x32dbg + Scyllahide
Packet captures and network activity: Wireshark
Process activity: procmon
DLL calls logging: DLL Api Monitor

## Useful links
CODOL builds - https://mega.nz/folder/SfglgLyD#VnmwxC4zAuMZ1dsh9gkRIA/folder/DDRQzBBa
S1x client - https://github.com/CBServers/s1x-client/tree/main
BO2 server binaries with PDB - https://mega.nz/file/VUVATCDK#aHbD69iprnTDgvrvwTQD_LZ9fCWUZ06zltZkNhULT1M
CODOL 3.3.3.3 x32dbg database - https://limewire.com/d/86qyJ#8NZfFIR80g


