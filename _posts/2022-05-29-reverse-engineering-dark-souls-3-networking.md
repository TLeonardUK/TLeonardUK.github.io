---
layout: post
title: Reverse Engineering Dark Souls 3 Networking (Part 1)
description: Breaking down and investigating how Dark Souls 3 communicates with its online services.
summary: Breaking down and investigating how Dark Souls 3 communicates with its online services.
tags: [reverse engineering,networking,dark souls 3,multiplayer,ds3os]
---

# The Backstory

FromSoftware's SoulsBorne series are some of my most cherished games. I've got a lot of fond memories in these games, mainly participating in jolly cooperations with friends.

Being a programmer, I've also had somewhat more interest than is probably healthy in how they've but together these masterpieces. To that end I spend a lot of time browsing social media related to modding and data-mining these games.

It was around a year ago that I was reading the ?ServerName? discord, probably the largest modding community for these games. Some of the members were discussing the prospect of private servers for games so that modded games had somewhere to play that wasn't the official banned-server, which by its nature was populated with all the worst kinds of cheaters.

This discussion peaked my interest as it just the kind of challenge I have the skillset to help with. 

From what I saw several members had attempted to make private servers for the latest game, Dark Souls III, but hadn't got much past just getting the game to try to establish connections to an third-party server, the information they had acquired gave me the starting point to eventually produce: [Dark Souls 3 Open Server (GitHub)](http://github.com/TLeonardUK/ds3os). 

This blog series is mostly a discussion about how DS3OS was produced, and contains a technical dive into how the games network architecture is setup. I may skip over some things here and there to avoid anyone having to read about hours of experimentation in a disassembler / debugger, so apologies for any logical jumps.

# What does the server actually do?

So if we're looking at how to emulate the retail game server in Dark Souls 3, you might be asking - what does that server actually *do*?

Well you might be forgiven for thinking about the server in terms of it being a dedicated server for the purpose of realtime multiplayer - as that is obviously the main apparent online feature in the game. 

However all multiplayer traffic in FromSoftware's games is actually sent entirely peer-to-peer. What the game server actually does is handle a variety of game services, most importantly;

- Matchmaking players for coop, invasions and covenant auto-summons.
- The funny messages players leave around the world. 
- Bloodstains left by players deaths.
- Ghosts used to show players glimpses into other peoples games.
- Leaderboards for various covenants.
- Cheat detection and banning.
- And various other miscellaneous services.

# Establishing connections to a different server

Getting the game to connect to a different server is suprisingly more involved than you might expect.

The initial connection made is a TCP connection, specifically to **fdp-steam-ope-login.fromsoftware-game.net**, this can be seen by taking a wireshark capture of the game booting up.

 ![Wireshark showing hostname](/assets/images/posts/ds3os/wireshark_hostname.png)

<sup>Note: 54.148.23.40 is the resolved ip of fdp-steam-ope-login.fromsoftware-game.net. Imagine a complete connection here, FromSoftware's servers are down at the time of writing.</sup>

My initial thoughts at this point would probably be to add a host file entry, or detour the dns resolution functions in the game to have that domain return the IP of my shiney new server. Then we could get down to figuring out the traffic protocol!

However if we look at the actual data being sent during this initial connection, you might notice a stumbling block:

 ![Wireshark showing initial packet data](/assets/images/posts/ds3os/wireshark_initial_data.png)

That looks suspiciously like its encrypted or ciphered in some way! Not a problem you might say? Just find the encryption key? 

Well if we load the game's exe up in my favourite decompiler, [Ghidra](https://github.com/NationalSecurityAgency/ghidra), we could have a look for clues! The game makes use of RTTI (Runtime Type Information), so there is a wealth of type names that can give us a general idea of how the game structures things.

 ![Ghidra showing Nauru functions](/assets/images/posts/ds3os/ghidra_nauru.png)

