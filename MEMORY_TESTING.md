# 内存使用测试场景

本文档提供了测试 sys-botbase 内存限制的实际场景。

## 测试 1: 基础内存分配

当前配置下的内存分配测试：

```bash
# 测试小数据 peek (应该成功)
peek 0x1000 0x10000  # 64 KB

# 测试中等数据 peek (应该成功)
peek 0x1000 0x100000  # 1 MB

# 测试大数据 peek (应该成功，接近极限)
peek 0x1000 0x200000  # 2 MB

# 测试非常大的数据 peek (可能失败，超过可用堆)
peek 0x1000 0x300000  # 3 MB - 可能 malloc 失败
```

## 测试 2: 流式 peek (peekInfinite)

这些操作应该都成功，因为使用固定的 16KB 缓冲区：

```bash
# 使用 peekAbsolute (内部使用 peekInfinite)
peekAbsolute 0x7100000000 0x10000     # 64 KB
peekAbsolute 0x7100000000 0x1000000   # 16 MB
peekAbsolute 0x7100000000 0x10000000  # 256 MB - 仍然可以工作！
```

## 测试 3: game 命令

测试 NsApplicationControlData 分配（约 144 KB）：

```bash
# 这些应该都能成功
game icon
game version
game author
game name
```

## 测试 4: 多个并发分配

测试多个大分配是否会耗尽堆：

```bash
# 启动多个 peek 操作（通过多个连接）
# 每个 peek 1 MB，如果同时进行 3 个，可能会失败
```

## 预期结果

### 当前 HEAP_SIZE = 2.5 MB (0x280000)

| 操作 | 大小 | 预期结果 |
|-----|------|---------|
| peek | < 1 MB | ✅ 成功 |
| peek | 1-2 MB | ✅ 成功（紧张）|
| peek | > 2.3 MB | ❌ malloc 失败 |
| peekInfinite | 任意大小 | ✅ 成功 |
| poke | < 1 MB | ✅ 成功 |
| game 命令 | 144 KB | ✅ 成功 |

### 如果 HEAP_SIZE = 2 MB (0x200000)

| 操作 | 大小 | 预期结果 |
|-----|------|---------|
| peek | < 1 MB | ✅ 成功 |
| peek | 1-1.8 MB | ✅ 成功（紧张）|
| peek | > 1.8 MB | ❌ malloc 失败 |
| peekInfinite | 任意大小 | ✅ 成功 |
| poke | < 1 MB | ✅ 成功 |
| game 命令 | 144 KB | ✅ 成功 |

### 如果 HEAP_SIZE = 1 MB (0x100000)

| 操作 | 大小 | 预期结果 |
|-----|------|---------|
| peek | < 500 KB | ✅ 成功 |
| peek | 500 KB-800 KB | ⚠️ 可能成功 |
| peek | > 800 KB | ❌ malloc 失败 |
| peekInfinite | 任意大小 | ✅ 成功 |
| poke | < 500 KB | ✅ 成功 |
| game 命令 | 144 KB | ⚠️ 可能在某些情况下失败 |

## 内存耗尽时的表现

当 malloc 失败时：
- 函数会返回 NULL
- 可能导致程序崩溃（如果没有检查 NULL）
- 或者操作静默失败

## 建议

1. **保持 HEAP_SIZE = 2.5 MB** 是安全的选择
2. **可以尝试 2 MB** 如果需要节省内存
3. **对于大数据操作，使用 peekAbsolute/peekMain**（它们使用 peekInfinite）
4. **避免一次 peek 超过 1 MB** 的数据，除非使用 peekInfinite

## 实际使用建议

### 读取大块内存
```bash
# 推荐：使用 peekAbsolute (无大小限制)
peekAbsolute 0x7100000000 0x10000000

# 不推荐：直接 peek 大数据
# peek 0x1000 0x10000000  # 这会失败！
```

### 写入大块内存
```bash
# 分批写入
poke 0x0 <first_1MB>
poke 0x100000 <second_1MB>
poke 0x200000 <third_1MB>
```
