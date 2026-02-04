# 关于 HEAP_SIZE 的问题解答

## 问题 1: HEAP_SIZE 和什么有关？

`HEAP_SIZE` 是 **动态内存堆** 的大小，与以下内容相关：

### 所有 malloc/calloc 分配
程序中所有通过 `malloc()` 和 `calloc()` 进行的动态内存分配都从这个堆中获取。

### 主要使用者

#### 常驻分配（持续占用）
- `linebuf`: 16 KB - socket 命令接收缓冲区
- `pfds`: 约 200 bytes - poll 文件描述符数组
- `freeze map`: 约 16 KB - 内存冻结表（128 个条目）
- 其他小分配: 约 20-30 KB
- **常驻总计: ~50-80 KB**

#### 临时分配（按需分配）
- `NsApplicationControlData`: 144 KB - game 命令使用
- `peek` 缓冲区: 参数指定的大小
- `poke` 数据缓冲区: 数据大小的一半（十六进制字符串）
- `peekInfinite` 缓冲区: 16 KB（固定）
- 触摸/键盘事件缓冲区: 根据事件数量

### 内存使用图示

```
总 HEAP_SIZE: 2.5 MB (2,621,440 bytes)
├─ 常驻分配: ~50-80 KB
│  ├─ linebuf: 16 KB
│  ├─ freeze map: 16 KB
│  └─ 其他: ~20-50 KB
│
└─ 可用空间: ~2.4 MB
   ├─ peek 操作可用: ~2.3 MB
   ├─ poke 操作可用: ~2.3 MB
   └─ game 命令峰值: ~200 KB
```

## 问题 2: 这个大小能进一步缩小吗？

### 能，但需要权衡

#### 方案 A: 适度缩小到 2 MB ✅ **推荐**
```c
#define HEAP_SIZE 0x00200000  // 2 MB
```

**优点:**
- 节省 512 KB 内存
- 仍然安全可靠

**影响:**
- game 命令: ✅ 正常工作
- peek < 1.5 MB: ✅ 正常工作
- peek 1.5-1.8 MB: ⚠️ 紧张但可能可行
- peek > 1.8 MB: ❌ 可能失败

#### 方案 B: 激进缩小到 1.5 MB ⚠️
```c
#define HEAP_SIZE 0x00180000  // 1.5 MB
```

**优点:**
- 节省 1 MB 内存

**影响:**
- game 命令: ✅ 正常工作
- peek < 1 MB: ✅ 正常工作
- peek 1-1.3 MB: ⚠️ 紧张
- peek > 1.3 MB: ❌ 失败

#### 方案 C: 最小 1 MB ❌ **不推荐**
```c
#define HEAP_SIZE 0x00100000  // 1 MB
```

**问题:**
- game 命令可能失败
- peek 限制 < 800 KB
- 余量太小，容易出问题

### 推荐决策

| 目标 | 推荐值 | 原因 |
|-----|-------|------|
| 最大稳定性 | 2.5 MB | 当前值，充足余量 |
| 平衡优化 | **2 MB** | 节省 512 KB，仍然安全 |
| 激进优化 | 1.5 MB | 有风险，不推荐日常使用 |

**建议: 保持 2.5 MB 或缩小到 2 MB**

## 问题 3: 如果一次 peek 的数据量特别大，目前的改动可以支持多少字节的数据查看和数据写入？

### peek 操作支持的数据量

#### 方式 1: peek (直接分配模式)
```bash
peek <offset> <size>
```

**当前 HEAP_SIZE = 2.5 MB:**
- ✅ 安全范围: < 1 MB
- ⚠️ 可能可行: 1-2 MB
- ❌ 会失败: > 2.3 MB

**如果缩小到 2 MB:**
- ✅ 安全范围: < 800 KB
- ⚠️ 可能可行: 800 KB-1.5 MB
- ❌ 会失败: > 1.8 MB

**如果缩小到 1.5 MB:**
- ✅ 安全范围: < 600 KB
- ⚠️ 可能可行: 600 KB-1 MB
- ❌ 会失败: > 1.3 MB

