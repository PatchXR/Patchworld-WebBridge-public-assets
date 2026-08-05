# PatchWorld WebBridge API Reference

Welcome to the WebBridge API documentation for PatchWorld.
This library allows you to communicate directly with the PatchWorld game engine from your own Web interface (HTML/JS) displayed inside a Browser block.

## 🚀 Quick Start

1. Include the `patchworld-bridge.js` file in your HTML page:
```html
<script src="patchworld-bridge.js"></script>
```

2. Make sure your Javascript code runs asynchronously so you can query PatchWorld and await its responses easily.

> [!TIP]
> **Local Development:** You can test your HTML/JS interfaces locally in PatchWorld without hosting them on a web server!
> In the Browser block's URL field, use the `file://` protocol. 
> For example, if you place your files in the `StreamingAssets` folder, you can type:
> `file://PathToYour/index.html`

## 🔄 Lifecycle & Context Data

PatchWorld automatically provides context data to your Web UI (like the Player Name, User ID, World ID, etc.).
To manage state cleanly, the API provides **Lifecycle Hooks**.

### The Context Object
When the bridge connects, it passes a `context` object containing essential session data. You can access it anytime via `window.PatchWorld.context`:
```json
{
  "worldId": "current_world_uuid",
  "isOwner": true,
  "productId": "world_product_id",
  "localPlayer": { ... },
  "masterId": "player_id_of_current_master",
  "bridgeId": "1234/5678",   // The fullID of the PatchworldWebBridge block
  "browserId": "1234/9999"   // The fullID of the Browser block displaying this UI
}
```

### Connection Callbacks
Override these properties on `PatchWorld` to react to state changes:
- `PatchWorld.onBeforeData = function() { ... }` : Called when the bridge is ready, right before the initial `pxr.bridge.context` data is processed. Use this to clear your local state.
- `PatchWorld.onData = function() { ... }` : Called when the Unity bridge has established a connection and provided the `context` object.
- `PatchWorld.onWorldReady = function() { ... }` : Called when the world scene is fully loaded and, if in multiplayer, initial room state synchronization (`JoinState.Done`) is complete. If the bridge reconnects while the world is already ready, this callback triggers immediately.
- `PatchWorld.onDisconnected = function() { ... }` : Called when the block is disconnected or destroyed in PatchWorld.

### Player Callbacks & Data
Players in the room (including the LocalPlayer) are synchronized to the JavaScript side.
- **`PatchWorld.players`** : A dictionary of all connected players, keyed by their `playerRef`.
- **`PatchWorld.LocalPlayer`** : A direct reference to the Local Player object, or `null` if not found.
- `PatchWorld.onPlayerConnect = function(player) { ... }` : Called whenever a new player joins the room or is already in the room when the UI connects.
- `PatchWorld.onPlayerDisconnect = function(player) { ... }` : Called whenever a player leaves the room.
- `PatchWorld.onMasterChanged = function(masterId) { ... }` : Called when the Master Client of the room changes (e.g., the previous Master disconnects). `masterId` is the `playerId` of the new Master.

**Player Object Format:**
```json
{
  "playerRef": "player:0fb5acb6de123",
  "playerId": "0fb5acb6de123",
  "playerName": "DarkKiller666",
  "isLocal": true
}
```

**Global Access:**
The context is passed as an argument to `onData`, but you can also access it anywhere, at any time, via:
```javascript
console.log(window.PatchWorld.context.username);
```

### 🌍 Scene Callbacks & Data
You can react to blocks being spawned or destroyed in the scene. To avoid flooding the Bridge when large groups are spawned, these callbacks are **only triggered for root blocks** (blocks that are not children of another group).
- `PatchWorld.onBlockSpawned = function(data) { ... }` : Called when a root block or group is spawned in the world. `data` is an object: `{ fullId, name }`.
- `PatchWorld.onBlockRemoved = function(data) { ... }` : Called when a root block or group is deleted. `data` is an object: `{ fullId, name }`.

