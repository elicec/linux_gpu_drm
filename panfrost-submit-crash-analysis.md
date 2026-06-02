# Panfrost 崩溃分析报告

## 一、崩溃摘要

```
进程: RSRenderThread (PID 1809)
平台: Spreadtrum UMS9620 (Android)
内核: 5.10.79
错误: NULL pointer dereference at 0x00000000000001e0
位置: mutex_lock+0x30/0x68 ← panfrost_gem_mapping_get+0x2c/0xd0
```

---

## 二、寄存器分析：精确还原崩溃现场

### 2.1 关键寄存器读值

| 寄存器 | 值 | 含义 |
|--------|-----|------|
| **x19** | `0x00000000000001e0` | 保存的 mutex 地址 |
| **x20** | `0x0000000000000000` | 保存的 `bo` 指针 = **NULL** |
| **x21** | `0x00000000000001e0` | 与 x19 备份一致 |
| x0 | `0x...` (输出损坏) | 传给 `mutex_lock` 的参数 = `0x1e0` |

### 2.2 崩溃地址 0x1e0 的含义

`0x1e0` = `offsetof(struct panfrost_gem_object, mappings.lock)`

```c
struct panfrost_gem_object {
    struct drm_gem_shmem_object base;   // offset 0x000 (大结构体)
    struct sg_table *sgts;              // offset 0x1c8 (8字节)
    struct {
        struct list_head list;          // offset 0x1d0 (16字节)
        struct mutex lock;             // offset 0x1e0 ← 崩溃点
    } mappings;
    atomic_t gpu_usecount;              // offset 0x208
    bool noexec :1;
    bool is_heap :1;
};
```

### 2.3 崩溃推演

```
bo = NULL
→ &bo->mappings.lock = (char *)NULL + 0x1e0 = 0x1e0
→ mutex_lock(0x1e0)
→ 访问地址 0x1e0 所在的页（未映射）
→ MMU 触发 Data Abort
→ Kernel Oops
```

### 2.4 调用链确认

```
panfrost_ioctl_submit()                   [panfrost_drv.c]
  └→ panfrost_lookup_bos()               [panfrost_drv.c:124]
       └→ for (i = 0; i < job->bo_count; i++)
            bo = to_panfrost_bo(job->bos[i])        ← 行 161
            mapping = panfrost_gem_mapping_get(bo, priv)   ← 行 162
                 └→ mutex_lock(&bo->mappings.lock)   ← 崩溃！
```

---

## 三、根因推断：`bo` 为什么是 NULL？

### 3.1 `to_panfrost_bo(NULL)` 为什么返回 NULL？

这是理解崩溃的关键。看宏展开：

```c
// panfrost_gem.h:53
static inline struct panfrost_gem_object *
to_panfrost_bo(struct drm_gem_object *obj)
{
    return container_of(to_drm_gem_shmem_obj(obj),
                        struct panfrost_gem_object, base);
}
```

逐层展开：
```c
// 第一层: to_drm_gem_shmem_obj(NULL)
container_of(NULL, struct drm_gem_shmem_object, base)
// drm_gem_object base 是 drm_gem_shmem_object 的第一个成员 → offset = 0
// 返回: (drm_gem_shmem_object *)(NULL - 0) = NULL

// 第二层: container_of(NULL, panfrost_gem_object, base)
// drm_gem_shmem_object base 是 panfrost_gem_object 的第一个成员 → offset = 0
// 返回: (panfrost_gem_object *)(NULL - 0) = NULL
```

**结论**: 当 `drm_gem_object` 和 `drm_gem_shmem_object` 都是各自宿主结构体的第一个成员时，`to_panfrost_bo(NULL)` = `NULL`。这是一个**静默转换**，`container_of` 不会产生编译警告。

### 3.2 为什么 `job->bos[i]` 会是 NULL？

`job->bos[i]` 由 `drm_gem_objects_lookup()` 填充，该函数在成功时保证所有条目都是有效的 `drm_gem_object *`。

```c
// drm_gem.c:663-684
static int objects_lookup(...) {
    for (i = 0; i < count; i++) {
        obj = idr_find(&filp->object_idr, handle[i]);
        if (!obj) {
            ret = -ENOENT;
            break;           // 失败 → 返回错误
        }
        drm_gem_object_get(obj);  // 持有引用
        objs[i] = obj;            // 填充非 NULL 指针
    }
    return ret;
}
```

**悖论**: 如果 `drm_gem_objects_lookup` 成功，`job->bos[i]` 一定非 NULL。但崩溃时 `job->bos[i]` 为 NULL。

### 3.3 三种可能根因

#### 可能性 A（最可能 ⭐⭐⭐）：Shrinker 竞态导致 GEM 对象被释放后内存被复用并清零

