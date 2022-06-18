---
layout: post
title: Reverse Engineering Dark Souls 3 Networking (#6 - Async Mechanics)
description: Breaking down and investigating how Dark Souls 3 communicates with its online services.
summary: Breaking down and investigating how Dark Souls 3 communicates with its online services.
tags: [reverse engineering,networking,dark souls 3,multiplayer,ds3os]
---

# Recap

In the previous entry we went over the initial message exchanges that happen with the server, and how character data is managed and updated.

In this post we're going to start looking into asynchronous gameplay mechanics. For those not familiar with the term, asynchronous gameplay is a term used to mean gameplay where you are no directly interacting with another player in real-time. In Dark Souls I use this to refer to mechanics such as blood stains, messages, ghosts and other similar things.

# Blood Messages

So lets dive in with the most well known, and meme'd, gameplay feature of From Software games - Messages. Or as known in the code "Blood Messages". These are messages people can place anywhere in the game-world, with the restriction that the message can only use fixed phrase-structures and a fixed vocabulary of insertable words.

[![Blood Messages](/assets/images/posts/ds3os_6/blood_message.png)](/assets/images/posts/ds3os_6/blood_message.png)

How blood messages work is super straight forward. 

Each time the player places a message, its stored locally in the users save data, there are slots for a maximum of 10 messages. 

At the same time the message is stored the follow exchange is sent to the server:

```protobuf
message RequestCreateBloodMessage {
    required uint32 online_area_id = 1;
    required uint32 character_id = 2;                      
    required bytes  message_data = 3;
}
```

- ```online_area_id``` is a unique numeric id for the area that the user is, it is used for filtering which blood messages are relevant at a given time.
- ```message_data``` is a buffer of serialized data that represents the text, associated animation and location of the message that was placed.

```protobuf
message RequestCreateBloodMessageResponse {
    required uint32 message_id = 1;
}
```

The ```message_id``` that is returned is a unique identifier on the server, every message has a unique id. The id is sequential and goes up each time a message has been placed in the game. When this is returned the game stores this information in the save data as well - we shall see why in a moment.

There is also an equivalent remove request if the user deletes one of their messages.

```protobuf
message RequestRemoveBloodMessage {
    required uint32 online_area_id = 1;
    required uint32 message_id = 2;
}
```

To actually view messages from other players the game will every few minutes send out the following request

```protobuf
message BloodMessageDomainLimitData {
    required uint32 online_area_id = 1;
    required uint32 max_type_1 = 2;                     
    required uint32 max_type_2 = 3;         // Maximum returned is the sum of both of these values.
}

message RequestGetBloodMessageList {
    required uint32 max_messages = 1;                   
    repeated BloodMessageDomainLimitData search_areas = 2;
}
```

- The ```search_areas``` field dictates which online areas we want to get messages from, and how many we want to get from that area. Typically this only includes the area the player is currently in.

```protobuf
message BloodMessageData {
    required uint32 player_id = 1;
    required uint32 character_id = 2;   
    required uint32 message_id = 3; 
    required uint32 good = 4;         
    required bytes  message_data = 5;
    required string player_steam_id = 6;
    required uint32 online_area_id = 7;
    required uint32 poor = 8;                   
}

message RequestGetBloodMessageListResponse {
    repeated BloodMessageData messages = 1;
}
```

The result here is a list of all messages that were found in the given area, including information about who made the message and how many times it's been rated as good or poor by other players.

To limit the amount of data that the server needs to store at a given time, the server does not actually store any of the bulky ```message_data```. Instead it appears to be held in a "hot cache" in memory, after a given amount of time and once the user is offline the message is evicted from the cache. Only messages in this cache will ever be returned from the RequestGetBloodMessageList. 

This is why when you go offline and come back after a week or so, your messages will have only gone up by a few points, as they will normally have been evicted shortly after you went offline. The cache seems to have some heuristics involved to keep messages active which are highly popular, judged by being frequently evaluated.

You might ask then how the game gives the illusion that your messages are persistent and always in the game world? Well what happens is when you log back in with a character the following request is exchanged:

```protobuf
message LocatedBloodMessage {
    required uint32 online_area_id = 1;
    required uint32 message_id = 2;
}

message RequestReentryBloodMessage {
    repeated LocatedBloodMessage messages = 1;
    required uint32 character_id = 2;                   
}
```

- The ```messages``` field is a list of all the messages the player has in their save data.

```protobuf
message RequestReentryBloodMessageResponse {
    repeated uint32 recreate_message_ids = 1;       
}
```

- The ```recreate_message_ids``` field is a list of all the messages requested which are no longer in the cache, and need to be resent to the server.

If this request returns any message ids, the game initiates another request:

```protobuf
message RequestReCreateBloodMessageList {
    required uint32 character_id = 2;                     

    repeated group Blood_message_info_list = 3 {
        required uint32 online_area_id = 1;
        required bytes  message_data = 2;
        required uint32 unknown_1 = 3;
        required uint32 unknown_2 = 4;
    }
}
```

- ```Blood_message_info_list``` is a list of data for all the messages that the server asked us to recreated.

```protobuf
message RequestReCreateBloodMessageListResponse {
    repeated uint32 message_ids = 1;
}
```

- ```message_ids``` are the unique id's for all the messages that have been recreated. I've never seen these ids actually change from what the messages were created with, but in theory they may do if the server ever clears its database.

So far, so simple. The only element of the blood messages we haven't looked at so far is how evaluations are performed. Each message can be rated either good or poor once by a user. When the player rates a message the following request is sent:

```protobuf
message RequestEvaluateBloodMessage {
    required uint32 online_area_id = 1;
    required uint32 message_id = 2;
    required bool was_poor = 3;
}
```

```protobuf
message RequestEvaluateBloodMessageResponse {
    // Empty response.
}
```

Simple enough, this request updates the good or bad counter on the server. The server also does one additional thing at this point - it sends a push request to the client who wrote the original message if they are online.

```protobuf
message PushRequestEvaluateBloodMessage {
    required PushMessageId push_message_id = 1;
    required uint32 player_id = 2; 
    required uint32 message_id = 3;
    required string player_steam_id = 4; 
    required bool was_poor = 5; 
}
```

This tells the original author who evaluated the message and if they evaluated as good or poor. This is used by the game to grant the player a health refill - which occurs regardless of it was a good or poor rating.

# Ritual Marks

Ritual Mark's aren't something you will ever see in game, they are a feature that was cut during development. I'm mentioning them now more as trivia than anything, the game contains a complete protobuf set that is structurally setup almost the same as the blood messages. It looks like this was likely something to do with the cut feature that allow you to create bonfires anywhere in the world using enemy corpses.

The below video is worth watching, it explains and recreates this functionality.

[![Sacrifice Youtube](https://img.youtube.com/vi/zziY68RdKK4/0.jpg)](https://www.youtube.com/watch?v=zziY68RdKK4)

# Blood Stains

So on to the next feature - blood stains! These are puddles of blood that are shown wherever players in the world have died. Going up to them allows the user to interact with them and view a ghostly replay of the original players death.

[![Blood Stains](/assets/images/posts/ds3os_6/blood_stain.png)](/assets/images/posts/ds3os_6/blood_stain.png)

So how do these work? Simple, when the user dies the following request is sent.

```protobuf
message RequestCreateBloodstain {
    required uint32 online_area_id = 1;
    required bytes  data = 2;                       
    required bytes  replay_data = 3;         
}
```

```protobuf
message EmptyResponse {
}
```

Weirdly this is one of the few requests that doesn't have a dedicated response type, it just uses a generic ```EmptyResponse``` protobuf.

Like the blood messages the creation request is simple, it specifies the online area where the bloodstain was made and then includes a block of serialized data that describes the bloodstain. The only interesting thing here is the data is split into two parts, the general ```data```, including things like the location, and the chonky ```replay_data``` that includes everything required to show a replay of the player's death - serialized animations. 

To actually view blood stains from other players the game will every few minutes send out the following request

```protobuf
message DomainLimitData {
    required uint32 online_area_id = 1;
    required uint32 max_items = 2;
}

message RequestGetBloodstainList {
    required uint32 max_stains = 1;               
    repeated DomainLimitData search_areas = 2;
}
```

```protobuf
message BloodstainInfo { 
    required uint32 online_area_id = 1;
    required uint32 bloodstain_id = 2;
    required bytes data = 3;    
}

message RequestGetBloodstainListResponse {
    repeated BloodstainInfo bloodstains = 1;
}
```

Again, this works similar to the blood messages. We say what areas we are interested in, and how many bloodstains we want, and the server replies with a list of the available bloodstains and the data that goes along with them.

When the user interacts with a bloodstain, an additional request is made for the data required to show the ghost. This avoids the server having to send the data to players who may never interact with the bloodstain.

```protobuf
message RequestGetDeadingGhost {
    required uint32 online_area_id = 1;    
    required uint32 bloodstain_id = 2;
}
```

```protobuf
message RequestGetDeadingGhostResponse {
    required uint32 online_area_id = 1;    
    required uint32 bloodstain_id = 2;
    required bytes replay_data = 3;
}
```

# Ghosts

Blood stains aren't the only place you can see ghosts of other players in the game. The game also actions that occur near bonfires and displays these in other players worlds, giving the illusion of a populated world.

[![Ghosts](/assets/images/posts/ds3os_6/ghost.png)](/assets/images/posts/ds3os_6/ghost.png)

This is probably one of the simplest features in the game. The game will randomly record actions several seconds of action near the bonfire, and based on unknown heuristics, when it decides one of these replays is "interesting" enough it will send a request to the server:

```protobuf
message RequestCreateGhostData {
    required uint32 online_area_id = 1;
    required bytes replay_data = 2;
}
```

```protobuf
message RequestCreateGhostDataResponse {
    // Empty response.
}
```

The replay data is structured almost identically to the blood stain ghosts.

When the player is sitting at, or near a bonfire, the game will very infrequently send the following request to the server:

```protobuf
message DomainLimitData {
    required uint32 online_area_id = 1;
    required uint32 max_items = 2;
}

message RequestGetGhostDataList {
    required uint32 max_ghosts = 1;               
    repeated DomainLimitData search_areas = 2;
}
```

```protobuf
message GhostData {
    required uint32 unknown_1 = 1;         
    required uint32 ghost_id = 2;
    required bytes  replay_data = 3;
}

message RequestGetGhostDataListResponse {
    repeated GhostData ghosts = 1;
}
```

Interestingly there is no additional request for the ghost data like you see in the bloodstains, it just sends it straight in the reply. I'm guessing the assumption is we are always going to use it when requested. The maximum number of ghosts requested is also low - normally 3, so they probably didn't think the additional request was worth it.

As soon as the response is received, one of the ghosts is selected and the ghost replay is run.

# The Great Bell

One small and easily missed feature is the great bell in Archdragon Peak. 

[![Great Bell](/assets/images/posts/ds3os_6/great_bell.png)](/assets/images/posts/ds3os_6/great_bell.png)

When you ring it and change the world state ready for the boss, a request is sent to the server.

```protobuf
message RequestNotifyRingBell {
    required uint32 online_area_id = 1;    
    required bytes  data = 2;
}
```

- ```data``` contains a very small block of information, seems to just be a single value, possibly the new-game level the player is on.

```protobuf
message RequestNotifyRingBellResponse {
    // Empty response.
}
```

The server then finds all the other players in the same area specified by online_area_id and sends the following push message to all of them.

```protobuf
message PushRequestNotifyRingBell {
    required PushMessageId push_message_id = 1;
    required uint32 player_id = 2;    
    required uint32 online_area_id = 3;    
    required bytes  data = 4;
}
```

What does all this network activity result in? If you listen carefully, you can hear other players ringing the bell in the background!

# Leaderboards

The Darkmoon Knights covenant maintains a leaderboard of how many contributions each player has made. A special multiplayer item known as the [Roster Of Knights](https://darksouls3.wiki.fextralife.com/Roster+of+Knights) allows the player to browse all the players in the covenant and see their current rank.

[![Leaderboard](/assets/images/posts/ds3os_6/leaderboards.png)](/assets/images/posts/ds3os_6/leaderboards.png)

Implementation of this feature works as a pretty standard leaderboard setup, the system is generic and can be used with any number of leaderboard's, though the game only uses a single one (board_id for all the messages below is always set to 0).

Each time the user contributes to the covenant their contribution is updated and stored locally in their save data. At the same time, a request is made to the server:

```protobuf
message RequestRegisterRankingData {
    required uint32 board_id = 1;     
    required uint32 character_id = 2;     
    required uint32 score = 3;         
    required bytes  data = 4;
}
```

- ```score``` contains the number of contributions the user has made.
- ```data``` contains a small block of serialized data that contains some misc information such as the users steam-id, used for displaying the user's name in the leaderboard.

```protobuf
message RequestRegisterRankingDataResponse {
    // Empty Response.
}
```

When the user opens the leaderboard the first request that occurs is:

```protobuf
message RequestCountRankingData {
    required uint32 board_id = 1;       
}
```

```protobuf
message RequestCountRankingDataResponse {
    required uint32 count = 1;
}
```

This tells the game how many entries are in the leaderboard, and thus how many pages of results it needs to show to the player.

As the player pages through each page a request is made to get the data for that particular page of results.

```protobuf
message RequestGetRankingData {
    required uint32 board_id = 1;     
    required uint32 offset = 2;       // 1-indexed.
    required uint32 count = 3;         
}
```

```protobuf
message RankingData {
    required uint32 player_id = 1;  
    required uint32 character_id = 2;   
    required uint32 serial_rank = 3;
    required uint32 rank = 4;
    required uint32 score = 5;          
    required bytes  data = 6;
}

message RequestGetRankingDataResponse {
    repeated RankingData data = 1;
}
```

- There are two ranks for each entry ```serial_rank``` and ```rank```. ```serial_rank``` is the order of entries, it always goes from 1 to the number of entries in the board. ```rank``` is almost the same except that if two entries have the same value they both share the same rank - this is what's actually displayed to the player in the UI.

If the player presses X to show their own ranking, a request is first made to find their currently rank, and then the UI switches to the relevant page. If the user has no rank, it just switches to the last page of rankings.

```protobuf
message RequestGetCharacterRankingData {
    required uint32 board_id = 1;       
    required uint32 character_id = 2; 
}
```

```protobuf
message RequestGetCharacterRankingDataResponse {
    optional RankingData data = 1;
}
```

If the user selects a player to view their info, the profile overview screen is shown and populated with a request to ```RequestGetPlayerCharacter``` to retrieve the player's character from the last time they uploaded it with the periodic call to ```RequestUpdatePlayerCharacter```

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

# Coming Up

We're almost at the part I'm sure everyone is interested in! The next post is going to go over the real-time mechanics, namely invasions, summoning and quick matches. Exciting!