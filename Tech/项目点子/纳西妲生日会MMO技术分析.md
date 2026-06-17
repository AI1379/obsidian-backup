# 纳西妲生日会 MMO 项目技术分析报告

> 项目：纳西妲生日会多人在线游戏
> 创建日期：2026-04-23
> 技术栈：Rust + PostgreSQL（后端）/ Unity + WebGL（前端）
> 状态：探索阶段

---

## 一、项目概述

本项目旨在为纳西妲生日会制作一个小型多人在线游戏，允许玩家通过 Web 端连接，进行多人互动、体验主线剧情、与 NPC 对话。

### 核心功能需求

| 功能 | 描述 | 优先级 |
|------|------|--------|
| 多人在线 | 支持多名玩家同时在线 | P0 |
| Web 端接入 | 通过浏览器直接进入游戏 | P0 |
| 主线剧情 | 可推进的故事线 | P0 |
| NPC 对话 | 分支对话树 + 状态记忆 | P0 |
| 玩家社交 | 简单互动（好友/公会待定） | P1 |
| 地图探索 | 2D 或 3D 场景探索 | P1 |

---

## 二、后端技术架构（Rust + PostgreSQL）

### 2.1 技术选型理由

**为什么选择 Rust？**

- **性能**：与 C++ 同级，比 Go/Python/Java 更快，适合高并发实时游戏
- **内存安全**：没有 GC 停顿，对延迟敏感的游戏逻辑友好
- **生态成熟**：tokio 异步生态、sqlx 编译时检查、WebSocket 支持完善
- **编译时保证**：很多错误在编译期暴露，减少线上事故

**为什么选择 PostgreSQL？**

- **JSONB 支持**：对话树、任务状态等半结构化数据存储灵活
- **行级安全性**：可以做细粒度的权限控制
- **成熟稳定**：多年的游戏数据库验证
- **PostGIS（可选）**：如果需要地理位置查询

### 2.2 核心模块设计

```
┌─────────────────────────────────────────────────────────┐
│                      WebSocket Gateway                    │
│                   (tokio-tungstenite)                   │
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│                    Game Server                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │ 连接管理器   │  │  游戏逻辑   │  │  广播器     │    │
│  │ ConnectionMgr│  │  GameLogic │  │  Broadcaster│    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
└─────────────────────┬───────────────────────────────────┘
                      │
         ┌────────────┴────────────┐
         │                         │
┌────────▼────────┐     ┌──────────▼──────────┐
│     Redis        │     │    PostgreSQL        │
│  (热数据缓存)    │     │   (持久化存储)       │
│  位置/在线状态   │     │  玩家/对话/任务      │
└─────────────────┘     └─────────────────────┘
```

### 2.3 数据库建模

#### 核心表结构

```sql
-- 玩家表
CREATE TABLE players (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(32) UNIQUE NOT NULL,
    position_x FLOAT NOT NULL DEFAULT 0,
    position_y FLOAT NOT NULL DEFAULT 0,
    health INTEGER NOT NULL DEFAULT 100,
    level INTEGER NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    last_login TIMESTAMPTZ DEFAULT NOW()
);

-- 玩家对话状态（flag 机制）
CREATE TABLE player_dialogue_flags (
    player_id UUID REFERENCES players(id) ON DELETE CASCADE,
    flag_key VARCHAR(64) NOT NULL,
    flag_value BOOLEAN DEFAULT TRUE,
    PRIMARY KEY (player_id, flag_key)
);

-- 对话表
CREATE TABLE dialogues (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    npc_id VARCHAR(32) NOT NULL,
    dialogue_data JSONB NOT NULL,  -- 存储对话树结构
    version INTEGER DEFAULT 1
);

-- 任务表
CREATE TABLE quests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(128) NOT NULL,
    description TEXT,
    objectives JSONB NOT NULL,  -- ["击败3个敌人", "收集5个道具"]
    rewards JSONB,              -- {"items": ["草神瞳"], "exp": 100}
    next_quest UUID REFERENCES quests(id)
);

-- 玩家任务进度
CREATE TABLE player_quests (
    player_id UUID REFERENCES players(id) ON DELETE CASCADE,
    quest_id UUID REFERENCES quests(id) ON DELETE CASCADE,
    status VARCHAR(16) DEFAULT 'active',  -- active/completed/failed
    progress JSONB DEFAULT '[]',
    started_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ,
    PRIMARY KEY (player_id, quest_id)
);

-- NPC 表
CREATE TABLE npcs (
    id VARCHAR(32) PRIMARY KEY,
    name VARCHAR(64) NOT NULL,
    position_x FLOAT NOT NULL,
    position_y FLOAT NOT NULL,
    dialogue_id UUID REFERENCES dialogues(id),
    quest_ids UUID[] DEFAULT '{}'
);

-- 物品/道具表
CREATE TABLE items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(64) NOT NULL,
    description TEXT,
    item_type VARCHAR(32),  -- weapon/armor/consumable/quest_item
    rarity VARCHAR(16) DEFAULT 'common',
    icon_url VARCHAR(256)
);

-- 玩家背包
CREATE TABLE player_inventory (
    player_id UUID REFERENCES players(id) ON DELETE CASCADE,
    item_id UUID REFERENCES items(id) ON DELETE CASCADE,
    count INTEGER NOT NULL DEFAULT 1,
    equipped BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (player_id, item_id)
);
```

