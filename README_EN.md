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

### 2. Solutions for RPC Object Transmission Limitations

#### Problem Description
In Godot's RPC system, you cannot directly transmit Object type instances. This is because RPC can only transmit basic data types and certain built-in types.

#### Solutions
There are two main approaches:

1. Using Node Paths:
   - Convert object references to node path strings
   - Retrieve objects using the path on the receiving end

2. Using Global Object Manager (Recommended):
   - Create an AutoLoad singleton as object manager
   - Assign unique IDs to each object
   - Query objects through the manager using IDs

#### Code Examples

1. Using Node Paths:
```gdscript
@rpc("any_peer")
func send_object_reference(node_path: String):
    do_something(get_tree().get_node(node_path))
```

2. Using Object Manager:
```gdscript
# ObjectManager.gd (AutoLoad script)
extends Node

var object_map = {}
var next_id = 0

func register_object(obj: Object) -> int:
    next_id += 1
    object_map[next_id] = obj
    return next_id

func get_object(id: int) -> Object:
    return object_map.get(id)

func remove_object(id: int) -> void:
    object_map.erase(id)

# Usage Example
@rpc("any_peer")
func do_something_with_object(obj_id: int):
    var obj = ObjectManager.get_object(obj_id)
    do_something(obj)
```

#### Best Practices
- Prefer the object manager approach for better object lifecycle management
- Remember to remove objects from the manager when they're no longer needed
- Use node paths for simple scene tree node references
- Ensure paths are correct relative to the receiver when using node paths
- Consider using object pooling for performance when dealing with many objects

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details 