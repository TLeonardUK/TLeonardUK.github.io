---
layout: post
title: Reverse Engineering Dark Souls 3 Networking (#5 - Character Management)
description: Breaking down and investigating how Dark Souls 3 communicates with its online services.
summary: Breaking down and investigating how Dark Souls 3 communicates with its online services.
tags: [reverse engineering,networking,dark souls 3,multiplayer,ds3os]
---

# Recap

In the previous posts we've looked over the full protocol used by game to communicate with its online services. At this point we are capable of producing a server that can fully communicate with the client.

The last part that remains, is what actually messages are sent between the client and the server to facilitate the game-play mechanics in the game. As with previous posts, I won't go into detail on how these were disambiguated, as it's mostly fairly boring pattern matching and browsing of disassembly.

# Login Awaits

When the user has fully established a connection to the server, the first request that is sent to the server is a nice straightforward one called RequestWaitForUserLogin, the protobuf for which looks as follows;

```protobuf
message RequestWaitForUserLogin {
    required string steam_id = 1;
    required uint32 unknown_1 = 2;
    required uint32 unknown_2 = 3;
    required uint32 unknown_3 = 4;
    required uint32 unknown_4 = 5;
}
```

And has a response message that looks like this:

```protobuf
message RequestWaitForUserLoginResponse {
    required string steam_id = 1;
    required uint32 player_id = 2; 
}
```

This exchange as you might guess from the names is to allow the server to do any setup it requires before the client starts making other requests. In DS3OS, and likely on the retail server, this involved grabbing the users account details from the storage database.

You will notice that in the response a ```player_id``` value is sent back. This is a unique number that identifies a player on the server, and persists forever, the users steam_id is permanently linked to it. These player ids are what will be used to identify players throughout any future exchanges with the server, steam_ids are never used as identities, they are only passed around in situations where it's required to facilitate the use of steam functionality.

Once the user has been logged in the server also sends once of its rare push-requests:

```protobuf
message PlayerInfoUploadConfigPushMessage { 
    required PushMessageId push_message_id = 1;                 
    required PlayerStatusUploadConfig config = 2;               
    required uint32 player_character_update_send_delay = 3;     
    required uint32 player_status_send_delay = 4;               
}
```

This configures how often, and what type, of large messages the client sends to the server. The server may also send this push message at any time while playing. The server uses this to load balance the traffic being sent to the server in real-time. The messages this affects are primarily about reporting of the player's status and character information.

# Announcements

As soon as the player is logged in the first user visible action occurs! This is a simple exchange that uses the following request and response:

```protobuf
message RequestGetAnnounceMessageList {
    required uint32 max_entries = 1;
}
```

```protobuf
message AnnounceMessageData {
    required uint32 unknown_1 = 1;  
    required uint32 index = 2;
    required uint32 unknown_2 = 3;  
    required string header = 4;
    required string message = 5;
    required Frpg2PlayerData.DateTime datetime = 6;
}

message AnnounceMessageDataList {
    repeated AnnounceMessageData items = 1;
}

message RequestGetAnnounceMessageListResponse {
    required AnnounceMessageDataList changes = 1;
    required AnnounceMessageDataList notices = 2;
}
```

This request is used to get a list of announcements that are shown to the user on the main menu. On the official server these are filled with patch notes and server status information. Interestingly there is a huge amount of redundant data in this exchange, many of the fields make no difference to how the information is displayed - the split between "change" and "notice" announcements for example, which all get displayed concatenated into a single list when displayed. I suspect some of this may be a left over from development, when they were perhaps displayed differently.

Implementing this allows us to make the first user-visible changes to the server-connection:

[![Announcement](/assets/images/posts/ds3os_5/announcement.png)](/assets/images/posts/ds3os_5/announcement.png)

# Character Registration

At this point if you stay on the main menu, no more exchanges will occur. Messages start being sent as soon as the player selects a character and loads into the game.

The first exchange that's sent on selecting the character involves these messages:

```protobuf
message RequestUpdateLoginPlayerCharacter {
    required uint32 character_id = 1;                               
    repeated uint32 unknown_2 = 2;
}
```

```protobuf
message QuickMatchRank {
    optional uint32 rank = 1;       
    optional uint32 xp = 2;         
}

message RequestUpdateLoginPlayerCharacterResponse {
    required uint32 character_id = 1;                              
    required QuickMatchRank quickmatch_brawl_rank = 2;   
    required QuickMatchRank quickmatch_dual_rank = 3;   
}
```

These messages essentially tell the server what character the player is going to use while playing. The ```character_id``` is assigned by the game when you created a new character, it increments sequentially for each character created on the same save file.

As each character has its own entries in most of the online mechanics - leaderboards, messages, etc. You will see that most references to a particular player in-game will uniquely identify them using both a character_id and player_id.

The response to this message also includes the players current ranks for the brawl and duel in the undead matches game mode. It's a bit of an odd place to put this information, but I figure From Software probably did it to save setting up additional more-specific messages.

Once the player has identified which character they are going to use, they will periodically send the following message exchange, the frequency its sent is determined by the values in the ```PlayerInfoUploadConfigPushMessage``` message we saw earlier.

```protobuf
message RequestUpdatePlayerCharacter {
    required uint32 character_id = 1;          
    required bytes character_data = 2;      
}
```

```protobuf
message RequestUpdatePlayerCharacterResponse {
    // Empty Response
}
```

<sup>Note: You will see a lot of these "Empty Response" protobufs, they exist purely to tell the game that their request was received. Failure to send these empty responses back will result in the game deadlocking.</sup>


This message essentially updates a serialized block of save-data describing the character's equipment/appearance/items/etc, which the server stores persistently even when the player is offline. This block of data can be retrieved for any character using the below message. 

In-game this is only used by the [Roster of Knights](https://darksouls3.wiki.fextralife.com/Roster+of+Knights) leaderboard item to show another players profile. It's also likely that from uses this on the backend for cheat-detection and similar purposes.

```protobuf
message RequestGetPlayerCharacter {
    required uint32 player_id = 1;
    required uint32 character_id = 2;            
}
```

```protobuf
message RequestGetPlayerCharacterResponse {
    required uint32 player_id = 1;
    required uint32 character_id = 2;           
    required bytes  character_data = 3;
}
```

# Player Telemetry

While playing, the game sends back a whole wreath of telemetry data to the server. A lot of this information is fairly uninteresting information - what the player killed, what items they bought, etc. Mostly useful for From Software's design team. You can look at all this telemetry data by browsing the [FpdLogMessage.proto](https://github.com/TLeonardUK/ds3os/blob/main/Protobuf/FpdLogMessage.proto) file, these protobuf's are wrapped in the following message the client sends when the relevant actions occur (there are also a few other message types as well all begin with RequestNotify*):

```protobuf
enum LogType
{
    UseMagicLog = 2020;
    ActGestureLog = 2021;
    UseItemLog = 3000;
    PurchaseItemLog = 3001;
    GetItemLog = 3002;
    DropItemLog = 3003;
    LeaveItemLog = 3004;
    SaleItemLog = 3005;
    StrengthenWeaponLog = 3010;
    GlobalEventLog = 5001;
    SystemOptionLog = 8001;
    VisitResultLog = 7040;
    QuickMatchResultLog = 7050;
    QuickMatchEndLog = 7060;
}

message RequestNotifyProtoBufLog {
    required LogType type = 1; 
    required bytes common = 2;
    required bytes data = 3;
}
```

However there is one bit of telemetry data that is -very- important, as it's used to provide functionality to a lot of the game mechanics.

This telemetry data is sent using these messages:

```protobuf
message RequestUpdatePlayerStatus {
    required bytes status = 1;
}
```

```protobuf
message RequestUpdatePlayerStatusResponse {
    // Empty Response
}
```

This message is sent very frequently, dictated by values in the ```PlayerInfoUploadConfigPushMessage``` message we saw earlier.

So what exactly does the mysterious ```status``` field contain? Well it contains a serialized protobuf. This protobuf is massive and contains a dump of everything from what item the player is currently using, to if they are in another users world, even down to how many NPC's they've killed.

How ever due to the frequency that this protobuf is sent, the game only sends it in its entirety once, all future messages contain only the fields that have changed which the server merges with its current value to get the latest status.

I'm not going to paste the entire protobuf here as its about 300 lines long, however if you can view it [Here](https://github.com/TLeonardUK/ds3os/blob/main/Protobuf/Frpg2PlayerData.proto), the AllStatus protobuf is the one used. But just to give you a basic idea, here's a snippet:

```protobuf
message PlayerStatus {

    optional uint32 regulation_version = 1;                
    optional uint32 unknown_2 = 2;                           // 2

    optional bool cross_region_matchmaking_disabled = 3;
    optional int32 soul_level = 4; 

    optional uint32 sinner_points = 5;
    optional uint32 unknown_6 = 6;                           
    optional uint32 is_invadable = 7;                        
    optional uint32 can_summon_for_way_of_blue = 8;     
    optional uint32 unknown_9 = 9;                           
    optional uint32 can_summon_for_watchdog_of_farron = 10;   
    optional uint32 can_summon_for_aldritch_faithful = 11;    
    optional uint32 can_summon_for_spear_of_church = 12;      
    optional uint32 unknown_13 = 13;                         
    optional uint32 unknown_14 = 14;                         
    optional WorldType world_type = 15;                      
    optional uint32 covenant = 16;                           

    repeated uint32 played_areas = 17;

    repeated uint32 unknown_18 = 18;                         
    optional uint32 embered = 19;                         
    optional uint32 souls = 20;                              
    optional uint32 soul_memory = 21;                        
    optional uint32 archetype = 22;                        

    optional int32 hp = 23;
    optional int32 max_hp = 24;
    optional int32 base_max_hp = 25;
    optional int32 fp = 26;
    optional int32 max_fp = 27;
    optional int32 base_max_fp = 28;
    optional int32 stamina = 29;
    optional int32 max_stamina = 30;
    optional int32 base_max_stamina = 31;

    optional uint32 unknown_32 = 32;                         
    optional uint32 unknown_33 = 33;                         
    optional NetMode net_mode = 34;                         
    optional uint32 dried_fingers_active = 35;              
    optional InvasionType invasion_type = 36;                       
    optional uint32 character_id = 37;                       

    optional string name = 38;
    optional bool is_male = 39;

    .... it goes on for many lines ....

};
```

You can see from some of the fields that this is how a lot of the game mechanic information is exchanged. For example field's like  ```can_summon_for_aldritch_faithful``` directly dictate if the player is using the Aldritch Faithful covenant can be summoned by other player's games for covenants invasion.

Interesting as this is user controlled, a game can cut themselves fully off from invasions or other game-mechanics they don't like simply by setting the relevant fields to false.

Also just because I find it interesting, the protobuf also contains the following field:

```protobuf
    repeated int32 anticheat_data = 62;
```

Which is hilariously obfuscated. It's an array of seemingly random values that are shuffled around to look like it's some kind of state being changed. How ever if the game thinks you've tampered with any executable code, it will silently slip in a value of ```0x1770```. When the server receives this value it will mark the players account as cheating and will ban them on the next ban-wave (which occur once a week).

# Coming Up

At this point we know about all the messages required for the user to register a character and get into the game. What remains is to go over the game mechanics.

I'm going to split the mechanics into two final posts, one going over asynchronous ones (ghosts, bloodstains, messages) and one going over all the real-time ones (invasions, summoning, quick matches).
