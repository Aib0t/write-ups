
# Plazma introduction

A lot of work on Plazma documentation was done by **abarichello** at https://github.com/abarichello/bfheroesFesl. We will avoid copy-pasting his findings here, and focus more on general Plazma documentation.

### Plazma/Fesl name confusion
If you dig around various codebases of EA games server emulators, you will be greet with a lot of confusing stuff. Some of codebases call it Plazma, some call it FESL.

This confusion is coming from EA itself, since they switched naming after version 2. If we check the residual strings in binaries we will see the following:

- 2006 - FESLSDK 2.9
- 2007 - PLAZMASDK 3.5
- 2008 - PLAZMASDK 4.4

Despite naming changes the protocol and everything else stayed 99% the same, so to save the reader from confusion the whole online network middleware will be refereed to as "Plazma".

With this aside, let's continue.

## FESL

FESL is a weird combo of authorization, profiles, stats and game queue services. In general, games would connect to FESL first to authorize, then use the authorization tickets to authorize in other services.

 Unlike other services, which utilize a single call verb in the header, FESL has service name in header (fsys, acct and others) and a task name in the tag TXN (meaning Taxon) in the payload
```
/EXAMPLE HERE/
```
  
One more curious thing about it is a rather complex field structure, supporting arrays of objects and optional fields.

For arrays the following format is used
```
array_name.[]=size
array_name.0.name=text
array_name.0.value=value
```
For objects with optional fields the following format is utilized  
```
props.{}=100
props.{attrib}=something
```
  
Despite all fields being text, in the code they are parsed differently, usually as text or u64, but there are other supported formats.

## Theater

Theater, or “GameCoordinator” is a combination of “Sessions storage” and “NAT Punching/Tunneling” solution in EA universe. This service utilizes 2 ports, TCP and UDP on the same port number. 

To get matched into a session client would send a packet to PlayNow service of FESL first, describing all filtering requests. This puts the client into a queue. After checking all sessions available to Theater client would get back a list of the sessions, or a task to create a new one (which usually does nothing, but more on this later)

After getting a list of the sessions client is negotiating with Theater to get all the session info, including IP and performs a connection.

The second main function is NAT Punching/Tunneling, which is listening on the UDP port of the Theater. This service is largely unused, since most games relied on dedicated servers by EA, and not P2P connection.

## Messenger

For whatever reason, EA moved friends, rich presence, invites and messaging service to a dedicated server. It’s not a very interesting service, so that’s pretty much all I can say about it.

## Other services
There are 2 other less common services - Magma and easo.

Magma is a HTTPS web API, which stores yet another subset of data, mainly related to stats. Not much is known about it, and the best documented Magma implementation exists as a part of Battlefield Heroes server emulator.

easo is a simple http based file storage, which keeps files for some titles. Like MOTD, in case of BFBC2.
