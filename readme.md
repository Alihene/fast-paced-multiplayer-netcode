# Writing fast-paced multiplayer netcode

Writing netcode for games is no easy task - especially for fast paced games like first person shooters where poorly written netcode can ruin matches. I wish to synthesize everything I've learned from many different articles and my own experience to make sure that any confusion about how to write netcode is resolved.

## The networking model
This guide will be focusing on using a client-server model instead of peer-to-peer or other alternatives since it is the most well suited model for our needs.

We are going to be using the UDP protocol for most parts as it has low overhead and reliability isn't a great concern (more on that later).

## Why is netcode so difficult?
Because of cheaters and latency. Almost all techniques described in this guide are to combat one or the other. We must assume at all times that the client **will** try to cheat and ruin the gameplay of others. This isn't a problem in singleplayer or co-operative games where the cheater is only ruining their own experience. However, in games that include player versus player combat, it is crucial to prevent and attempts of cheating made by the client.

We must also assume that the players will have an awful connection to the server and a very high latency and packet loss rate. Therefore, we will implement several techniques to mitigate this.

## Player movement
The first topic we will cover is player movement. Unfortunately, this is one of the more complex parts of multiplayer game development. There are many things to account for: cheaters, latency, interpolation, extrapolation, etc.

### Authoritative server
Due to cheating concerns, the server cannot trust the client to do any calculations. That means we cannot just calculate the player position on the client and periodically send the player's position to the server. Instead, we must send inputs to the server (e.g. key presses, mouse clicks, etc) and have the server authoritatively calculate the player's position before sending it back to the client, where the client will update its local position.

Unfortunately, there is a big problem to this approach: key presses are delayed according to the player's latency. This means that if the player has 200ms latency and presses the key 'A' to move left, they will only move left 200ms later. In a LAN environment or with very low latency, this isn't a problem. However as said earlier, we must assume that our players will be playing with horrible connections. With our current architecture, the game would be completely unplayable for them.

The solution to this is client side prediction

### Client side prediction
Instead of waiting for the server to receive the packet and send back the player's position, we can immediately process any inputs on the client and not wait for the server's response. This means that the client is in the future, the server is in the present.

### Server reconciliation
Predicting the player's position on the client is great, however the local simulation may not be accurate 100% of the time. The server still processes the inputs that the client sends and sends back an authoritatively calculated position. This means that if there are any inconsistencies we can set the local position to the "real" position.

However, there is a problem due to latency. Since the authoritative positions are from the past in relation to the client, we cannot just blindly apply them as it would cause the client's present position to be overridden by a past position.

The fix for this is to queue up inputs as they are processed on the client and keep an input counter which gets incremented each time a new input gets processed. This way, each input packet will be assigned a number. The server will send position packets with the last processed input number. Upon receiving these packets, the client will go in the queue and resolve all relevant inputs.

The code may look something like this:

```cpp
// Client

struct State {
    float x, y, z;
    int last_processed_input;
};

// Store inputs as vectors
struct PositionChange {
    float x, y, z;
    int input_id;
};

int last_processed_input = 0;

std::vector<PositionChange> position_changes;

void reconcile(const State &state) {
    player_position.x = state.x;
    player_position.y = state.y;
    player_position.z = state.z;

    int i = 0;
    while(i < inputs.size()) {
        auto &change = position_changes[i];

        if(change.input_id <= state.last_processed_input) {
            changes.pop_back();
        } else {
            player_position.x += change.x;
            player_position.y += change.y;
            player_position.z += change.z;
        }
    }
}
```
With something similar to this, your game's movement will be smooth and cheater proof! However, we're not quite done yet.

### Handling packet loss and jitter
A bad connection doesn't just mean high latency, it also means some packets arrive too early or too late, or don't arrive at all.

Since we are using UDP, packet loss is inevitable and some packets will be dropped. There is a simple fix for this: make every packet include a certain number of redundant inputs (for example the last 10). This means that if an input from the client gets lost, it will be included in the packets that come after it.

As for dealing with out of order packets (jitter), we can implement a jitter buffer. This means that incoming inputs are not immediately processed but queued up in the correct order. The downside to this is that there is added latency, so choose a sensible size for the buffer (e.g. 100ms) to keep gameplay smooth.

### Interpolation
One thing I haven't mentioned yet is that the client and server should update game state at a fixed update rate. This can be anything you want it to be: 20 Hz, 60 Hz, even 128 Hz like valorant. This is so that things like physics are kept as deterministic as possible on both the client and server. However, this introduces yet another problem: the render loop might run at a higher rate than the update loop. Players might be playing on 60 FPS on a game that runs at 20 Hz. This will lead to very choppy movement. The solution to this is interpolation. Instead of updating entities' positions every time you receive a snapshot, you smoothly animate what happens between two authoritative snapshots. So, if you receive an entity's position at tick 100 you wait until you get tick 101 and only then start animating what happens in between.

Finally, you have a robust system for player movement that handles both cheaters and latency!

## Hit registration
Hit registration is another important part of writing netcode. Games with bad hit registration netcode are very frustrating to play. It's important to resolve this issue so that your players can play fair games no matter how bad their connection is.

### Rollback netcode
The first thing we need to consider is that the client sees entities in the past due to latency. That means, if a player hits a moving target, they will miss because on the server, the target has a different position than the one the player sees. There is a simple fix for this: keep track of all recent player positions for a suitable amount of time on the server and keep track of the player's average latency. This way, we can estimate by how many ticks the player is lagging and pick a suitable position of the target from the past player positions. This way, hit detection is more accurate and games are fair.

## Additional Resources
These are resources I used myself to learn netcode and to write this article.

- [Fast-Paced Multiplayer](https://gabrielgambetta.com/client-server-game-architecture.html)
- [Source Multiplayer Networking](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking)
- [Server In-game Protocol Design and Optimization](https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods_in_Client/Server_In-game_Protocol_Design_and_Optimization)
- [Peeking Into Valorant's Netcode](https://technology.riotgames.com/news/peeking-valorants-netcode)
- [Networked Physics](https://gafferongames.com/post/introduction_to_networked_physics/)
- [Overwatch Gameplay Architecture and Netcode](https://youtu.be/W3aieHjyNvw)