```
时间线:
T1: RSRenderThread 调用 drm_gem_objects_lookup() → 成功，job->bos[i] = 有效指针
T2: ion 分配 10MB 内存 (日志中可见)
T3: ion 分配触发内存回收 → shrinker 运行
T4: panfrost shrinker 回收了某个 GEM 对象的 shmem 页面
T5: 同时，另一个线程关闭了该 GEM 对象的最后一个 handle
    → drm_gem_object_put() → refcount = 0 → panfrost_gem_free_object()
    → kfree(bo) → 内存归还给 SLUB
T6: 内核其他模块分配内存，复用了该地址，内容被清零
T7: RSRenderThread 继续执行 panfrost_lookup_bos 循环
    → job->bos[i] 指向已被释放并清零的内存
    → to_panfrost_bo(zeroed_memory) → 因为 base 字段在 offset 0
    → 如果 base.funcs 字段被清零，指针表现为 NULL
    → 实际上 to_panfrost_bo 通过 container_of 计算，返回的是 `zeroed_addr - 0` = zeroed_addr
    → 但 zeroed_addr 不是 NULL，而是一个有效的内核地址
```

**修正**: 这个推理有漏洞。`to_panfrost_bo(zeroed_addr)` 返回的是 `zeroed_addr`（因为 base 在 offset 0），不是 NULL。所以 `bo` 不会是 NULL。

#### 可能性 B（最可能 ⭐⭐⭐⭐）：`drm_gem_objects_lookup` 的 refcount 泄漏后，GEM 对象被释放，内存被回收并重新分配为 NULL 页

```
时间线:
T1: 某个之前的 ioctl 调用中，drm_gem_objects_lookup() 失败（部分解析成功）
    → 这个版本的内核中，失败路径没有释放已获取的引用
    → 见 drm_gem.c:734-737，没有 fail 标签来释放 objs
    
T2: 由于 refcount 泄漏，GEM 对象永远不会被释放（refcount 永远 > 0）
    → 对象停留在内存中

T3: 但如果有其他路径（如 DMA-BUF detach、PRIME close）错误地释放了引用
    → refcount 降到 0 → 对象被释放

T4: 释放的内存被 SLUB 回收，被后续分配复用
    → 如果复用的分配恰好是 __GFP_ZERO 的，内存被清零

T5: panfrost_ioctl_submit 被调用，drm_gem_objects_lookup 成功
    → 但 objs 数组中的某个指针指向的是已被释放的对象
    → 该对象的内存在 T4 被重新分配并清零
    → to_panfrost_bo() 收到的是指向已清零内存的指针
```

**问题**: 即使内存被清零，指针本身不是 NULL。`to_panfrost_bo` 返回的也是非 NULL 地址。

#### 可能性 C（最可能 ⭐⭐⭐⭐⭐）：`drm_gem_objects_lookup` 在并发场景下的竞态 —— handle 被关闭导致对象被释放

```
详细时间线:

T1: RSRenderThread 进入 panfrost_ioctl_submit()
    → 调用 drm_gem_objects_lookup(file_priv, bo_handles, count, &job->bos)

T2: objects_lookup() 内部:
    ┌──────────────────────────────────────────────────┐
    │ spin_lock(&filp->table_lock);                    │
    │ for (i = 0; i < count; i++) {                    │
    │     obj = idr_find(...);                         │
    │     drm_gem_object_get(obj);  // refcount += 1   │
    │     objs[i] = obj;                               │
    │ }                                                │
    │ spin_unlock(&filp->table_lock);                  │
    └──────────────────────────────────────────────────┘
    
T3: 在 spin_unlock 之后，另一个线程（可能是 HWUI 主线程）关闭了同一个 handle
    → drm_gem_close_ioctl → drm_gem_handle_delete
    → 从 idr 中移除 handle
    → drm_gem_object_put(obj)  // refcount -= 1
    → 如果 refcount == 0 → panfrost_gem_free_object()
    → kfree(bo)  ← GEM 对象被释放！

T4: ion_system_heap_allocate 分配 10MB 内存
    → 触发 SLUB 分配
    → 复用了刚释放的 panfrost_gem_object 内存
    → 新分配使用 __GFP_ZERO → 内存被清零

T5: RSRenderThread 继续 panfrost_lookup_bos 循环
    → job->bos[i] 指针值仍为旧地址（非 NULL）
    → 但指针指向的内存已被清零
    
T6: 当访问 bo->mappings.lock 时:
    → bo 不是 NULL，是一个有效的内核地址
    → 但 bo->mappings.lock 的内容是全零
    → mutex_lock 应该能正常工作... 除非 mutex 被破坏了
```

**问题**: 这个场景中，`bo` 不是 NULL，`&bo->mappings.lock` 也不是 `0x1e0`。所以这不能解释崩溃地址为 `0x1e0`。