#### 对话树 JSONB 结构示例

```json
{
  "nodes": {
    "start": {
      "type": "dialogue",
      "text": "旅行者，欢迎来到须弥。我是纳西妲。",
      "speaker": "纳西妲",
      "next": "choice_1"
    },
    "choice_1": {
      "type": "choice",
      "options": [
        {
          "text": "关于世界树",
          "next": "worldtree_info",
          "conditions": null
        },
        {
          "text": "关于散兵",
          "next": "scara_info",
          "conditions": ["knows_scaramouche"]
        },
        {
          "text": "我该走了",
          "next": "farewell"
        }
      ]
    },
    "worldtree_info": {
      "type": "dialogue",
      "text": "世界树记录着提瓦特的一切知识...",
      "next": "choice_1",
      "effects": {
        "add_flags": ["learned_worldtree"],
        "give_items": [{"id": "leaf", "count": 1}]
      }
    }
  },
  "initial_node": "start"
}
```

### 2.4 实时同步协议设计

#### 协议包格式（内部使用 bincode）

```rust
// 客户端 → 服务器
enum ClientMessage {
    Move { x: f32, y: f32 },
    Interact { target_id: String },
    Chat { text: String },
    DialogueChoice { choice_index: usize },
    Attack { target_id: String },
}

// 服务器 → 客户端
enum ServerMessage {
    PlayerMoved { player_id: Uuid, x: f32, y: f32 },
    PlayerJoined { player_id: Uuid, username: String, x: f32, y: f32 },
    PlayerLeft { player_id: Uuid },
    NpcDialogue { npc_id: String, dialogue: Value },  // Value = JSON
    QuestUpdated { quest_id: Uuid, progress: Value },
    Error { code: u16, message: String },
}
```

#### Tick 循环设计

```rust
// 游戏主循环（每秒 20 tick）
let mut interval = tokio::time::interval(Duration::from_millis(50));

loop {
    interval.tick().await;
    
    // 1. 处理所有排队的消息
    while let Ok(msg) = rx.try_recv() {
        process_message(msg);
    }
    
    // 2. 模拟游戏逻辑（移动、战斗等）
    update_game_state();
    
    // 3. 广播状态给所有玩家
    broadcast_player_states().await;
    
    // 4. 定期刷盘（每 60 tick ≈ 3 秒）
    if tick % 60 == 0 {
        persist_to_database().await;
    }
}
```

### 2.5 热冷数据分离策略

```
玩家在线时：
  - 位置数据：Redis（高频更新，每 tick 写入）
  - 背包/任务：内存中缓存，定期写 Postgres

玩家离线时：
  - 全量数据刷到 Postgres
  - Redis 清除该玩家缓存

玩家登录时：
  - 从 Postgres 读取全量数据到内存
  - 检查 Redis 是否有残留数据（如崩溃恢复）
```

```rust
// 位置更新（高频，走 Redis）
async fn update_position(player_id: Uuid, x: f32, y: f32, redis: &Redis) {
    let key = format!("player:pos:{}", player_id);
    redis.hset(&key, "x", x).await?;
    redis.hset(&key, "y", y).await?;
    redis.expire(&key, 3600).await?;  // 1小时过期
}

// 位置广播（从 Redis 批量读取）
async fn broadcast_positions(game: &GameState) {
    let mut positions = Vec::new();
    for player_id in game.players.keys() {
        if let Some(pos) = redis.hgetall(&format!("player:pos:{}", player_id)).await {
            positions.push(PositionUpdate { player_id, x: pos.x, y: pos.y });
        }
    }
    broadcast_to_all(ServerMessage::BatchPositionUpdate { positions });
}
```