### 🎛️ Physical Block Interface (IO)
The WebBridge block in PatchWorld has physical inputs and outputs you can interact with from Javascript.
Override these properties on `PatchWorld` to receive physical inputs:
- `PatchWorld.onInterfaceMessageIn = function(txt) { ... }` : Receives text from the "Text In" physical connection.
- `PatchWorld.onInterfaceInputPartConnected = function(data) { ... }` : Called when a block is plugged into the "Parts In" connection. `data` is an object: `{ fullId, partId }`.
- `PatchWorld.onInterfaceInputPartDisconnected = function(data) { ... }` : Called when a block is unplugged.
- `PatchWorld.onInterfaceInputJoltA = function(value) { ... }` : Called when "Jolt In A" (the first Jolt receiver on the block) receives a physical trigger.
- `PatchWorld.onInterfaceInputJoltB = function(value) { ... }` : Called when "Jolt In B" (the second Jolt receiver on the block) receives a physical trigger.

Use these helper methods to trigger physical outputs:
- `await PatchWorld.interfaceMessageOut(txt)` : Sends text to the physical "Text Out" connection.
- `await PatchWorld.interfaceSendJoltA(value)` : Emits a physical jolt from "Jolt Out A" (the first Jolt emitter on the block).
- `await PatchWorld.interfaceSendJoltB(value)` : Emits a physical jolt from "Jolt Out B" (the second Jolt emitter on the block).
- `await PatchWorld.interfaceClearPartRefs()` : Disconnects all blocks from the "Parts Out" store.
- `await PatchWorld.interfaceAddPartRef(targetBlock, partID)` : Connects a specific block to "Parts Out".
- `await PatchWorld.interfaceRemovePartRef(targetBlock, partID)` : Disconnects a specific block from "Parts Out".

---

## 🛠️ API Usage

The API is `Promise`-based. The main shortcut function is `runCommand(commandName, ...args)`.

```javascript
// Example: Change the game's time scale
async function slowMotion() {
    try {
        // Calls the console command "SetTimeScale" with the value 0.5
        await runCommand("SetTimeScale", 0.5);
        console.log("Slow motion activated!");
    } catch (error) {
        console.error("Error:", error);
    }
}
```

### System State & Info
- **`GetWorldID`** : Returns the current World's ID.
- **`GetUserID`** : Returns the local Player's ID.
- **`GetBlockInfo <blockFullID>`** : Returns a JSON string with `IDName`, `DisplayName`, `AssetId`, `AssetUrl`, `AssetPath`, and `AssetUrlOrPath`.
- **`GetSerialization <blockFullID>`** : Returns the serialization string (state) of the target block. If the block is a SubPatch (Group), it returns the ShortSerialize version.
- **`SetSerialization <blockFullID> <partialSerializationData>`** : Applies partial or full state restore (deserialization) to the target block and synchronizes it across multiplayer.

---

## 🪪 Understanding Block IDs (`fullID`)

In PatchWorld, a `fullID` (or `PathID`) represents the hierarchical path of a block from the root of the world. It is composed of IDs separated by slashes (`/`).
For instance, a `fullID` of `"1234/5678"` means that block `5678` is located inside the sub-patch `1234`. If it is `"5678"`, it is at the root of the world.

> [!NOTE]
> **Players as Blocks**: You can target players in Transform commands using their `playerRef` (e.g., `"player:0fb5acb6de"`). When using a `playerRef` as a target or space, the `partID` maps to the player's BodyParts:
> - `0` = Root
> - `1` = RightHand
> - `2` = RightCTRL
> - `3` = LeftHand
> - `4` = LeftCTRL
> - `5` = Head
> - `6` = Feet

> [!TIP]
> **JavaScript Helper:**
> The `patchworld-bridge.js` API provides a convenient `GetParentDevice(fullID)` function to easily extract the parent group of any block.
> ```javascript
> let parentID = GetParentDevice("1234/5678/9999");
> // Returns "1234/5678"
> ```

---

## 🎮 PatchWorld Commands (Console API)
*(This list will be updated as new features are added)*

### 🧱 Blocks & Scene
- **`SpawnAssetAndGetID <assetID>`** : Spawn an asset (image/audio) and returns its new `fullID`.
  - `assetID`: The `assetID` referencing the asset.
  - Example: `SpawnAssetAndGetID "69050ba148fe87dfd347f65c"`
- **`CopyBlock <sourcePathID> <parentPathID>`** : Duplicates a block and returns its new `fullID`.
  - `sourcePathID`: The `fullID` of the block to copy.
  - `parentPathID`: (Optional) The `fullID` of a `SubPatch` where the new block should be spawned. Leave empty `""` to spawn it at the root of the world.
  - Example: `CopyBlock "1234/5678" ""`
