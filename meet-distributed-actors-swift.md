# Meet distributed actors in Swift

Presenters:
- Konrad 'ktoso' Malawski, Swift Team

Distributed actors extend the conceptual actor model to multiple processes, such as multiple devices or servers in a cluster.

Example app has player actors and the MainActor (UI?)

## Conceptual

Actor state isolation allows the compiler to guarantee that once an actor-based program compiles, it is free from low-level data races.
Apply the conceptual model to the example app game, reimagined as a distributed system. 
We can think of each device, node in a cluster, or process of an operating system as if it were an "independent sea of concurrency." 
We're able to synchronize information rather easily because they're sharing the same memory space. Distribution requires a few more restrictions.

By using distributed actors, we're able to establish a channel between two processes and send messages between them.

For all intents and purposes, distributed actors are as useful as local actors - with the difference that they're also ready to participate in remote interactions whenever necessary.
The ability to be potentially remote without having to change how we interact with a distributed actor is called "location transparency." Regardless of where a distributed actor is located, we can interact with it the same way. This enables us to move our actors to wherever they should be located without having to change their implementation.

## Convert an actor to a distributed actor

Let's start with a regular actor:

```swift
public actor BotPlayer: Identifiable {
    nonisolated public let id: ActorIdentity = .random
    
    var ai: RandomPlayerBotAI
    var gameState: GameState
    
    public init(team: CharacterTeam) {
        self.gameState = .init()
        self.ai = RandomPlayerBotAI(playerID: self.id, team: team)
    }
    
    public func makeMove() throws -> GameMove {
        return try ai.decideNextMove(given: &gameState)
    }
    
    public func opponentMoved(_ move: GameMove) async throws {
        try gameState.mark(move)
    }
}
```

Import the `Distributed` module that is new in Swift 5.7:

```swift
import Distributed

public actor BotPlayer: Identifiable {
// ...
```

Add the `distributed` keyword in front of the `actor BotPlayer` declaration:

```swift
import Distributed

public distributed actor BotPlayer: Identifiable {
// ...
```

This enables additional type checking and errors related to the new Distributed module.

Distributed actors must be used with an actor system. We need to declare what type of actor system this distributed actor is intended to be used with.
For now, we'll use the `LocalTestingDistributedActor System` that comes with the Distributed module.

We tell the compiler what system we're going to use use by either:
- Declaring a module-wide `DefaultDistributedActorSystem` typealias
- Declaring an `ActorSystem` typealias in the body of the specific actor

The latter is more specific, so we'll use it here.

```swift
import Distributed 

public distributed actor BotPlayer: Identifiable {
    typealias ActorSystem = LocalTestingDistributedActorSystem
// ...
```

The next error is about the `id` implementation. The ID property cannot be defined explicitly as it conflicts with a distributed actor synthesized property.

IDs are crucial in distributed actors. They are used to uniquely identify an actor in the entire distributed actor system that it's part of. They are assigned to the distributed actor system as the actor is initialized, and later managed by that system. As such, we cannot declare or assign the ID property manually - the actor system does that for us. Remove the manually-declared ID property.

The final error relates to the initializer. The compiler says the actorSystem property has not been initialized before use. This is another compiler-synthesized property that is part of every distributed actor. In addition to declaring the type of actor system that we want to use, we need to initialize the synthesized `actorSystem` property with some concrete actor system.

Generally, the right thing to do here is to accept an actor system in the initializer, and pass it through to the property. This way, we could pass in a different actor system implementation in tests to facilitate easy unit testing. We'll also have to pass an instance whenever we create a new bot player.

```swift
import Distributed 

public distributed actor BotPlayer: Identifiable {
    typealias ActorSystem = LocalTestingDistributedActorSystem
  
    var ai: RandomPlayerBotAI
    var gameState: GameState
    
    public init(team: CharacterTeam, actorSystem: ActorSystem) {
        self.actorSystem = actorSystem // first, initialize the implicitly synthesized actor system property
        self.gameState = .init()
        self.ai = RandomPlayerBotAI(playerID: self.id, team: team) // use the synthesized `id` property
    }
    
    public func makeMove() throws -> GameMove {
        return try ai.decideNextMove(given: &gameState)
    }
    
    public func opponentMoved(_ move: GameMove) async throws {
        try gameState.mark(move)
    }
}
```

Now we're done with declaration site errors. Time to address call-site errors.

```swift
if let opponent = opponent {
    waitForOpponentMove(true)
    Task {
        // inform the opponent about this player's move
        try await opponent.opponentMoved(move) /// Error: Only `distributed` instance methods can be called on a potentially remote distributed actor

        guard gameResult == nil else {
            // we're done here, the game has some result already
            return
        }

        // the game is not over yet
        // ask the opponent to make their move:
        let opponentMove = try await opponent.makeMove() /// Error: Only `distributed` instance methods can be called on a potentially remote distributed actor
        log("model", "Opponent moved: \(opponentMove)")
        try markOpponentMove(opponentMove)
    }
}
```