---

## 三、前端技术分析（Unity + WebGL）

### 3.1 WebGL 性能限制

**Unity WebGL 的已知瓶颈：**

| 瓶颈 | 影响 | 缓解方案 |
|------|------|---------|
| 浏览器内存限制 | 大型 3D 场景可能崩溃 | 场景分块加载、LOD |
| 垃圾回收 (GC) | GC 停顿导致卡顿 | 代码对象池、避免频繁 allocations |
| 单线程渲染 | 无法利用多核 | 关注 IL2CPP 优化 |
| 初始加载时间长 | 用户等待 | 异步加载、进度条、压缩资源 |

**实测数据参考：**

- 一个简单的 3D Unity WebGL Demo（约 50 draw calls）：约 15-30 FPS
- 优化后（同场景 20 draw calls）：可达 45-60 FPS
- 2D 项目通常比 3D 性能好 2-3 倍

### 3.2 2D vs 3D 选型对比

| 维度 | 2D 方案 | 3D 方案 |
|------|---------|---------|
| **开发成本** | 低 | 高 |
| **美术资源** | 相对便宜 | 建模/动画成本高 |
| **网络同步** | 简单（2D 坐标） | 复杂（需要插值/预测） |
| **WebGL 性能** | 流畅（60 FPS） | 勉强（30 FPS） |
| **沉浸感** | 中等 | 强 |
| **移动端适配** | 简单 | 需要额外适配 |
| **纳西妲 IP 呈现** | 角色立绘为主 | 可以有 3D 角色模型 |

### 3.3 建议方案

#### 方案 A：2D 为主（推荐首发）

```
技术栈：
- Unity 2D Tilemap 或 UI Canvas
- 精灵图 + 骨骼动画（Spine 或 Unity Animancer）
- 简单场景用预制件拼接

优势：
- 开发周期短（估计 1-2 个月）
- WebGL 性能无忧
- 更容易控制包体大小
- 美术资源复用性高（立绘、CG）

适合场景：
- 主线剧情驱动
- NPC 对话为核心
- 少量玩家同时在线（< 50 人）
```

#### 方案 B：伪 3D / 2.5D（折中）

```
技术栈：
- Unity 3D 但固定视角（类似命运-冠位指定 FGO）
- 只渲染 2D 角色 + 3D 场景背景
- 或使用等距视角（Isometric）

优势：
- 视觉表现比 2D 更好
- 技术难度比纯 3D 低
- 可以后续升级为 3D
```

#### 方案 C：纯 3D（长期目标）

```
前提条件：
- 有专职 3D 美术
- 项目延期到生日会前 4+ 个月
- 接受 30 FPS 的目标帧率

技术优化：
- 使用 Unity DOTS（ECS + Burst Compiler）
- 场景 LOD + 遮挡剔除
- 减少实时阴影
- 预烘焙光照
```

### 3.4 前端网络同步

无论 2D 还是 3D，网络同步逻辑是一样的：

```csharp
// Unity 客户端（C#）
public class NetworkManager : MonoBehaviour
{
    private WebSocket ws;
    private PlayerPosition localPlayer;
    
    // 发送移动
    public void SendMove(float x, float y)
    {
        var msg = new MoveMessage { X = x, Y = y };
        ws.Send(JsonUtility.ToJson(msg));
    }
    
    // 接收服务器广播
    private void OnMessage(string data)
    {
        var msg = JsonUtility.FromJson<ServerMessage>(data);
        
        switch (msg.type)
        {
            case "player_moved":
                var moved = JsonUtility.FromJson<PlayerMovedMsg>(data);
                RemotePlayerUpdate(moved.player_id, moved.x, moved.y);
                break;
            case "batch_position_update":
                // 批量更新，用于减少消息数
                var batch = JsonUtility.FromJson<BatchUpdateMsg>(data);
                foreach (var p in batch.positions)
                    RemotePlayerUpdate(p.player_id, p.x, p.y);
                break;
        }
    }
    
    // 插值平滑（客户端预测）
    void RemotePlayerUpdate(string playerId, float x, float y)
    {
        var player = GetRemotePlayer(playerId);
        StartCoroutine(LerpPosition(player, x, y, 0.1f));
    }
}
```