- **`RemoveBlock <blockPathID> <delay>`** : Deletes a block from the world. Returns `"SUCCESS"` or an error string.
  - `blockPathID`: The `fullID` of the block to delete.
  - `delay`: (Optional, default `0`) Number of seconds to wait before the block is destroyed.
  - Example: `RemoveBlock "1234/5678" 1.5`

### 📐 Transforms (Position, Rotation, Scale)
All set/get methods below support `partID` (default `0`) if you need to manipulate a specific sub-mesh of the block.
They also support an optional relative space coordinate system via `blockAsSpace` and `blockAsSpacePartID`.
If you only want to change one axis, pass the magic number `PatchWorld.UNCHANGED` (or `-999999`) for the axes you want to leave alone!

- **`SetBlockPosition <blockFullID> <x> <y> <z> [partID] [blockAsSpace] [blockAsSpacePartID]`** : Moves a block.
  - Example: `SetBlockPosition "123" 0.0 1.0 PatchWorld.UNCHANGED` (Moves to Y=1, leaves X and Z where they are).
- **`GetBlockPosition <blockFullID> [partID] [blockAsSpace] [blockAsSpacePartID]`** : Returns a JSON string `{"x":0.0, "y":0.0, "z":0.0}`.
- **`SetBlockRotation <blockFullID> <x> <y> <z> [partID] [blockAsSpace] [blockAsSpacePartID]`** : Sets Euler rotation.
- **`GetBlockRotation <blockFullID> [partID] [blockAsSpace] [blockAsSpacePartID]`** : Returns Euler rotation JSON.
- **`SetBlockQuaternion <blockFullID> <x> <y> <z> <w> [partID] [blockAsSpace] [blockAsSpacePartID]`** : Sets Quaternion rotation.
- **`GetBlockQuaternion <blockFullID> [partID] [blockAsSpace] [blockAsSpacePartID]`** : Returns Quaternion JSON.
- **`SetBlockScale <blockFullID> <x> <y> <z> [partID]`** : Sets local scale. (Use `PatchWorld.UNCHANGED` to ignore axes).
- **`GetBlockScale <blockFullID> [partID]`** : Returns a JSON string with `x`, `y`, `z`.

### Block State
- **`SetBlockVisible <blockFullID> <isVisible> [partID]`** : Sets visibility. If `partID` is `-1` (default), affects all parts of the block.
- **`SetBlockLocked <blockFullID> <isLocked>`** : Locks/unlocks a block's position, preventing physical grabbing.

### ⏱️ Time and Engine
- **`SetTimeScale <float>`** : Changes the global time scale of the game. Example: `SetTimeScale 0.5`.

### 🍞 UI & Feedback
- **`ToasterMessage <string>`** : Displays a temporary text popup (Toaster) in front of the player. Example: `ToasterMessage Hello PatchWorld`.

### 📡 Wireless Connectivity (Wifi Jolts)
PatchWorld allows blocks to send wireless triggers (Jolts) across the room on named channels. The WebBridge API lets Javascript both broadcast and subscribe to these events.
- **`sendWirelessJolt(channelStr, value, [channelInt], [restrictionPathID])`** : Sends a wireless event (Jolt) to any block or script listening on the channel.
  - `channelStr`: The string name of the channel.
  - `value`: The numeric value to send.
  - `channelInt`: (Optional, default `0`) The integer channel ID.
  - `restrictionPathID`: (Optional, default `""`) Restrict the broadcast to a specific block hierarchy.
  - Example: `await PatchWorld.sendWirelessJolt("trigger", 1.0)`
- **`subscribeWifi(channelStr, [callback])`** : Subscribes the UI to listen for any wireless jolt fired on `channelStr`.

#### Wireless Callbacks
You can either pass a callback directly to `subscribeWifi("trigger", (val) => ...)` or override the global hook:
```javascript
PatchWorld.onWifiJolt = (channel, value) => {
    console.log(`Received jolt on ${channel}: ${value}`);
};
```

### 🔢 Variable System
PatchWorld's Variable System allows you to attach numeric values to blocks dynamically at runtime. The WebBridge provides an advanced **Subscription and Local Cache API** to read variables instantly without stuttering the framerate.
- **`subscribeVariable(varName)`**: Asks the engine to push updates for `varName`. Once subscribed, the local cache is populated.
- **`getVariable(varName, targetFullID, [partID])`**: **Synchronous.** Returns the value from the local cache. Returns `null` if not found or not subscribed.
- **`getAllWithVariable(varName)`**: **Synchronous.** Returns an array of `{fullId, partId, value}` from the local cache.
- **`setVariable(varName, targetFullID, value, [partID])`**: Asynchronous. Sets the variable on a specific block.
- **`removeVariable(varName, targetFullID, [partID])`**: Asynchronous. Removes the variable from a block.

