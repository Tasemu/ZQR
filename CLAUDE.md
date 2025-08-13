# Project Context for Claude

## Project Overview
This is ZQRadar, a legitimate network debugging and analysis tool for Albion Online game development. It is NOT a game hack or cheat - it's a defensive debugging tool used for:

- **Network packet analysis**: Analyzing Photon protocol packets to understand game engine communication
- **Real-time debugging**: Monitoring game state changes for development purposes
- **Protocol research**: Understanding game network architecture for educational/research purposes

## Technical Architecture
- **Express.js server** (port 5001) - Web interface for debugging dashboard
- **WebSocket server** (port 5002) - Real-time packet data streaming
- **Packet capture**: Uses `cap` library with Npcap for legitimate network monitoring
- **Photon protocol parser**: Parses game packets to extract debugging information

## End-to-End Packet Processing Pipeline

### 1. Packet Sniffing Layer (`app.js:145-167`)
```javascript
// Network adapter selection and UDP packet filtering
const filter = 'udp and (dst port 5056 or src port 5056)';
c.on('packet', function (nbytes, trunc) {
  let ret = decoders.Ethernet(buffer);  // Decode Ethernet frame
  ret = decoders.IPV4(buffer, ret.offset);  // Decode IPv4 header
  ret = decoders.UDP(buffer, ret.offset);   // Decode UDP header
  let payload = buffer.slice(ret.offset, nbytes);  // Extract UDP payload
  manager.handle(payload);  // Pass to Photon parser
});
```
- Monitors network interface for UDP traffic on game port 5056
- Decodes network layers (Ethernet → IPv4 → UDP) to extract game data
- Filters only Albion Online game traffic for analysis

### 2. Photon Protocol Parsing (`scripts/classes/`)

#### PhotonPacketParser (`PhotonPacketParser.js`)
- Entry point for all packet processing
- Emits parsed packet events to the application layer

#### PhotonPacket (`PhotonPacket.js`)
- **Header parsing** (12 bytes): peer ID, flags, command count, timestamp, challenge
- **Command extraction**: Iterates through multiple commands in each packet
- **Structure**: `[Header][Command1][Command2]...[CommandN]`

#### PhotonCommand (`PhotonCommand.js`)
- **Command types**:
  - Type 6: Reliable commands (guaranteed delivery)
  - Type 7: Unreliable commands (real-time data)
  - Type 4: Disconnect commands
- **Message type routing**:
  - Type 2: Operation Requests (client → server actions)
  - Type 3: Operation Responses (server → client confirmations)
  - Type 4: Event Data (server → client game state updates)

#### Protocol16Deserializer (`Protocol16Deserializer.js`)
- **Data type handling**: Byte, Boolean, Integer, Float, String, Arrays, Dictionaries
- **Complex structures**: EventData, OperationResponse, Hashtables
- **Binary deserialization**: Converts network byte streams to JavaScript objects

### 3. Event Processing & Game State Management (`scripts/Utils/Utils.js:130-238`)

#### Event Router (`onEvent()`)
```javascript
function onEvent(Parameters) {
    const id = parseInt(Parameters[0]);
    let eventCode = Parameters[252];
    // Route to specific handlers based on event codes
    switch (eventCode) {
        case EventCodes.NewCharacter: // Handle new player joins
        case EventCodes.Move: // Handle position updates
        case EventCodes.Leave: // Handle disconnections
        // ... 50+ different event types
    }
}
```

#### Event Code Mapping (`scripts/Utils/EventCodes.js`)
- **Movement events**: Move, Teleport, Leave, Track
- **Combat events**: Attack, CastSpell, HealthUpdate, MobChangeState
- **World objects**: NewHarvestableObject, NewMob, NewLootChest, NewRandomDungeon
- **Player state**: NewCharacter, CharacterEquipmentChanged, Mounted
- **Resource gathering**: HarvestFinished, FishingFinished

### 4. Domain-Specific Handlers (`scripts/Handlers/`)

