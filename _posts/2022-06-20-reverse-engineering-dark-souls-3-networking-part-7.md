---
layout: post
title: Reverse Engineering Dark Souls 3 Networking (#7 - Realtime Mechanics)
description: Breaking down and investigating how Dark Souls 3 communicates with its online services.
summary: Breaking down and investigating how Dark Souls 3 communicates with its online services.
tags: [reverse engineering,networking,dark souls 3,multiplayer,ds3os]
---

# Recap

In the previous entry we went over asynchronous gameplay mechanics. No we are going to move onto the more interesting real-time gameplay mechanics, these are mechanics were you are directly interacting with other players - namely summoning or invading them! 

This should be quite an interesting post!

# Summoning

The main functionality of Dark Souls 3's cooperative online comes from summoning. Unlike a lot of traditional games, there are no lobbies and no invites - instead players can place signs anywhere in the world, which can be seen by other players, and can be interacted with to bring the player who placed the sign into the other users world.

[![Summoning](/assets/images/posts/ds3os_7/summoning.png)](/assets/images/posts/ds3os_7/summoning.png)

Initially this works very similarly to the async mechanics. To create a remove a sign the player uses some familiar types of requests.

```protobuf
message RequestCreateSign {
    required uint32 map_id = 1;
    required uint32 online_area_id = 2;
    required MatchingParameter matching_parameter = 3;
    required bool is_red_sign = 4;
    required bytes player_struct = 5;
}
```

- ```is_red_sign``` determines if the sign is for summoning for PvP rather than the normal PvE usage of signs.
- ```player_struct``` contains a serialized representation of the player character, which is shown as a ghost in other players worlds when they examine the sign.
- ```matching_parameter``` I'll explain how this one works below as it's fairly critical to a lot of gameplay elements.

```protobuf
message RequestCreateSignResponse {
    required uint32 sign_id = 1;
}
```

- ```sign_id``` is a unique, sequentially incrementing identifier for the sign and used by all future requests that reference the placed sign.

Removal is again, very familiar, just providing the sign id and its area.

```protobuf
message RequestRemoveSign {
    required uint32 map_id = 1;
    required uint32 online_area_id = 2;
    required uint32 sign_id = 3;
}
```

```protobuf
message RequestRemoveSignResponse {
    // Empty response.
}
```

Viewing signs in an area is fairly straightforward, every couple of minutes or so request is made to get a list of signs in the area the player is in.

```protobuf
message SignInfo {
    required uint32 player_id = 1;                      
    required uint32 sign_id = 2;                        
}

message SignDomainGetInfo {
    required uint32 online_area_id = 1;                    
    required uint32 max_signs = 2;                         
    repeated SignInfo already_have_signs = 3;
}

message RequestGetSignList {
    required uint32 unknown_id_1 = 1;                       
    repeated SignDomainGetInfo search_areas = 2;
    required uint32 max_signs = 3;                          
    required MatchingParameter matching_parameter = 4;
    required SignGetFlags sign_get_flags = 5;
}
```

```protobuf
message SignData {
    required SignInfo sign_info = 1;                    
    required uint32 online_area_id = 2;                 
    required MatchingParameter matching_parameter = 3;  
    required bytes player_struct = 4;                   
    required string steam_id = 5;                       
    required uint32 is_red_sign = 6;                    
}

message GetSignResult {
    repeated SignInfo sign_info_without_data = 1; 
    repeated SignData sign_data = 2;
}

message RequestGetSignListResponse {
    required GetSignResult get_sign_result = 1;
}
```

While this is made up of a lot of different structures, the actual concept is fairly straightforward. The game sends a list of areas we want signs from (```SignDomainGetInfo```), and the server sends back any signs in those areas. A bit of optimization is done here by providing a list of signs the game already knows about in the ```already_have_signs``` list, the server will not send back the full data for these signs, just a confirmation they still exist in ```sign_info_without_data```.

Where things become interesting is in the ```MatchingParameter``` structure. The server doesn't send back all signs that exist in an area, only the ones the player can actually summon, the rules that dictate who they can summon are adjusted such that the player is matched with someone of similar skill and gets a fair gameplay experience. This structure is also used in a lot of other mechanics that require matchmaking, you will see it more below.

```protobuf
message MatchingParameter {
    required uint32 regulation_version = 1;         
    required uint32 unknown_id_2 = 2;                        
    required uint32 allow_cross_region = 3;               
    required uint32 nat_type = 4;                     
    required uint32 region = 5;                     
    required uint32 soul_level = 6;                                             
    required uint32 soul_memory = 7;                                           
    optional string unknown_string = 8;                   
    required uint32 clear_count = 9;                     
    required string password = 10;                                              
    required Covenant covenant = 11;                                            
    required uint32 weapon_level = 14;                                              
    optional string unknown_id_15 = 15;                   
}
```

So what are the actual rules that dictate who can play together? Essentially the server goes through every sign thats been created, and compares the MatchingParameter, provided when the sign was created, against the MatchingParameter provided with the ```RequestGetSignList``` request. The following rules are applied:

- Can match only if ```regulation_version```, which represents the version of the game, is equal.
- Can match only if ```region``` is the same, or ```allow_cross_region``` is true for both players.
- Can match only if ```nat_type```, which represents the ability for different games to connect together, is compatible.
- Can match only if ```password``` is the same for both players.
- Can match only if ```soul_level``` and ```weapon_level``` are within limits (described below), except where a ```password``` is supplied, or both players have a ```soul_level``` over 351.

The big one here is how we determine if a player has a soul level and weapon level within limits. The limits depend upon what type of gameplay we are trying to summon for - PvE has more relaxed limits that PvP for example.

For cooperative summoning the equation used is:

```cpp
Minimum = (Level * 0.9) - 10
Maximum = (Level * 1.1) + 10
```

Where Level is the soul level of the player requesting a list of signs.

For example, a player with a soul level of 125 can summon:

```cpp
Minimum = ((125 * 0.9) - 10) = 102
Maximum = ((125 * 1.1) + 10) = 147
```

Weapon level matching is slightly more complicated, an array of the maximum weapon level a player can match is checked against both the summoner and the owner of the sign (called the host).

```cpp
MaxLevels = [ 1, 2, 3, 4, 6, 7, 8, 9, 10, 10, 10 ]
CanMatch = (WeaponLevel <= WeaponLevelUpperLimit[HostWeaponLevel] && HostWeaponLevel <= WeaponLevelUpperLimit[WeaponLevel]);
```

Once the player has been provided a list of signs by the server, they will be spawned into the world, when the player activates them and attempts to summon them, they will send the following request to the server:

```protobuf
message RequestSummonSign {
    required uint32 map_id = 1;
    required uint32 online_area_id = 2;
    required SignInfo sign_info = 3;
    required bytes player_struct = 4;
}
```

- ```player_struct``` is a similar structure to the one we provided when we created a sign, except this time it represents the person summoning the sign.

```protobuf
message RequestSummonSignResponse {
    // Empty response.
}
```

Once the server receives this request it will send a push message to the original creator of the sign,

```protobuf
message SummonSignMessage {
    required uint32 player_id = 1;
    required string steam_id = 2;
    required SignInfo sign_info = 3;
    required bytes player_struct = 4;
}

message PushRequestSummonSign {
    required PushMessageId push_message_id = 1;     
    required SummonSignMessage message = 2;
}
```

This push message supplies the ```steam_id``` and ```player_struct``` of the player who summoned the sign.

At this point the game will determine if the sign is still valid, and if the original creator is still in a location they can be summoned from (e.g. they haven't walked into a boss battle or been invaded). If everything goes as expected, then both players establish a peer-to-peer connection via steams network API, this connection is then used for exchanging all real-time gameplay network state, the main server is no longer used.

If however something prevents the summoning from taking place, the game will send the following request to the server.

```protobuf
message RequestRejectSign {
    required uint32 player_id = 1;              
    required uint32 unknown_2 = 2;              
    required uint32 sign_id = 3;
    required bool unknown_4 = 4;                
    required bool unknown_5 = 5;                
}
```
<sup>Most of the values in this structure appear to be constant, so can mostly be ignored.</sup>

```protobuf
message RequestRejectSignResponse {
    // Empty response.
}
```

This will cause the server to send a push message back to the player that attempted to summon the sign.

```protobuf
message RejectSignMessage {
    required uint32 sign_id = 1;
    required uint32 player_id = 2;      
}

message PushRequestRejectSign {
    required PushMessageId push_message_id = 1;
    required RejectSignMessage message = 2;
}
```

At this point the game will show a "Summoning failed" message to the player and will refresh the signs again.

One interesting bit of trivia - When a sign is used or removed by the player who created it, the server sometimes sends a push message to all players's it has sent the sign info to in the past.

```protobuf
message RemoveSignMessage {
    required uint32 player_id = 1;
    required uint32 sign_id = 2;
}

message PushRequestRemoveSign {
    required PushMessageId push_message_id = 1;     
    required RemoveSignMessage message = 2;
}
```

You would expect that to remove the sign from the world so the player's cannot accidentally summon it right? Well actually the game seems to entirely ignore it. This might go some way to explaining the common problem of people trying and failing to summon signs during busy activity, because even if they've been used they stick around until the next refresh!

# Invasions

So we've explained how the cooperative PvE aspect of the game works, but what about the PvP aspect? This mainly comes from a feature that people either love or loath - Invasions. Invasions allow other players to invade your game without request and come and kill you, often at the least opportune times!

[![Invasions](/assets/images/posts/ds3os_7/invasions.png)](/assets/images/posts/ds3os_7/invasions.png)

So how do the invasions work? Well when a user uses one of the multiplayer items that allow them to invade another game - such as the [Red Eye Orb](https://darksouls3.wiki.fextralife.com/Red+Eye+Orb) - The game is placed into a state where it is constantly looking for other players they can invade.

This searching is accomplished by sending a request every few minutes to the main server.

```protobuf
message RequestGetBreakInTargetList { 
    required uint32 map_id = 1;
    required uint32 online_area_id = 2;
    required uint32 max_targets = 3;                       
    required MatchingParameter matching_parameter = 4;
    required uint32 unknown_5 = 5;                          
}
```

```protobuf
message BreakInTargetData {
    required uint32 player_id = 1;
    required string steam_id = 2;
}

message RequestGetBreakInTargetListResponse {
    optional uint32 map_id = 1;
    optional uint32 online_area_id = 2;
    repeated BreakInTargetData target_data = 3;
}
```

As you can guess this works the same as all the other request-list-of-things type requests. We provide an area we want to look at, and our matching parameters and the server sends us back any other players that match the rules.

The matching parameters are almost the same as PvE except for a couple of additional rules:

- Cannot match with players that have been invaded in the last 20 minutes, unless a [Dried Finger](https://darksouls3.wiki.fextralife.com/Dried+Finger) is in use.
- Cannot match with players that have a full game (There are a total of 6 players slots for pvp and pve, the number of slots allocated to each depends on the area).
- Cannot match with players who are not currently in an "embered" form.
- Games with more than one PvE summon in it are prioritized for invasion.

In addition to these rules the soul level range is adjusted to;

```cpp
Minimum = (Level * 0.9)
Maximum = (Level * 1.1) + 20
```

Or if you are in the mound maker covenant, you can invade slightly higher players:

```cpp
Minimum = (Level * 0.9)
Maximum = (Level * 1.15) + 20
```

For the targets of invasion the server uses the information supplied by the call to ```RequestUpdatePlayerStatus``` to do the matching.

Once the invader has a target player they send the server a request to invade:

```protobuf
message RequestBreakInTarget {
    required uint32 map_id = 1;
    required uint32 online_area_id = 2;
    required uint32 player_id = 3;
    required uint32 unknown_4 = 4;                   
}
```

```protobuf
message RequestBreakInTargetResponse {
    // Empty response.
}
```

At which point the server sends a push request to the player being invaded.

```protobuf
message PushRequestBreakInTarget { 
    required PushMessageId push_message_id = 1;
    required uint32 player_id = 2;                        
    required string steam_id = 3;       
    required uint32 unknown_4 = 4;                          
    required uint32 map_id = 5;
    required uint32 online_area_id = 6;
}
```

If the targeted player is no longer in a state where they can be invaded they send a rejection to the server.

```protobuf
message RequestRejectBreakInTarget {
    required uint32 player_id = 1;                         
    required uint32 unknown_2 = 2;                          
    required uint32 map_id = 3;
    required uint32 online_area_id = 4;
    required uint32 unknown_5 = 5;                          
}
```

Which results in the server sending a push message back to the invader telling them they have been rejected, and causing a "Failed to invade" message to be shown to the player and for the game to try a different target.

```protobuf
message PushRequestRejectBreakInTarget {
    required PushMessageId push_message_id = 1;            
    required uint32 player_id = 2;
    required uint32 unknown_3 = 3;                         
    required string steam_id = 4;                           
    required uint32 unknown_5 = 5;                         
}
```

# Remote Code Execution, Oh No!

Small segway, bare with me!

Now we get onto something very worrying. If the player is able to be invaded then a request is made to the server to provide them with the invader's information so a peer-to-peer connection can be established and gameplay can begin.

```protobuf
message RequestSendMessageToPlayers { 
    repeated uint32 player_ids = 1; 
    required bytes message = 2;
}
```

Ok, that's a weird function, it doesn't match any of the unambiguous requests the server normally takes? Why they chose to use this rather than a specific request I will never know. What this absolutely awful request does is it will send a serialized message directly to any player_id's provided - without any serialization or limitation on which players it can be sent to. Normally it's used to send the following message to the invader.

```protobuf
message PushRequestAllowBreakInTarget { 
    required PushMessageId push_message_id = 1;
    required uint32 player_id = 2;                         
    required bytes  player_struct = 3;
    required uint32 unknown_4 = 4;                         
}
```

However it can actually be sent at any time when connected to the server, and can be used to send absolutely any protobuf message type in the game.

Does that sound ***really dodgy***? Well it sure is! It's actually the vector used by the Remote Code Execution vulnerability that got the servers taken offline for the last 6 months. It can be used to send the exploit to -everyone- currently connected to the server, they don't need to be active in multiplayer, they just need to be connected. Very worrying! The actual exploit was due to the game not sanitizing any of the data blobs the server exchanges with players (such as the ```player_struct``` above), allowing for buffer overruns and RCE. This function takes a nasty RCE and turbo-charges it to be capable of exploiting tens of thousands of people at once.

A nice rewrite up of the exploit by the original author is available on GitHub [Here](https://github.com/tremwil/ds3-nrssr-rce) if you are interested.

# Covenant Visits

There is one final piece of the puzzle of the invasion and summoning mechanics. This is what the game calls "Visits", these are situations where players can be automatically summoned for coop or invasion automatically without requesting it. This function is tied to different covenants in the game, and mostly revolve around either protecting an area from anyone who enters it, or protecting players who have been invaded.

[![Covenant Visits](/assets/images/posts/ds3os_7/covenant_visits.png)](/assets/images/posts/ds3os_7/covenant_visits.png)

Unlike invasions or summoning, no network requests are sent by the player who will be visiting another world. Instead they mark themselves as able to visit when they send a ```RequestUpdatePlayerStatus```. Players who are in a situation where they can be visited then search for potential visitors.

There are 5 different covenants whos members can be potential visitors. They all have slightly different criteria and level ranges.

| Covenant | Criteria | Lower Range | Upper Range |
| -------- | -------- | ----------- | ----------- |
| Blue Sentinels | Player must be being invaded, friendly to player. | ```(Level * 0.9) - 15``` | ```(Level * 1.1) + 15``` |
| Blades of the Darkmoon | Player must be being invaded, friendly to player. | ```(Level * 0.9) - 15``` | ```(Level * 1.1) + 15``` |
| Watchdog of Farron | Player must be embered and inside the Farron Swamp area. Hostile to player. | ```(Level * 0.8) - 20``` | ```(Level * 1.1) + 0``` |
| Aldrich Faithful | Player must be embered and inside the Anor Londo area. Hostile to player. | ```(Level * 0.8) - 20``` | ```(Level * 1.1) + 0``` |
| Spears of the Church | Summoned as part of Halflight boss fight. Hostile to player. | ```(Level * 0.8) - 20``` | ```(Level * 1.1) + 0``` |

When a player meets the criteria for a given covenant they will send out the following request every 10 minutes or so.

```protobuf
message RequestGetVisitorList {
    required uint32 map_id = 1;   
    required uint32 online_area_id = 2;   
    required uint32 max_visitors = 3;
    required MatchingParameter matching_parameter = 4;
    required VisitorPool visitor_pool = 5;                      
    required uint32 unknown_6 = 6;                             
}
```

- ```visitor_pool``` Identifies the pool of potential visitors we pull from. Some covenants (such as Blue Sentinels and Blades of the Darkmoon) use the same pool.

```protobuf
message VisitorData {
    required uint32 player_id = 1;
    required string player_steam_id = 2;
}
```

```protobuf
message RequestGetVisitorListResponse {
    required uint32 map_id = 1;   
    required uint32 online_area_id = 2;  
    repeated VisitorData visitors = 3;
}
```

Once a candidate is found, the game will send a request to ask the visitor to join:

```protobuf
message RequestVisit {
    required uint32 map_id = 1;   
    required uint32 online_area_id = 2;   
    required VisitorPool visitor_pool = 3;        
    required uint32 player_id = 4;              
    required bytes  player_struct = 5;
}
```

- ```player_struct``` has the same kind of serialized data as you have seen for invasions and summons, the summoning players state.

```protobuf
message RequestVisitResponse {
    // Empty response.
}
```

The server then sends a push message to the player being summoned.

```protobuf
message PushRequestVisit {
    required PushMessageId push_message_id = 1;       
    required uint32 player_id = 2; 
    required string player_steam_id = 3;
    required bytes  player_struct = 4;                           
    required VisitorPool visitor_pool = 5;       
    required uint32 map_id = 6;
    required uint32 online_area_id = 7;
}
```

The player being summoned will then either establish a connection with the summoner and enter their world, otherwise they will send back a request to the server asking to reject the request.

```protobuf
message RequestRejectVisit {
    required uint32 player_id = 1;                                        
    required VisitorPool visitor_pool = 2;              
    required uint32 map_id = 3;
    required uint32 online_area_id = 4;
    required uint32 unknown_5 = 5;                      
}
```

```protobuf
message RequestRejectVisitResponse {
    // Empty response.
}
```

In this case the server will send a push request notifying the summoner of the rejection. That player will then go back to scanning for other potential candidates.

```protobuf
message PushRequestRejectVisit {
    required PushMessageId push_message_id = 1;    
    required uint32 player_id = 2;
    optional VisitorPool visitor_pool = 3;              
    required string steam_id = 4;                           
    required uint32 unknown_5 = 5;                         
}
```

All super easy right? It's almost the exact flow that's used for summoning and invasions, just with some slightly different data in the requests.

# Undead Matches

The final real-time multiplayer feature is the "Undead Arena". Unlike the other mechanics this one isn't diegetic, it is controlled via a pretty standard matchmaking menu. This menu allows the user to match up with a number of other players to compete in various PvP game modes.

[![Undead Matches](/assets/images/posts/ds3os_7/undead_matches.png)](/assets/images/posts/ds3os_7/undead_matches.png)

As with the other game modes the level matching has its own values, matching close to those used by summon-signs.

```cpp
Minimum = (Level * 0.9)
Maximum = (Level * 1.1) + 20
```

These limits can be removed by setting a matchmaking password in the undead match menu.

Network wise the undead matches have probably the most complex request flow in the game. It all starts when the user searches for a match, at which point the following request is made:

```protobuf
enum QuickMatchGameMode {
    Duel = 0;
    TwoPlayerBrawl = 1;
    FourPlayerBrawl = 2;
    SixPlayerBrawl = 3;
    TwoVersusTwo = 4;
    ThreeVersusThree = 5;
    TwoVersusTwo_Team = 6;
    ThreeVersusThree_Team = 7;
}

message RequestSearchQuickMatch {
    required QuickMatchGameMode mode = 1;
    repeated group Map_id_list = 2 {
        required uint32 map_id = 1;
        required uint32 online_area_id = 2;
    }
    required uint32 max_results = 3;                      
    required MatchingParameter matching_parameter = 4;
}
```

- ```Map_id_list``` contains a list of maps the user wants to play in. These are unique to undead matches, they are not part of the normal game world.
- ```mode``` is the pool of matches the player wants to search for results in. This doesn't perfectly match up with the game modes in the UI - for example the *_Team pools are used instead of the equivalent standard pool if the player sends a password that applies only to one team in a game mode. 

```protobuf
message QuickMatchData {
    required uint32 host_player_id = 1;      
    required string host_player_steam_id = 2; 
    required uint32 online_area_id = 3;             
}

message QuickMatchSearchResult {
    optional QuickMatchData data = 2;
    required uint32 unknown_3 = 3;                
    required uint32 unknown_4 = 4;                  
}

message RequestSearchQuickMatchResponse {
    repeated QuickMatchSearchResult matches = 1;
}
```

If no matches are found the player sends a request to create a new match:

```protobuf
message RequestRegisterQuickMatch {
    required QuickMatchGameMode mode = 1;
    required uint32 map_id = 2;
    required uint32 online_area_id = 3; 
    required MatchingParameter matching_parameter = 4;
    required uint32 unknown_5 = 5;                      
}
```

```protobuf
message RequestRegisterQuickMatchResponse {
    // Empty response.
}
```

If the player ever cancels matchmaking they will dispose of their match using the following request.

```protobuf
message RequestUnregisterQuickMatch {
    required QuickMatchGameMode mode = 1;
    required uint32 map_id = 2;
    required uint32 online_area_id = 3; 
    required uint32 unknown_4 = 4;                      
}
```

```protobuf
message RequestUnregisterQuickMatchResponse {
    // Empty response.
}
```

Matches are uniquely identified by the player who is hosting them. A player can only host a single match at a time.

If a match was found during the initial search the joining flow begins, which starts by a request asking the server to let us join the match.

```protobuf
message RequestJoinQuickMatch {
    required QuickMatchGameMode mode = 1;
    required uint32 character_id = 2;
    required uint32 host_player_id = 4;              
    required uint32 map_id = 5;                  
    required uint32 online_area_id = 6;             
    required uint32 unknown_7 = 7;                  
    required string password = 8;                  
}
```

```protobuf
message RequestJoinQuickMatchResponse {
    // Empty response.
}
```

The server then sends the host of the match a push message notifying them of the pending join.

```protobuf
message JoinQuickMatchMessage {
    required uint32 join_player_id = 1;                                        
    required string join_player_steam_id = 2;                                  
    required uint32 join_character_id = 3;                                     
    required uint32 online_area_id = 4;                                        
    required uint32 unknown_5 = 5;                                             
    required string password = 6;                                              
}

message PushRequestJoinQuickMatch {
    required PushMessageId push_message_id = 1;                  
    required JoinQuickMatchMessage message = 2;
}
```

The host can then either respond by accepting the join request.

```protobuf
message RequestAcceptQuickMatch {
    required QuickMatchGameMode mode = 1;
    required uint32 join_player_id = 4;                           
    required bytes  data = 5;                  
}
```

```protobuf
message RequestAcceptQuickMatchResponse {
    // Empty response.
}
```

Or by rejecting the join request.

```protobuf
message RequestRejectQuickMatch {
    required QuickMatchGameMode mode = 1;
    required uint32 map_id = 2;  
    required uint32 online_area_id = 3;
    required uint32 join_player_id = 4; 
    required uint32 unknown_5 = 5;              
}
```

```protobuf
message RequestRejectQuickMatchResponse {
    // Empty response.
}
```

While results in an appropriate push message being send to the joining player.

```protobuf
message AcceptQuickMatchMessage {
    required uint32 host_player_id = 1;  
    required string host_player_steam_id = 2;
    required bytes  metadata = 3;   
}

message PushRequestAcceptQuickMatch {
    required PushMessageId push_message_id = 1;                  
    required AcceptQuickMatchMessage message = 2;
}
```

```protobuf
message RejectQuickMatchMessage {
    required uint32 host_player_id = 1;                         
    required uint32 unknown_2 = 2;                              
}

message PushRequestRejectQuickMatch {
    required PushMessageId push_message_id = 1;                     
    required RejectQuickMatchMessage message = 2;
}
```

If the player is accepted into the game, the games establish a peer to peer connection and await for enough other players to join for the game to begin.

Once the game begins the host will first unregister the match using the ```RequestUnregisterQuickMatch``` shown above, followed by telling the server the game is starting and who is in it.

```protobuf
message RequestSendQuickMatchStart {
    required uint32 unknown_1 = 1;                 

    repeated group  Session_member_list = 2 {
        required uint32 player_id = 1;
        required uint32 character_id = 2;           
    }
}
```

```protobuf
message RequestSendQuickMatchStartResponse {
    // Empty response.
}
```

Once the game is complete, each player in the game will then send the result of the match to the server so their new ranks can be calculated.

```protobuf
enum QuickMatchResult{
    QuickMatchResult_Win = 0;
    QuickMatchResult_Lose = 1;
    QuickMatchResult_Draw = 2;
    QuickMatchResult_Disconnect = 3;                    
}

message QuickMatchRank {
    optional uint32 rank = 1;       
    optional uint32 xp = 2;         
}

message RequestSendQuickMatchResult {
    required QuickMatchGameMode mode = 1;
    required uint32 unknown_2 = 2;                      
    required QuickMatchResult result = 3;               
    required bool local_won = 4;                        
    required QuickMatchRank remote_rank = 5;            
    required QuickMatchRank local_rank = 6;             
    optional string unknown_7 = 7;                      
}
```

The server then responds with the players new rank.

```protobuf
message RequestSendQuickMatchResultResponse {
    required uint32 unknown_1 = 1;             
    required QuickMatchRank new_local_rank = 2;
}
```

Interestingly the rank just seems to be incremented from the local_rank sent in the initial request. You can likely just "cheat" your way to the highest rank by fiddling with the initial local_rank value, not that I recommend you do that.

# Final Words

[![It's Done](/assets/images/posts/ds3os_7/its-done-frodo.gif)](/assets/images/posts/ds3os_7/its-done-frodo.gif)


Well I hope you found this series interesting, I'm sorry the last couple of posts were basically over-complicated flow charts. Unless I can think of some other interesting tid-bits of information, this is the conclusion. We've gone all the way from finding out how to get the game to connect to a different server, to mapping out how all the game features work at a request level.

There are a handful of other features the main server is capable of, but most are either restricted administration functions, or things that are just plain uninteresting. Feel free to check out the current reversed [protobuf file](https://github.com/TLeonardUK/ds3os/blob/main/Protobuf/Frpg2RequestMessage.proto) if you want to see the other requests that are floating around.

Dark Souls 3 is certainly a technically interesting game, and its network services definitely have some bizarre quirks, not to mention alarming security vulnerabilities. If you haven't already though, you should pick up a copy of it, it's without a doubt one of best games of all time, and From Software deserve your support to continue producing these types of games (I need my fix!).

Hopefully they can resolve the security vulnerabilities and get the official servers back online soon, and we can all enjoy some jolly cooperation on much more populated servers!

<iframe src="https://store.steampowered.com/widget/374320/69024/" frameborder="0" width="646" height="190"></iframe>