This is easily fixed by adding the `distributed` keyword to those functions in the `BotPlayer` distributed actor:

```swift
import Distributed 

public distributed actor BotPlayer: Identifiable {
    typealias ActorSystem = LocalTestingDistributedActorSystem
  
    var ai: RandomPlayerBotAI
    var gameState: GameState
    
    public init(team: CharacterTeam, actorSystem: ActorSystem) {
        self.actorSystem = actorSystem // first, initialize the implicitly synthesized actor system property
        self.gameState = .init()
        self.ai = RandomPlayerBotAI(playerID: self.id, team: team) // use the synthesized `id` property
    }
    
    public distributed func makeMove() throws -> GameMove {
        return try ai.decideNextMove(given: &gameState)
    }
    
    public distributed func opponentMoved(_ move: GameMove) async throws {
        try gameState.mark(move)
    }
}
```

This resolves the call-site errors, but now there are errors in the distributed actor: `Result type GameMove does not conform to the serialization requirement Codable.` This is because distributed method calls & their parameters & return values can cross network boundaries - so they must conform to the serialization requirement of the actor system. In this case, the actor system is using `Codable`. This is an easy thing to make Codable (in this example app) - we can make it `Codable` just by marking conformance.

## Convert distributed actor from local to remote system

Now we can move the bot player from the local system to a server-side Swift application, and resolve a remote reference to it from our mobile game.

For the actor, we only need to change the declared `ActorSystem` from the `LocalTestingDistributedActorSystem` to a `SampleWebSocketActorSystem` prepared for the sample app. The rest of the actor code is the same: 

```swift
import Distributed 

public distributed actor BotPlayer: Identifiable {
    typealias ActorSystem = SampleWebSocketActorSystem
  
    var ai: RandomPlayerBotAI
    var gameState: GameState
    
    public init(team: CharacterTeam, actorSystem: ActorSystem) {
        self.actorSystem = actorSystem // first, initialize the implicitly synthesized actor system property
        self.gameState = .init()
        self.ai = RandomPlayerBotAI(playerID: self.id, team: team) // use the synthesized `id` property
    }
    
    public distributed func makeMove() throws -> GameMove {
        return try ai.decideNextMove(given: &gameState)
    }
    
    public distributed func opponentMoved(_ move: GameMove) async throws {
        try gameState.mark(move)
    }
}
```

Next, resolve a remote bot player reference rather than creating one locally. Instead of initializing one locally, we attempt to resolve a remote reference:

**Creating a "local" bot**

```swift
let bot = BotPlayer(team: .rodents, actorSystem: ...)
```

**Resolving a "potentially remote" bot**

```swift
let sampleSystem: SampleWebSocketActorSystem

let opponentID: BotPlayer.ID = .randomID(opponentFor: self.id)
let bot = try BotPlayer.resolve(id: opponentID, using: sampleSystem) // resolve potentially remote bot player
```

This is synchronous code so it should return quickly. At the time of resolving the ID, the actual instance on the remote system might not exist yet. It will be created on the server-side system as the first message designated to this ID is received.

## TicTacFish Server and On-Demand Bot Players

First, create the WebSocket actor system in server mode, which makes it bind and listen to the port rather than connect to it. Wait until the system is terminated:

```swift
// TicTacFishServer / boot.swift
import TicTacFishShared
import SampleActorSystems

@main
struct Main {
    static func main() async throws {
        let system = try! SampleWebSocketActorSystem(mode: .serverOnly(host: "localhost", port: 8888))

        print("=== TicTacFish Server Running on: ws://\(system.host):\(system.port)")
        try await server.terminated
    }
}
```

Next, we need to handle the pattern of creating actors on demand as we receive messages addressed to IDs.

```swift
// TicTacFishServer / boot.swift
import TicTacFishShared
import SampleActorSystems

@main
struct Main {
    static func main() async throws {
        let system = try! SampleWebSocketActorSystem(mode: .serverOnly(host: "localhost", port: 8888))

        system.resolveCreateOnDemand { id in
            // Invoked whenever an ID known to live on this host, fails to resolve an existing actor.
            //
            // Here we can create new bot players "ad-hoc" as they are requeted for.
            // Subsequent resolves will return the same instance.
            if id.isBotPlayerId {
                return WebSocketBotPlayer(team: .rodents, actorSystem: system)
            }
            return nil // unable to create-on-demand for given id
        }

        print("=== TicTacFish Server Running on: ws://\(system.host):\(system.port)")
        try await server.terminated
    }
}
```

We had to do some up-front work to convert our bot player to a distributed actor, but moving it to the remote system was pretty straightforward. We didn't have to deal with any networking or serialization implementation details. All the heavy lifting was done for us by the distributed actor system. There aren't many hardened feature-complete implementations available just yet, but this ease of going distributed is something the feature strives for.

## Build a multiplayer experience: peer-to-peer experience

Consider a scenario where players are on a shared network but don't have good internet access. We could implement another actor system using local networking features offered by Network framework.