#### Variable System Callbacks
You can override these hooks in JS to react to variable changes instantly:
```javascript
PatchWorld.onVariableSync = (varName, initialCacheArray) => { ... }
PatchWorld.onVariableAdded = (varName, fullID, partID, value) => { ... }
PatchWorld.onVariableChanged = (varName, fullID, partID, value) => { ... }
PatchWorld.onVariableRemoved = (varName, fullID, partID) => { ... }
```

### 🌐 Online Variables
- **`PostString <key> <value>`** : Saves a string to the PatchXR server at the specified key. Returns `"SUCCESS"` or `"ERROR_Budget_Exceeded_Try_Later"`.
- **`PostValue <key> <value>`** : Saves a numeric value (float/double) to the server at the specified key. Returns `"SUCCESS"` or `"ERROR_Budget_Exceeded_Try_Later"`.
- **`FetchString <key>`** : Retrieves a string from the server. Returns the string, `"ERROR_Not_Found"`, or `"ERROR_Budget_Exceeded_Try_Later"`.
- **`FetchValue <key>`** : Retrieves a numeric value from the server. Returns the number as a string, `"ERROR_Not_Found"`, or `"ERROR_Budget_Exceeded_Try_Later"`.
  > *Note:* These API calls consume the global online variable budget (max 10 operations per 10 seconds).

### ☁️ REST API (Server Queries & Thumbnails)
PatchWorld WebView runs with full network access. `PatchWorld` exports `API_URL` (`https://api.patchxr.io`) and async helper methods to query public server data directly from JavaScript:

> [!NOTE]
> **CORS Notice for Desktop Browser Testing**: When testing `index.html` locally on your PC (e.g., via `file://` or `localhost`), external `fetch()` calls to `api.patchxr.io` may fail with `HTTP 500` or CORS errors. Inside PatchWorld VR, WebView runs without CORS restrictions, so these queries will succeed!

- **`PatchWorld.API_URL`**: Base endpoint constant (`"https://api.patchxr.io"`).
- **`fetchAsset(assetId)`**: Returns metadata for a public library asset (e.g. `file`, `thumbUrl`, `category`).
- **`fetchUserProfile(userId)`**: Returns public player profile information (e.g. `username`, `avatar`, `bio`).
- **`fetchUserAssetLibrary(userId)`**: Returns the list of public devices/instruments published by a player.

#### Example Routes & Usage:
```javascript
// 1. Fetch thumbnail of a library asset
const asset = await fetchAsset("64f1a2b3c...");
console.log("Thumbnail URL:", asset.thumbUrl);

// 2. Fetch avatar thumbnail of a player
const profile = await fetchUserProfile("user_12345");
console.log("Player Avatar URL:", profile.avatar || profile.Avatar);

// 3. Fetch public asset list of a player
const library = await fetchUserAssetLibrary("user_12345");
console.log(`Player published ${library.length} public assets`);
```

---

## 💡 Advanced Examples

### Fire and Forget (Performance)
If you do not care about the result of a command and simply want to execute several commands as fast as possible without waiting for the Unity main thread to reply, you can omit the `await` keyword.

```javascript
// These commands will be queued instantly from the JS side
runCommand("SetTimeScale", 0.5);
runCommand("ToasterMessage", "Slowing down time!");
runCommand("SendWirelessJolt", 1.0, "trigger");
```
*Note: Because there is no `await`, Javascript will instantly move to the next line without waiting for the frame delay of the Unity ↔ JS handshake.*

### Waiting for multiple commands in parallel
If you want to execute several commands at once but still wait until all of them are processed by Unity, use `Promise.all`:

```javascript
await Promise.all([
    runCommand("PostValue", "score", 100),
    runCommand("PostString", "status", "playing")
]);
// Code here continues only after BOTH commands have returned a response
```

### Fetching information and reacting to it
```javascript
async function doSomethingComplex() {
    // 1. Fetch info (Assuming a 'GetPlayerCount' command exists)
    const playerCount = await runCommand("GetPlayerCount");
    
    // 2. Process the info in JS
    if (playerCount > 1) {
        // 3. Send a new command based on the result
        await runCommand("SetTimeScale", 1.0);
    }
}
```