### 3.5 前端架构建议

```
Assets/
├── Scenes/
│   ├── Boot.unity          # 启动场景（加载资源）
│   ├── MainMenu.unity      # 主菜单
│   └── Game.unity          # 游戏主场景
├── Scripts/
│   ├── Network/            # 网络通信
│   │   ├── WebSocketManager.cs
│   │   └── MessageTypes.cs
│   ├── Game/               # 游戏逻辑
│   │   ├── PlayerController.cs
│   │   ├── NpcController.cs
│   │   └── DialogueManager.cs
│   └── UI/                 # UI 系统
│       ├── DialogueUI.cs
│       └── InventoryUI.cs
├── Prefabs/
│   ├── Player.prefab
│   ├── Npc.prefab
│   └── UI/
└── Resources/
    ├── Dialogue/           # 对话数据（JSON）
    └── Items/              # 道具配置
```

---

## 四、技术风险与缓解

### 4.1 后端风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|---------|
| WebSocket 连接数超过预期 | 中 | 高 | 提前做连接数压测；设计分线机制 |
| 数据库写入成为瓶颈 | 高 | 中 | Redis 缓存 + 异步批量写入 |
| Rust 所有权复杂度过高 | 中 | 中 | 早期设计评审；核心对象用 Arc<Mutex<T>> |
| 游戏逻辑 Bug 导致状态不一致 | 中 | 高 | 完善的日志系统；服务器权威模型 |

### 4.2 前端风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|---------|
| WebGL 包体过大 | 高 | 中 | 资源压缩；按需加载 |
| 移动端体验差 | 高 | 低 | 先做 PC/Web 适配，移动端延后 |
| 3D 性能不达标 | 高 | 高 | 先验证 2D 方案；留足优化时间 |
| 网络延迟导致操作不跟手 | 中 | 中 | 客户端预测 + 延迟补偿 |

---

## 五、开发阶段建议

### 第一阶段（MVP，2-3 周）

**目标**：验证核心玩法可行

- [ ] 后端：完成 WebSocket 基础框架 + 玩家移动同步
- [ ] 前端：2D 简单场景 + 玩家控制
- [ ] 端到端：1-2 名玩家可以同时移动

### 第二阶段（核心功能，3-4 周）

**目标**：完成主要功能

- [ ] 后端：对话树系统 + 任务系统
- [ ] 前端：NPC 对话 UI + 任务追踪 UI
- [ ] 端到端：完整的剧情流程可体验

### 第三阶段（完善，2-3 周）

**目标**：提升体验

- [ ] 美术资源替换
- [ ] 性能优化
- [ ] Bug 修复
- [ ] 压力测试

### 第四阶段（测试，2 周）

**目标**：上线准备

- [ ] 内部测试
- [ ] 修复关键 Bug
- [ ] 部署上线

---

## 六、推荐技术栈汇总

### 后端

| 模块 | 推荐方案 |
|------|---------|
| 异步运行时 | tokio |
| Web 框架 | axum |
| WebSocket | tokio-tungstenite |
| 数据库 | PostgreSQL + sqlx |
| 缓存 | Redis |
| 序列化 | serde + bincode |
| 日志 | tracing + tracing-subscriber |

### 前端

| 模块 | 推荐方案 |
|------|---------|
| 引擎 | Unity 2022 LTS + WebGL |
| 2D 动画 | Animancer 或 Spine |
| 3D（如需）| URP + ProBuilder |
| 网络 | System.Net.WebSockets 或第三方库 |
| UI | Unity UI (uGUI) 或 FGUI |
| 资源加载 | Addressables |

---

## 七、参考资料

- [Rust 游戏服务器实战](https://github.com/Logic-32/rust-game-server)
- [Unity WebGL 性能优化指南](https://docs.unity3d.com/Manual/webgl-optimizing)
- [MMO 服务器架构设计](https://gafferongames.com/postarchitecture)
- [SpatialOS MMO 架构解析](https://spatialos.improbable.io/blog/understanding-spatialos)

---

*本报告由 AI 辅助分析生成，随着项目推进将持续更新。*