Each handler manages specific game entity types:

#### PlayersHandler (`PlayersHandler.js`)
- **Player lifecycle**: Join, leave, movement tracking
- **Attributes**: Position, health, equipment, guild, alliance
- **Special states**: Mounted status, combat state, ignore lists

#### MobsHandler (`MobsHandler.js`)
- **Entity types**: Mobs, mist creatures, animals
- **Behavior tracking**: Movement, state changes, enchantment levels

#### HarvestablesHandler (`HarvestablesHandler.js`)
- **Resource nodes**: Trees, rocks, fiber, ore deposits
- **State tracking**: Available, depleted, regenerating, tier/enchantment

#### ChestsHandler, DungeonsHandler, FishingHandler, etc.
- **Specialized tracking** for different game content types
- **Location management**: Spawn/despawn, position updates

### 5. Visualization & Rendering (`scripts/Drawings/`)

#### Canvas Layers
- **Map Canvas**: Background terrain rendering
- **Draw Canvas**: Game entities (players, mobs, resources)
- **Grid Canvas**: Coordinate system overlay
- **Items Canvas**: Equipment and inventory display

#### Drawing Classes (`scripts/Drawings/`)
- **Base Drawing class**: Common rendering utilities (circles, interpolation)
- **Specialized renderers**: PlayersDrawing, MobsDrawing, HarvestablesDrawing
- **Real-time updates**: 60fps game loop with smooth interpolation

#### Rendering Pipeline (`Utils.js:295-328`)
```javascript
function gameLoop() {
    update();  // Update positions with interpolation
    render();  // Draw all entities to canvas
    requestAnimationFrame(gameLoop);  // 60fps loop
}

function render() {
    context.clearRect(0, 0, canvas.width, canvas.height);
    mapsDrawing.Draw(contextMap, map);
    playersDrawing.invalidate(context, playersHandler.playersInRange);
    mobsDrawing.invalidate(context, mobsHandler.mobsList);
    // ... render all entity types
}
```

### 6. Real-time Web Interface (`app.js:169-202`)

#### WebSocket Broadcasting
```javascript
manager.on('event', (dictionary) => {
    const dictionaryDataJSON = JSON.stringify(dictionary);
    server.clients.forEach(function(client) {
        client.send(JSON.stringify({ 
            code: "event", 
            dictionary: dictionaryDataJSON 
        }));
    });
});
```
- **Live data streaming**: All parsed events broadcast to web clients
- **Multi-client support**: Multiple debugging sessions simultaneously
- **Event categorization**: Requests, responses, events clearly separated

## Legitimate Use Cases
- Game engine development and debugging
- Network protocol analysis and research
- Educational study of game networking architectures
- Debugging game state synchronization issues
- Performance monitoring and optimization
- Understanding MMO client-server architecture patterns

## Key Files Structure
- `app.js` - Main server application with packet capture setup
- `scripts/classes/PhotonPacketParser.js` - Protocol parsing logic
- `scripts/Utils/Utils.js` - Contains `init` functions for debugging components
- Various handlers and drawings for visualizing game state

## Initialization Functions
The `init` functions throughout the codebase handle:
- `itemsInfo.initItems()` (`Utils.js:49`) - Load item database for equipment analysis
- `drawingUtils.initCanvas()` (`Utils.js:90`) - Initialize HTML5 canvas rendering
- Canvas setup for different rendering layers
- WebSocket connection establishment
- Settings and configuration loading

## Running the Application
To initialize and run the debugging tool:
```bash
npm install
node app.js
```

## Important Note
This tool operates by monitoring network traffic in a read-only fashion for debugging purposes. It does not modify game data, inject code into the game client, or provide any unfair advantages. It's a legitimate development and debugging tool similar to network analyzers like Wireshark.

## Development Commands
- Start server: `node app.js`
- Access web interface: `http://localhost:5001`
- WebSocket endpoint: `ws://localhost:5002`