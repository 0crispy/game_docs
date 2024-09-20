# Clean code

## Code style
- We use the [One True brace](https://en.wikipedia.org/wiki/Indentation_style#One_True_Brace) identation style:
```csharp
foreach (var connection in connections) {
    PlayerID playerID = connection.Key;
    HSteamNetConnection steamConnection = connection.Value;
    IntPtr[] messages = new IntPtr[MAX_MESSAGES_PER_FRAME];
    int messageCount = SteamGameServerNetworkingSockets.ReceiveMessagesOnConnection(steamConnection, messages, MAX_MESSAGES_PER_FRAME);
    if (messageCount > 0) {
        // Use braces even for single-statement blocks
        if (messageCount == MAX_MESSAGES_PER_FRAME) {*
            Debug.LogWarning("Too many messages are being sent!"); 
        }
        for (int i = 0; i < messageCount; i++) {
            SteamNetworkingMessage_t netMessage = Marshal.PtrToStructure<SteamNetworkingMessage_t>(messages[i]);
            byte[] message = new byte[netMessage.m_cbSize];
            Marshal.Copy(netMessage.m_pData, message, 0, message.Length);
            using Packet packet = new(message);
            ushort packetId = packet.ReadUShort();
            Server.Handle(packetId, packet, new PlayerID(netMessage.m_identityPeer.GetSteamID()));
        }
    }
}
```
## Design patterns
- Avoid using singletons. That is exactly why we have `ClientApp` and `ServerApp`.
These static classes contain static references to important classes, such as 
`ClientNetworkManager` or `LobbyManager`. 
- Use hierarchy for mono behaviours. For example, if I have a class called `Player`,
which changes variables of `PlayerMovement`, `PlayerMovement` should not change the
variables of `Player`. Meaning `Player` is the parent of `PlayerMovement`. 

## MonoBehaviour structure

1. Variables, grouped.
2. Main methods:
    1. Init (not all have this)
    2. Awake
    3. Start
    4. Update
    5. FixedUpdate
3. Other public methods.
4. Other private methods.
5. Structs or classes

```csharp
 public class Player : MonoBehaviour {
    // 1. 
    public PlayerID id;
    public Transform physicsBodyTransform;
    public Transform visualProxyTransform;
    public Transform visualProxyHead;
    // ...
    // 2.
    public void Init(PlayerID id) {
        this.id = id;
        // ...
    }
    void Awake() {
        // ...
    }
    void Update() {
        // ...
    }
    void FixedUpdate() {
        // ...
    }
    // 3.
    public bool IsPlayerLocal() {
        return this.id == ClientApp.myID;
    }
    public void SetPlayerPosition(Vector3 position) {
        // ...
    }
    // ...
    // 4.
    void SetRotation(Vector2 rotation) {
        // ...
    }
}
// 5.
public struct PlayerInputs {
    // ...
}
```
## Naming and comments
- Functions and variables should be named
in such a way that you wouldn't need any
comments to explain what their purpose is. Obviously to prevent naming
functions `SetPlayerPositionForLocalPlayerClientSideAndOverrideClientPrediction`,
use the shortest possible name and if it is not entirely clear, use a doc comment.
- Use normal C# naming conventions.
- In most cases avoid including the type of the variable in the name. For example,
instead of `snapshotsList`, just use `snapshots`.