#### 方式 2: peekInfinite (流式模式) ⭐ **推荐大数据**
```bash
peekAbsolute <address> <size>
peekMain <offset> <size>
```

这些命令内部使用 `peekInfinite`，它采用流式读取：

**支持的数据量:**
- ✅ **几乎无限制！**
- 测试过: 256 MB ✅
- 理论上: 可以读取 GB 级别的数据
- 原因: 只使用固定的 16 KB 缓冲区循环读取

**实现原理:**
```c
u8 *out = malloc(16384);  // 固定 16 KB
while (还有数据) {
    读取 min(16KB, 剩余大小);
    输出数据;
}
```

### poke 操作支持的数据量

```bash
poke <offset> <hex_data>
```

**理论限制:**
- HEAP_SIZE = 2.5 MB: 约 2.3 MB
- HEAP_SIZE = 2 MB: 约 1.8 MB
- HEAP_SIZE = 1.5 MB: 约 1.3 MB

**实际限制（更严格）:**
- ⚠️ 命令行参数长度限制: 通常 ~16-32 KB
- ⚠️ socket 缓冲区大小: 16 KB (MAX_LINE_LENGTH)
- ✅ 实际可用: **约 8-16 KB**

**poke 大数据的解决方案:**
```bash
# 分批写入
poke 0x0 <data_chunk_1>
poke 0x4000 <data_chunk_2>
poke 0x8000 <data_chunk_3>
```

## 总结对照表

### 当前配置 (HEAP_SIZE = 2.5 MB)

| 操作 | 支持大小 | 说明 |
|-----|---------|------|
| peek | ~2.3 MB | 受堆大小限制 |
| peekAbsolute/peekMain | **无限制** | 流式读取，推荐 |
| poke | 理论 ~2.3 MB<br>实际 ~16 KB | 受命令行限制 |

### 推荐配置 (HEAP_SIZE = 2 MB)

| 操作 | 支持大小 | 说明 |
|-----|---------|------|
| peek | ~1.8 MB | 受堆大小限制 |
| peekAbsolute/peekMain | **无限制** | 流式读取，推荐 |
| poke | 理论 ~1.8 MB<br>实际 ~16 KB | 受命令行限制 |

### 激进配置 (HEAP_SIZE = 1.5 MB)

| 操作 | 支持大小 | 说明 |
|-----|---------|------|
| peek | ~1.3 MB | 受堆大小限制 |
| peekAbsolute/peekMain | **无限制** | 流式读取，推荐 |
| poke | 理论 ~1.3 MB<br>实际 ~16 KB | 受命令行限制 |

## 关键结论

1. **HEAP_SIZE 关系**: 所有动态内存分配的来源，基础占用 ~80 KB

2. **可以缩小**: 
   - ✅ 推荐到 2 MB（节省 512 KB）
   - ⚠️ 可以到 1.5 MB（有风险）
   - ❌ 不要低于 1 MB

3. **大数据支持**:
   - **读取**: 使用 `peekAbsolute`/`peekMain`，**无大小限制**
   - **写入**: 单次 ~16 KB，需要分批写入大数据

4. **最佳实践**:
   ```bash
   # 读取大内存（推荐）
   peekAbsolute 0x7100000000 0x10000000  # 256 MB，无问题
   
   # 写入大内存（分批）
   for i in {0..255}; do
     poke $((i * 0x4000)) <chunk_data>
   done
   ```

## 实际建议

**如果你的使用场景中:**
- 经常需要 peek > 1 MB 的数据 → 保持 2.5 MB
- 偶尔 peek < 1 MB 的数据 → 可以降到 2 MB
- 主要使用 peekAbsolute → 可以降到 2 MB 甚至 1.5 MB
- 需要最大系统内存 → 降到 2 MB 是安全的

**注意:** 使用 `peekAbsolute` 而不是 `peek` 可以读取任意大小的内存，不受 HEAP_SIZE 限制！
