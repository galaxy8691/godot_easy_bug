# Godot 开发经验与常见错误记录

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
