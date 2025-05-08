# Godot Development Experience and Common Issues

This repository is used to record common issues, solutions, and experience summaries encountered during game development using the Godot engine.

## Experience Records

### 1. Multiplayer Scene Creation Order Issue

#### Problem Description
When using `MultiplayerSpawner` in multiplayer development, the order of scene creation affects the normal operation of the game. If the server creates scenes faster than the clients, clients may encounter errors where they cannot find the spawner.

#### Solution
Follow these steps:
1. Let clients create their scenes first
2. After client scene creation is complete, notify the server
3. Ensure MultiplayerSpawner is used only after the client is ready

#### Code Examples

Client Code:
```gdscript
# In client scene
func _ready():
    # Ensure this code only runs on the client
    if not is_multiplayer_authority():
        # Notify server that this client has completed scene creation
        MpManager.tell_server_scene_created.rpc(multiplayer.get_unique_id())
```

Server Code:
```gdscript
# In MpManager (Multiplayer Manager)
# Dictionary to track each client's scene creation status
var player_scene_created = {}

# Signal emitted when all clients are ready
signal server_can_create_scene

@rpc("any_peer", "call_local")
func tell_server_scene_created(peer_id: int):
    if multiplayer.is_server():
        # Record that this client is ready
        player_scene_created[peer_id] = true
        print("player_scene_created", player_scene_created)
        
        # Check if all clients are ready
        for key in player_scene_created:
            if not player_scene_created[key]:
                return
        
        # All clients are ready, emit signal
        server_can_create_scene.emit()
```

#### Best Practices
- Implement proper synchronization mechanisms in network communication
- Use signals or RPCs to ensure correct scene creation order
- Add error handling and retry mechanisms
- Use a dictionary to track each client's ready status
- Only start server-side scene creation when all clients are ready

## Directory Structure

```
.
├── docs/                    # Documentation directory
│   ├── common-issues/       # Common issues records
│   ├── best-practices/      # Best practices
│   └── tutorials/           # Tutorials and guides
├── examples/                # Example code
└── README.md               # Project description
```

## How to Contribute

1. Fork this repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some feature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details 