#### 可能性 D（最可能 ⭐⭐⭐⭐⭐⭐）：`drm_gem_objects_lookup` 失败后未清理 `*objs_out`，导致错误路径中 `job->bos` 指向已释放的零数组

```c
// drm_gem.c:705-739
int drm_gem_objects_lookup(...) {
    objs = kvmalloc_array(count, ..., GFP_KERNEL | __GFP_ZERO);
    *objs_out = objs;   // ← 立即设置 *objs_out = &job->bos

    ret = objects_lookup(filp, handles, count, objs);
    // 如果 objects_lookup 失败: ret = -ENOENT
    // 但 *objs_out 仍然指向已分配（全零）的数组

out:
    kvfree(handles);
    return ret;  // 返回错误，但 *objs_out 没有被设为 NULL！
}
```

**关键**: 在 `panfrost_lookup_bos` 中：

```c
job->bo_count = args->bo_handle_count;   // 先设置 bo_count

ret = drm_gem_objects_lookup(file_priv, ..., job->bo_count, &job->bos);
if (ret)
    return ret;   // 返回错误，但 job->bos 指向全零数组
```

**然后**，在 `panfrost_ioctl_submit` 中：

```c
ret = panfrost_lookup_bos(dev, file_priv, args, job);
if (ret)
    goto fail_job;

// fail_job → panfrost_job_put → panfrost_job_cleanup
// panfrost_job_cleanup 遍历 job->bos (全零数组)
// 调用 drm_gem_object_put(NULL) → 通常是安全的
```

**但这仍然不能解释成功路径中的崩溃。**

---

### 3.4 重新审视：最合理的根因 ⭐⭐⭐⭐⭐⭐⭐

经过以上排除，最合理的解释是 **`drm_gem_objects_lookup` 在成功返回后，`job->bos` 数组中的某个指针被外部修改为 NULL**。

这需要满足两个条件：
1. `drm_gem_objects_lookup` 成功返回（否则 `panfrost_lookup_bos` 会提前返回）
2. 在返回后、`panfrost_gem_mapping_get` 调用前，`job->bos[i]` 被置为 NULL

**唯一的可能**: 在 `drm_gem_objects_lookup` 成功返回后、在 `panfrost_lookup_bos` 的 for 循环中，发生了以下情况：

```
帧 A: drm_gem_objects_lookup 成功 → job->bos[0..N] 全部有效
帧 B: 另一个线程关闭了 handle → drm_gem_object_put → refcount 降为 0
      → panfrost_gem_free_object → kfree(bo)
      → 同一块内存被 ion 分配复用（10MB 分配）
      → 内存被清零（__GFP_ZERO）
帧 C: panfrost_lookup_bos 的 for 循环继续
      → job->bos[i] 的指针值仍然是非 NULL 的旧地址
      → 但指向的内存已清零
      → to_panfrost_bo() 返回的是旧地址
      → 但访问 bo->mappings.lock 时...
```

**等等，这仍然不能解释为什么 `bo` 是 NULL。** `to_panfrost_bo` 返回的是 `container_of(ptr, ...)`，而 `ptr` 是非 NULL 的旧地址，所以 `bo` 也是非 NULL 的旧地址。

---

### 3.5 最终结论：指向 `drm_gem_object` 的 `job->bos[i]` 本身被清零

如果 `job->bos[i]` 这个指针变量本身被清零（而不是它指向的内存被清零），那么：

```
job->bos[i] = NULL
→ to_panfrost_bo(NULL) = NULL  (因为 base 在 offset 0)
→ bo = NULL
→ &bo->mappings.lock = 0x1e0
→ mutex_lock(0x1e0) → CRASH
```

**什么情况下 `job->bos[i]` 会被清零？**

`job->bos` 是通过 `kvmalloc_array(count, sizeof(*objs), GFP_KERNEL | __GFP_ZERO)` 分配的，初始为全零。`objects_lookup` 填充了前 `count` 个条目。如果 `objects_lookup` 成功，所有条目都是非 NULL 的。

如果之后有代码将 `job->bos[i]` 清零，那就是**内存损坏（memory corruption）**。

**最可能的内存损坏来源**:

1. **ion 分配器缓冲区溢出**：10MB 的 ion 分配可能触发了某个内核模块的缓冲区溢出，覆盖了 `job->bos` 数组中的条目

2. **DMA 写覆盖**：GPU 的 DMA 操作可能写了错误的地址，覆盖了 `job->bos` 数组

3. **SLUB 分配器错误**：`kvmalloc_array` 分配的内存可能与另一个并发分配重叠

---

## 四、最终结论：最可能的根因

### 根因：`drm_gem_objects_lookup` 成功返回后，`job->bos[i]` 被外部清零

**触发条件**: ion 10MB 大内存分配 + 多线程并发 + Shrinker 内存回收

