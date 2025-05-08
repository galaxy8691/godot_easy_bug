# Godot 开发经验与常见错误记录

[English Version](README_EN.md)

这个仓库用于记录在使用 Godot 引擎进行游戏开发时遇到的常见问题、解决方案以及经验总结。

## 经验记录

### 1. 多人联机场景创建顺序问题

#### 问题描述
在使用 `MultiplayerSpawner` 进行多人联机开发时，场景创建顺序会影响游戏的正常运行。如果服务器创建场景的速度比客户端快，客户端可能会出现找不到 spawner 的错误。

#### 解决方案
应该遵循以下顺序：
1. 让客户端先创建好场景
2. 客户端场景创建完成后，再通知服务器创建场景
3. 确保 MultiplayerSpawner 在客户端已经准备就绪后再进行使用

#### 代码示例

客户端代码：
```gdscript
# 在客户端场景中
func _ready():
    # 确保这段代码只在客户端执行
    if not is_multiplayer_authority():
        # 通知服务器该客户端已完成场景创建
        MpManager.tell_server_scene_created.rpc(multiplayer.get_unique_id())
```

服务端代码：
```gdscript
# 在 MpManager（多人管理器）中
# 用于记录每个客户端场景创建状态的字典
var player_scene_created = {}

# 当所有客户端都准备就绪时发出的信号
signal server_can_create_scene

@rpc("any_peer", "call_local")
func tell_server_scene_created(peer_id: int):
    if multiplayer.is_server():
        # 记录该客户端已准备就绪
        player_scene_created[peer_id] = true
        print("player_scene_created", player_scene_created)
        
        # 检查所有客户端是否都已准备就绪
        for key in player_scene_created:
            if not player_scene_created[key]:
                return
        
        # 所有客户端都准备就绪，发出信号
        server_can_create_scene.emit()
```

#### 最佳实践
- 在网络通信中实现适当的同步机制
- 使用信号（signal）或 RPC 来确保场景创建的正确顺序
- 添加错误处理和重试机制
- 使用字典记录每个客户端的准备状态
- 只有在所有客户端都准备就绪后才开始服务器端的场景创建

### 2. RPC 无法直接传输 Object 对象的解决方案

#### 问题描述
在 Godot 的 RPC 系统中，无法直接传输 Object 类型的对象。这是因为 RPC 只能传输基本数据类型和一些内置类型。

#### 解决方案
有两种主要的解决方案：

1. 使用节点路径（NodePath）：
   - 将对象引用转换为节点路径字符串
   - 在接收端通过路径重新获取对象

2. 使用全局对象管理器（推荐）：
   - 创建一个 AutoLoad 单例作为对象管理器
   - 为每个对象分配唯一ID
   - 通过 ID 在管理器中查询对象

#### 代码示例

1. 使用节点路径的方式：
```gdscript
@rpc("any_peer")
func send_object_reference(node_path: String):
    do_something(get_tree().get_node(node_path))
```

2. 使用对象管理器的方式：
```gdscript
# ObjectManager.gd (AutoLoad 脚本)
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

# 使用示例
@rpc("any_peer")
func do_something_with_object(obj_id: int):
    var obj = ObjectManager.get_object(obj_id)
    do_something(obj)
```

#### 最佳实践
- 优先使用对象管理器方案，它提供更好的对象生命周期管理
- 在对象不再需要时，记得从管理器中移除
- 对于简单的场景树中的节点引用，可以使用节点路径方案
- 确保在使用节点路径时，路径是相对于接收者的正确路径
- 在传输大量对象时，考虑使用对象池来优化性能

## 许可证

本项目采用 MIT 许可证 - 查看 [LICENSE](LICENSE) 文件了解详情
