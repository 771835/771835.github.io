---
title: Minecraft 数据包虚拟文件系统协议规范倡议──统一数据包持久化存储格式
date: 2026-08-14 03:26:57
tags:
  - minecraft
  - datapack
  - vfs
  - protocol
  - spec
categories:
  - Minecraft Datapack
---

## 背景与动机

当前 Minecraft 数据包（Datapack）对于游戏内数据存储普遍采用`data`与`scoreboard`指令的方式手动存储数据，这种方式存在以下问题：

1. **数据组织碎片化**：每个数据包自行决定 storage 路径和 scoreboard objective 命名，无统一规范，冲突频发。
2. **跨包数据交换困难**：数据包 A 无法以标准方式发现和访问数据包 B 的数据，只能靠硬编码 storage 名约定。
4. **数据生命周期不可追踪**：storage 中的数据无元信息（创建时间、修改时间、类型），调试时只能盲猜。

这种方式难以追踪和管理数据，且多个数据包之间难以统一的方式修改数据和交换数据，游戏外工具也难以方便地管理数据。

本文提出 **Minecraft 数据包虚拟文件系统协议（MCVFS）**，本质上是 **对 `data storage` 中 NBT 数据的标准化组织方案**，而非在
MC 内部实现一个真正的文件系统。协议包含两个层面：

1. **节点树序列化格式**：统一的 JSON 结构，用于工具链间数据交换与编译产物快照。
2. **原生 MCF API**：一组标准 `.mcfunction` 函数，在运行时通过 `data` 指令操作 storage 中的 VFS 节点树。

---

## 一、核心概念

### 1.1 什么是"虚拟文件系统"

MCVFS **不是**一个真正的文件系统。MC 没有文件系统 API，没有指针，没有系统调用。 MCVFS 的实质是 **一套约定**，安全的操作 `NBT`
以模拟真实的文件系统。
MCVFS 是全局单例，仅由首个加载的数据包初始化。

"文件"、"目录"、"路径"等术语原意指代真实文件系统中的内容，此处是 **对 NBT 节点的别称**，便于理解和规范数据布局。

### 1.2 与 data/scoreboard 指令的关系

MCVFS **不取代** `data` 和 `scoreboard` 指令，仅作为提供标准访问函数的数据存储与交互工具。

---

## 二、节点树数据结构

### 2.1 节点定义

每个 VFS 节点是一个 NBT Compound，统一结构如下：

```json5
{
  // "directory" | "file" | "symlink"
  "type": "directory",
  "stat": {
    "size": 0,
    // 游戏刻时间戳（tick），非 Unix 时间戳
    "mtime": 0,
    // 创建刻
    "ctime": 0,
    // 创建者标识
    "owner": "vfs"
  }
}
```

- 时间戳使用 **游戏刻**（`gametime`），通过 `execute store ... run time query gametime` 获取。
- `owner` 记录创建该节点的标识，用于多包场景下的权限边界。
- `size` 仅对 file 节点有意义，目录固定为 `0`。

### 2.2 目录节点

```json5
{
  "type": "directory",
  "stat": {
    "size": 0,
    "mtime": 0,
    "ctime": 0,
    "owner": "vfs"
  },
  "children": {
    "config.json": {
      "type": "file",
      "stat": {
        ...
      },
      "content": "{...NBT...}"
    },
    "players": {
      "type": "directory",
      "stat": {
        ...
      },
      "children": {
        ...
      }
    }
  },
  // _index 由 API 函数在增删子节点时自动维护
  "_index": ["a", "b"]
}
```

- `children` 为 NBT Compound，键为节点名（即"文件名"），值为子节点。
- 空目录的 `children` 为空 Compound `{}`。
- **NBT 路径对应文件路径**：`root.children.config.children.database` 即路径 `/config/database`。

### 2.3 文件节点

```json5
{
  "type": "file",
  "stat": {
    "size": 42,
    "mtime": 123456,
    "ctime": 123000,
    "owner": "vfs"
  },
  "content": "{\"hp\": 20,\"name\": \"Steve\"}"
}
```

- `content` 存储该文件节点的 NBT 值。NBT 类型不限，由写入方决定。
- `size` 为 `content` 序列化后的近似字节数，可选字段，为 `-1` 时表示未计算。
- `content` 的 NBT 存储类型由对应 MCVFS 具体实现为准，但应当支持后续提及 api。

### 2.4 符号链接节点

```json5
{
  "type": "symlink",
  "stat": {
    "size": 0,
    "mtime": 0,
    "ctime": 0,
    "owner": "vfs"
  },
  "target": "root.children.shared.children.config"
}
```

- `target` 存储的是 **NBT 路径**，而非"文件路径"。这是因为 MC 没有路径解析能力，直接存 NBT 路径可避免二次解析。
- 工具链在序列化为 JSON 时可将 NBT 路径转换为人类可读的文件路径。
- 循环链接检测由 API 函数在遍历时处理，超过最大深度（默认 16）即截断。

---

## 三、Storage 布局规范