In this scenario, we can't work with made-up IDs - we'll be dealing with distributed actors that exist on other devices. In the domain of distributed actors, there is a common pattern and style of API actor system that offers strongly-typed APIs throughout the code: the receptionist pattern. Actors need to check in with it in order to become known and available for others to meet.

Here is a simple receptionist for this sample:

```swift
protocol DistributedActorReceptionist<ActorSystem: DistributedActorSystem> {
    func checkIn<Guest>(_ actor: Guest, tag: String?) async 
        where Guest: DistributedActor, Guest: Codable, Guest.ActorSystem == ActorSystem

    func listing<Act>(of type: Act.Type, tag: String?) async 
        -> AsyncCompactMapSequence<AsyncStream<any DistributedActor>, Guest> 
        where Guest: DistributedActor, Guest: Codable, Act.ActorSystem == ActorSystem
}
```

Now let's discover an actor to start a game:

```swift
// Matchmaking / Actor Discovery
HStack {
    ProgressView().padding(2)
    Text("Looking for opponent...")
}.task {
    await startMatchMaking()
}
func startMatchMaking() async {
    guard model.opponent == nil else { return }

    let listing = await localNetworkSystem
        .receptionist
        .listing(of: LocalNetworkPlayer.self, 
                tag: model.opposingTeam)
    for try await opponent in listing {
        model.opponent = opponent

        let first = checkPlayerGoesFirst(player, opponent: opponent)
        return try await opponent.startGameWith(opponent: player, 
                                                startTurn: !first)
    }
}
```

For this gameplay mode, we'll have to change the offline player implementation. We'll add a `humanSelectedField` async function, powered by a `@Published` value that is triggered when a human user clicks one of the fields.

```swift
// LocalNetworkPlayer.swift

public distributed actor LocalNetworkPlayer {
    public typealias ActorSystem = SampleLocalNetworkActorSystem

    public distributed func makeMove() async -> GameMove {
        // This is the new `humanSelectedField` async function
        let field = await model.humanSelectedField()

        defer { movesMade += 1 }
        return GameMove(field: field, /* ... */)
    }

    public distributed func makeMove(at position: Int) async -> GameMove {
        defer { movesMade += 1 }
        let move = GameMove(/* ... */)

        return await model.userMadeMove(move: move)
    }

    public distributed func opponentMoved(_ move: GameMove) async throws {
        try await model.markOpponentMove(move)
    }

    public distributed func startGameWith(opponent: OpponentPlayer, startTurn: Bool) async {
        await model.foundOpponent(opponent, myself: self, informOpponent: false)
        await model.waitForOpponentMove(shouldWaitForOpponentMove(myselfID: self.id, opponentID: opponent.id))
    }
}
```

## Combine different actor systems

We can use the WebSocket system to register device-hosted player actors in a server-side lobby system that will pair them up and act as a proxy for distributed calls between them. We might implement a GameLobby actor, with which device-hosted player actors are able to regsiter themselves. As devices enter the play online mode, they would discover the GameLobby using a receptionist, and call join on it. The GameLobby keeps track of available players and starts a game session when a pair of players has been identified:

```swift
/// Each node has a GameLobby instance and exchange information about available players

distributed actor GameLobby {
    /// In-progress sessions
    var gameSession: [GameSession] = [:]

    /// Players waiting for a game session
    var waitingPlayers: [CharacterTeam: [WebSocketPlayer]] = [:]

    /// A new player joined the lobby and we should find an opponent for it
    distributed func join(player: WebSocketPlayer, team: CharacterTeam) {
        // ...
    }

    /// As a session completes, remove it from the active game sessions
    distributed func sessionCompleted(_ session: GameSession) {
        // ...
    }
}
```

A GameSession would act as the driver of the game, polling moves and marking them in the server-stored representation of the game:

```swift
/// Keeps track of an active game between two players.
distributed actor GameSession {
    let lobby: GameLobby

    let player1: WebSocketPlayer
    let player2: WebSocketPlayer
    var gameState: GameState

    distributed func play() async throws {
        var nextPlayer = selectNextPlayer()
        while game.checkWin() == nil {
            let move = try await nextPlayer.makeMove()
            try gameState.mark(move)
        }

        try await lobby.sessionCompleted(self)
    }

    /// ...
}
```

More interestingly, we can scale the design horizontally. We can create more game session actors to serve more games concurrently on a single server. Thanks to distributed actors, we could even create a game session on other nodes in order to load balance the number of concurrent games across a cluster - if we had a Cluster actor system!

Distributed actors have access to a "feature-rich" Cluster Actor system library, implemented using SwiftNIO, and specialized for server-side data-center clustering. It applies advanced techniques for failure detection, and comes with its own implementation of a cluster-wide receptionist.

## Resources

Sample Code (+ docs): [TicTacFish: Implementing a game using distributed actors](https://developer.apple.com/documentation/swift/tictacfish_implementing_a_game_using_distributed_actors)

No docs! Not even API docs! But instead a link to the [Swift forum](https://forums.swift.org/c/server/distributed-actors/79)?