Aha, that is promising, lots of class names that look very much like they handle network communications. And if we look even closer we can see two very interesting types - CWCObject and RSAObject, both of which contain names of encryption ciphers. In fact the Nauru namespace is actually where the game stores all the logic for handling connections to the game server.

To determine if these are used we can search in Ghidra for the places these types are referenced. Doing so will turn up a handful of functions that look like constructors.

 ![Ghidra showing RSAObject constructor](/assets/images/posts/ds3os/ghidra_rsaobjet_constructor.png)

If we run the game with a debugger attached, and place breakpoints in these functions, we can determine the only type used at the same time the initial connection takes place is RSAObject.

This is bad news for us RSA is a form of public-key cryptography, which means traffic encrypted with a recipients public-key can only be decrypted by their secret private-key. So while we can get the game to connect to our new server, we have no way to decrypt the data the game sends it, as the private key to decrypt it is safely hidden away out of our reach on FromSoftware's servers.

So the question then becomes, can we trick the game into using a key we supply, that we have the matching private key for, as well as trick it into connecting to our new server?

So lets start with the easy steps first, and search the exe for reference to the server hostname or anything that look like public keys (which are normally stored in some very obvious, easily searchable, formats like [PEM](https://www.cryptosys.net/pki/rsakeyformats.html)). If we find these, we should just be able to patch them.

 ![Ghidra showing no string results](/assets/images/posts/ds3os/ghidra_string_search.png)

Bad luck, none of our string searchs show anything useful. 

Interestingly if we look at the previous entries in the Dark Souls series we can see the keys and hostname ARE hard-coded in the exe in a visible format.

 ![Ghidra showing Dark Souls 2 keys](/assets/images/posts/ds3os/ghidra_darksouls2_keys.png)

So unless Dark Souls 3 has fundementally changed how it handles online connections, which seems unlikely as most of the RTTI in the area matches up, then the key is probably obfuscated or stored in a different location.

We know that the key has to be available in memory at the point its passed to the OpenSSL library to do the packet encryption, so what we can do is find the relevant openssl functions, put breakpoints in and trace backwards up the callstack to where the key comes from? Fortunately openssl has a lot of RTTI information and it's functions tend to compile to the same machine code, so its easy to pattern match from known compiled code and find the functions we want.

Doing so (and spending several hours staring at a debugger) will eventually lead us to this function, which is what produces the public key (and hostname!).

 ![Ghidra showing TEA encrypt](/assets/images/posts/ds3os/ghidra_tea_encrypt.png)

You can ignore the disassembly for this one, Ghidra does a poor job of it generating it in this case.

So what is this function? It takes in a block of data and 4 ints, then does ~something~ to it and spits out a block of data with our encryption key and hostname in it?

Well it turns out its actually an implementation of the [Tiny Encryption Algorithm](https://en.wikipedia.org/wiki/Tiny_Encryption_Algorithm). FromSoftware is being extra sneaky by encrypting the encryption keys. The 4 ints are the TEA key and the block of data is what contains our public key and hostname.

The data being decrypted is static and stored at offset *0x144F4A5B1* (on the current patch at time of writing). Its hidden interleaved in a block of unrelated code (very sneaky!). Its encrypted with a static key of:

```
0x4B694CD6, 0x96ADA235, 0xEC91D9D4, 0x23F562E5
```

 ![Ghidra showing TEA data](/assets/images/posts/ds3os/ghidra_tea_block.png)

So now the offset is known, connecting to a different game server is as simple as writing our own data blob, with our own keys and hostname, and patching the game memory at the offset.

The format of the data blob is nice and straightforward:

| Offset      | Size        | Description |
|:----------- |:----------- |:----------- |
| 0           | 0x1B0       | UTF8 PEM encoded public key |
| 0x1B0       | 0x58        | UTF16 hostname of server |


# Coming up

In the next post I'll be discussing how the initial connection protocol was reverse engineered. Coming up shortly ...
