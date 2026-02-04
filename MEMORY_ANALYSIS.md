# sys-botbase 内存使用分析

## HEAP_SIZE 是什么？

`HEAP_SIZE` 定义了 **动态内存堆** 的大小，这是系统模块用于所有 `malloc()` 和 `calloc()` 分配的内存池。

### 当前配置
```c
#define HEAP_SIZE 0x00280000  // 2.5 MB = 2,621,440 bytes
```

### HEAP_SIZE 的作用
在 `__libnx_initheap()` 函数中，创建了一个静态数组作为堆：
```c
static u8 inner_heap[HEAP_SIZE];
```
这个数组是所有动态内存分配的来源。

## HEAP_SIZE 与什么有关？

HEAP_SIZE 的大小需要考虑以下因素：

### 1. 常驻内存分配
这些是程序运行期间持续占用的内存：

| 项目 | 大小 | 说明 |
|-----|------|------|
| linebuf | 16 KB | socket 命令读取缓冲区 |
| pfds | ~40 bytes × 5 | poll 文件描述符数组 |
| freeze map | ~16 KB | calloc(128, sizeof(FreezeBlock)) |
| 其他小分配 | ~20-30 KB | 各种临时变量 |
| **总计** | **~50-80 KB** | |

### 2. 临时大内存分配
这些是特定操作时的临时分配：

| 操作 | 内存分配 | 何时发生 |
|-----|---------|---------|
| `game icon/version/author/name` | ~144 KB | NsApplicationControlData |
| `peek <offset> <size>` | `size` bytes | 直接读取内存 |
| `poke <offset> <data>` | data 长度的一半 | 解析十六进制数据 |
| `peekInfinite` | 16 KB | 循环缓冲区 |
| `touch` 操作 | count × sizeof(HidTouchState) | 触摸事件 |
| `key` 操作 | count × sizeof(HiddbgKeyboardAutoPilotState) | 键盘事件 |

### 3. 内存使用峰值估算

在正常操作下：
- 基础占用：~50-80 KB
- game 命令峰值：~200-250 KB
- **可用于 peek/poke：约 2.3-2.4 MB**

## peek/poke 数据操作的大小限制

### peek 操作

有两种 peek 实现：

#### 1. `peek(offset, size)` - 直接分配模式
```c
u8 *out = malloc(sizeof(u8) * size);
```
- **理论上限**：HEAP_SIZE - 已用内存 ≈ 2.3-2.4 MB
- **实际建议**：< 2 MB 以保证稳定性
- **使用场景**：小到中等数据量（< 1 MB）

#### 2. `peekInfinite(offset, size)` - 流式读取模式
```c
u8 *out = malloc(sizeof(u8) * MAX_LINE_LENGTH);  // 16 KB
while (sizeRemainder > 0) {
    u64 thisBuffersize = sizeRemainder > MAX_LINE_LENGTH ? MAX_LINE_LENGTH : sizeRemainder;
    readMem(out, offset + totalFetched, thisBuffersize);
    // 输出数据...
}
```
- **理论上限**：几乎无限（仅受输出缓冲区限制）
- **缓冲区大小**：16 KB (MAX_LINE_LENGTH)
- **使用场景**：大数据量读取（可达 GB 级别）

### poke 操作

```c
u8* data = parseStringToByteBuffer(argv[2], &size);
poke(offset, size, data);
free(data);
```
- **理论上限**：HEAP_SIZE - 已用内存 ≈ 2.3-2.4 MB
- **实际限制**：
  - 命令行参数长度限制（通常 < 16 KB）
  - 十六进制字符串长度的一半即为字节数
  - 例如：32,768 个十六进制字符 = 16,384 bytes
- **建议上限**：< 1 MB 数据

## 可以进一步缩小 HEAP_SIZE 吗？

### 当前余量分析

| 场景 | 需要内存 | 当前 HEAP_SIZE (2.5 MB) | 余量 |
|-----|---------|------------------------|------|
| 基础操作 | ~50-80 KB | 2.5 MB | 充足 ✅ |
| game 命令 | ~200-250 KB | 2.5 MB | 充足 ✅ |
| peek 1 MB | ~1.05 MB | 2.5 MB | 充足 ✅ |
| peek 2 MB | ~2.05 MB | 2.5 MB | 紧张 ⚠️ |

### 进一步缩小的建议

#### 选项 1: 适度缩小到 2 MB (0x200000)
```c
#define HEAP_SIZE 0x00200000  // 2 MB
```
- **优点**：节省 512 KB
- **风险**：低
- **影响**：
  - game 命令：仍然安全
  - peek 操作：限制约 1.8 MB
  - 仍有足够余量

#### 选项 2: 激进缩小到 1.5 MB (0x180000)
```c
#define HEAP_SIZE 0x00180000  // 1.5 MB
```
- **优点**：节省 1 MB
- **风险**：中等
- **影响**：
  - game 命令：仍然安全
  - peek 操作：限制约 1.3 MB
  - 余量较小，可能在多个大分配时失败

#### 选项 3: 最小可行 1 MB (0x100000)
```c
#define HEAP_SIZE 0x100000  // 1 MB
```
- **优点**：节省 1.5 MB
- **风险**：高 ⚠️
- **影响**：
  - game 命令：可能在某些情况下失败
  - peek 操作：限制约 800 KB
  - 不推荐：余量不足

### 推荐方案

**推荐保持 2.5 MB 或适度缩小到 2 MB**

理由：
1. 2.5 MB 提供了很好的余量
2. 可以安全处理大多数合理的 peek/poke 操作
3. game 命令不会失败
4. 进一步缩小的收益递减（< 1 MB）

## 数据操作最佳实践

### 对于大数据读取
使用 `peekAbsolute` 命令（它内部使用 peekInfinite）：
```
peekAbsolute 0x<address> 0x100000  # 读取 1 MB，无内存压力
```

### 对于大数据写入
分批进行多次 poke 操作：
```bash
# 分 4 次写入，每次 256 KB
poke 0x0 <data1>
poke 0x40000 <data2>
poke 0x80000 <data3>
poke 0xC0000 <data4>
```

### 监控内存使用
如果需要，可以添加内存使用统计：
```c
// 获取当前堆使用情况
extern void* sbrk(intptr_t increment);
void* current_heap = sbrk(0);
```

## 总结

| 问题 | 答案 |
|-----|------|
| HEAP_SIZE 与什么有关？ | 所有动态内存分配 (malloc/calloc)，包括临时缓冲区、数据结构等 |
| 能否进一步缩小？ | 可以适度缩小到 2 MB，但不建议低于 1.5 MB |
| 当前 peek 支持多大数据？ | peek: ~2.3 MB (直接)；peekInfinite: 几乎无限 |
| 当前 poke 支持多大数据？ | ~2.3 MB (理论)；实际受命令行参数限制 ~16 KB |

**关键点**：使用 `peekInfinite` (通过 peekAbsolute 命令) 可以读取任意大小的内存而不受 HEAP_SIZE 限制！