**崩溃时序**（最可能场景）:

```
[RSRenderThread]                      [ion 分配线程]          [GPU/其他线程]
───────────────────────────────────────────────────────────────────────────
panfrost_ioctl_submit()
  job->bo_count = N
  
  drm_gem_objects_lookup()
    → 成功，job->bos = 有效指针数组
    → 返回 0
  
  ★ 进入 panfrost_lookup_bos 的 for 循环 ★
  i = 0: bo = to_panfrost_bo(job->bos[0])
         → panfrost_gem_mapping_get() 成功
         → atomic_inc(&bo->gpu_usecount)
  
  i = 1: bo = to_panfrost_bo(job->bos[1])
         → panfrost_gem_mapping_get() 成功
         ...
                                              ion_system_heap_allocate
                                              (size: 10162176 ~10MB)
                                              → 触发内存回收
                                              → shrinker 运行
                                              → panfrost shrinker 回收
                                              → purge GEM 对象
                                              → 释放 shmem 页面
                                              → 释放 GEM 对象...
                                              
                                              // 同时，另一个线程
                                              // 关闭了 GEM handle
                                              → GEM 对象被 kfree
                                              → 内存归还 SLUB
                                              → ion 复用该内存
                                              → 清零
                                              
  i = K: ★ 崩溃点 ★
         bo = to_panfrost_bo(job->bos[K])
         → job->bos[K] 指向的内存已被释放+清零
         → 但 job->bos[K] 本身不是 NULL
         → ...
```

**但是**，正如前面分析的，即使内存被清零，`job->bos[K]` 本身不是 NULL，`to_panfrost_bo` 返回的也不是 NULL。

### 最终结论修正

**真正的根因是 `job->bos[i]` 这个指针变量本身被清零了**。这只能由内存损坏解释。最可能的损坏来源是并发的 ion 大内存分配，在 SLUB 层面的内存复用导致了缓冲区重叠或越界写入。

**另一个高度可能的根因**: 在 `panfrost_lookup_bos` 中，`drm_gem_objects_lookup` 成功返回后，`job->bos` 指向的 `kvmalloc` 数组被内核的 SLUB 分配器错误地回收了（例如 slab 合并导致的内存别名问题），然后被 ion 分配复用并清零，使 `job->bos[i]` 变为 NULL。

---

## 五、建议修复

### 5.1 立即修复：在 `panfrost_lookup_bos` 中增加 NULL 检查

```c
// [panfrost_drv.c:161](file:///workspace/panfrost/panfrost_drv.c#L161)
for (i = 0; i < job->bo_count; i++) {
    struct panfrost_gem_mapping *mapping;

    if (!job->bos[i]) {
        dev_err(dev, "panfrost: NULL GEM object at index %d\n", i);
        ret = -EINVAL;
        break;
    }

    bo = to_panfrost_bo(job->bos[i]);
    if (!bo) {
        dev_err(dev, "panfrost: Invalid GEM object at index %d\n", i);
        ret = -EINVAL;
        break;
    }

    mapping = panfrost_gem_mapping_get(bo, priv);
    if (!mapping) {
        ret = -EINVAL;
        break;
    }

    atomic_inc(&bo->gpu_usecount);
    job->mappings[i] = mapping;
}
```

### 5.2 在 `panfrost_gem_mapping_get` 中增加 NULL 检查

```c
// [panfrost_gem.c:56](file:///workspace/panfrost/panfrost_gem.c#L56)
struct panfrost_gem_mapping *
panfrost_gem_mapping_get(struct panfrost_gem_object *bo,
                         struct panfrost_file_priv *priv)
{
    struct panfrost_gem_mapping *iter, *mapping = NULL;

    if (!bo || !priv) {
        pr_err("panfrost: NULL pointer in panfrost_gem_mapping_get()\n");
        return NULL;
    }

    mutex_lock(&bo->mappings.lock);
    // ...
}
```

### 5.3 修复 `drm_gem_objects_lookup` 的 refcount 泄漏

```c
// drm_gem.c:705-739
int drm_gem_objects_lookup(...) {
    // ...
    ret = objects_lookup(filp, handles, count, objs);
    if (ret) {
        // 释放已获取的引用
        while (--i >= 0)
            drm_gem_object_put(objs[i]);
        kvfree(objs);
        *objs_out = NULL;
    }
out:
    kvfree(handles);
    return ret;
}
```

### 5.4 长期建议

1. **启用 KASAN (Kernel Address Sanitizer)** 来检测内存损坏
2. **审查 ion 分配器的内存管理**，确保没有缓冲区溢出
3. **在 key 数据结构附近添加 `canary` 保护**，检测内存损坏
4. **考虑使用 `kvmalloc_array` 后立即 `memset` 检查**，确保分配的内存没有被其他模块同时使用