### 3.1 命名约定

```
data storage vfs:file
```

MCVFS 的数据包实现应在命名空间 `vfs` 下维护一个 `file` storage。

### 3.2 顶层结构

```json5
// data storage vfs:file
{
  "root": {
    "type": "directory",
    "stat": {
      "size": 0,
      "mtime": 0,
      "ctime": 0,
      "owner": "vfs"
    },
    "children": {
      ...
    }
  },
  "meta": {
    "mcvfs_version": 1,
    // created 记录数据包首次加载的 gametime
    "created": 0,
  }
}
```

- `root` 为 VFS 根节点，必须是 directory 类型。
- `meta` 记录协议版本和元信息，工具链读取此字段判断兼容性。

### 3.3 路径与 NBT 路径的映射

| VFS 路径          | NBT 路径（相对于 storage 根）             |
|-------------------|-------------------------------------------|
| `/`               | `root`                                    |
| `/config`         | `root.children.config`                    |
| `/config/db.json` | `root.children.config.children."db.json"` |
| `/players/steve`  | `root.children.players.children.steve`    |

路径分隔符 `/` 对应 NBT 路径中的 `.children.`。

---

## 四、原生 MCF API

### 4.1 标准 API 函数清单

以下函数位于命名空间 `vfs` 下。

| 函数                 | 说明                             | 输入 (storage `vfs:in`)  | 输出 (storage `vfs:out`)     |
|----------------------|----------------------------------|--------------------------|------------------------------|
| `vfs:read`           | 读取文件 content                 | `path` (NBT path string) | `result` (NBT value)         |
| `vfs:write`          | 写入文件 content                 | `path`, `data` (NBT)     | `ok` (0b/1b)                 |
| `vfs:exists`         | 检查节点是否存在                 | `path`                   | `exists` (0b/1b)             |
| `vfs:delete`         | 删除文件节点                     | `path`                   | `ok` (0b/1b)                 |
| `vfs:stat`           | 读取 stat 信息                   | `path`                   | `result` (stat compound)     |
| `vfs:list`           | 列举目录子项名                   | `path`                   | `result` (list of strings)   |
| `vfs:mkdir`          | 创建目录节点                     | `path`                   | `ok` (0b/1b)                 |
| `vfs:walk`           | 递归遍历（递归函数）             | `path`                   | `result` (list of compounds) |
| `vfs:create_symlink` | 创建符号链接                     | `path`, `target`         | `ok` (0b/1b)                 |
| `vfs:resolve`        | 解析路径为实际节点，处理符号链接 | `path`                   | `result` (node compound)     |

- 由于上述函数均需要动态拼接，因此需 Minecraft 1.20.2 **及以上**版本

### 4.2 错误处理

API 通过 `vfs:error` 设置错误码代码 (数字形式) 以提示调用失败

错误码对应表：

| 错误码           | 代码 | 含义                       |
|------------------|------|----------------------------|
| `OK`             | 0    | 正常运行                   |
| `NOT_FOUND`      | 1    | 路径不存在                 |
| `NOT_A_DIR`      | 2    | 期望目录但节点为文件       |
| `NOT_A_FILE`     | 3    | 期望文件但节点为目录       |
| `ALREADY_EXISTS` | 4    | 创建时节点已存在           |
| `DIR_NOT_EMPTY`  | 5    | 删除非空目录               |
| `LOOP_SYMLINK`   | 6    | 符号链接循环（超过 16 层） |

### 4.3 初始化

VFS 在数据包首次加载时通过 `#load` 函数初始化：

```mcfunction
# vfs:load（在 load.json 中注册）
# 检查是否已初始化
execute if data storage vfs:file root run return 0
# 初始化根节点
data modify storage vfs:file set value {root:{type:"directory",stat:{size:0,mtime:0,ctime:0,owner:"vfs"},children:{}},meta:{mcvfs_version:1,created:0}}
# 记录创建时间
execute store result storage vfs:file meta.created int 1 run time query gametime
execute store result storage vfs:file root.stat.ctime int 1 run time query gametime
```

---

## 五、跨数据包访问

### 5.1 命名空间约定

MCVFS 统一存储于 `vfs:file`。跨包访问通过调用 VFS 提供的函数：

```mcfunction
# 读取其他数据包的配置
data modify storage vfs:in path "/shared/config"
function vfs:read
data get storage vfs:out result
```

### 5.2 权限边界

- `stat.owner` 字段标识节点创建者。
- **注意**：这只是 **约定级别**的保护。没有任何机制可以阻止 `data modify` 直接写入他人 storage。 该检查仅在通过标准 API
  调用时生效（通过主动设置目标以防止错误修改），直接使用 `data` 指令可绕过。

## 六、局限性
 
1. **1.20.2 以下版本不可用**：动态路径依赖宏，无法降级实现。
2. **NBT 路径深度限制**：建议目录深度不超过 8 层。
3. **无强制权限隔离**：`owner` 仅约定级别，`data modify` 可绕过。
4. **性能较差**：调用过程中检查较多。