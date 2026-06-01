# panfrost_gem_create_with_handle() 函数分析与风险评估

## 一、函数功能

`panfrost_gem_create_with_handle()` 是 Panfrost 驱动中 **BO (Buffer Object) 创建的核心函数**，由用户空间 `PANFROST_IOCTL_CREATE_BO` 触发调用。

### 1.1 调用链

```
用户空间: PANFROST_IOCTL_CREATE_BO
    │
    ▼
panfrost_ioctl_create_bo()              [panfrost_drv.c:77]
    │
    ├── 参数校验: size != 0, 非法 flags 检查
    ├── HEAP 必须同时设置 NOEXEC
    │
    ├── panfrost_gem_create_with_handle()  ← 本函数
    │   ├── drm_gem_shmem_create()          # 创建 shmem 后备 GEM 对象
    │   │   └── panfrost_gem_create_object()  # 分配 panfrost_gem_object
    │   │       ├── INIT_LIST_HEAD(&mappings.list)
    │   │       ├── mutex_init(&mappings.lock)
    │   │       └── 设置 funcs = &panfrost_gem_funcs
    │   │
    │   ├── 设置 bo->noexec / bo->is_heap 标志
    │   │
    │   ├── drm_gem_handle_create()         # 分配用户空间句柄
    │   │   └── panfrost_gem_open()          # 创建 mapping + 虚拟地址 + MMU 映射
    │   │       ├── drm_mm_insert_node_generic()  # 分配虚拟地址
    │   │       ├── panfrost_mmu_map()       # 建立 IOMMU 页表 (非 HEAP)
    │   │       └── 加入 bo->mappings.list
    │   │
    │   └── drm_gem_object_put()            # 释放创建引用，句柄持有剩余引用
    │
    ├── panfrost_gem_mapping_get()          # 查找刚创建的 mapping
    └── 返回 GPU 虚拟地址偏移量给用户空间
```

### 1.2 函数源码

```c
// [panfrost_gem.c:239-272](file:///workspace/panfrost/panfrost_gem.c#L239-L272)
struct panfrost_gem_object *
panfrost_gem_create_with_handle(struct drm_file *file_priv,
                struct drm_device *dev, size_t size,
                u32 flags,
                uint32_t *handle)
{
    int ret;
    struct drm_gem_shmem_object *shmem;
    struct panfrost_gem_object *bo;

    // ① HEAP BO 的 size 向上取整到 2MB
    if (flags & PANFROST_BO_HEAP)
        size = roundup(size, SZ_2M);

    // ② 创建 shmem 后备 GEM 对象 (refcount=1)
    shmem = drm_gem_shmem_create(dev, size);
    if (IS_ERR(shmem))
        return ERR_CAST(shmem);

    // ③ 设置 BO 属性
    bo = to_panfrost_bo(&shmem->base);
    bo->noexec = !!(flags & PANFROST_BO_NOEXEC);
    bo->is_heap = !!(flags & PANFROST_BO_HEAP);

    // ④ 分配用户空间句柄 (refcount=2, 触发 panfrost_gem_open)
    ret = drm_gem_handle_create(file_priv, &shmem->base, handle);

    // ⑤ 释放创建引用 (refcount=1)
    drm_gem_object_put(&shmem->base);

    if (ret)
        return ERR_PTR(ret);

    return bo;
}
```

### 1.3 引用计数生命周期

```
drm_gem_shmem_create()      → refcount = 1   (创建者持有)
drm_gem_handle_create()     → refcount = 2   (句柄也持有)
drm_gem_object_put()        → refcount = 1   (创建者释放)
... 用户关闭句柄时 ...
drm_gem_handle_delete()     → refcount = 0   → panfrost_gem_free_object()
```

---

## 二、潜在风险分析

### 🔴 风险 1：`drm_gem_object_put()` 顺序问题 —— 错误路径下的 Use-After-Free

**位置**: [panfrost_gem.c:265-269](file:///workspace/panfrost/panfrost_gem.c#L265-L269)

**问题代码**:
```c
ret = drm_gem_handle_create(file_priv, &shmem->base, handle);
drm_gem_object_put(&shmem->base);  // ← 先 put，再检查 ret
if (ret)
    return ERR_PTR(ret);
```

**问题分析**:

当 `drm_gem_handle_create()` 失败时:
1. `drm_gem_object_put()` 将 refcount 从 1 降为 0
2. 触发 `panfrost_gem_free_object()` → `drm_gem_shmem_free_object()` → 释放 BO 内存
3. 此时 `shmem` 和 `bo` 指针变为**悬挂指针**
4. 函数返回 `ERR_PTR(ret)`，调用者通过 `IS_ERR()` 检测到错误

**当前代码是否安全？**

当前调用者 `panfrost_ioctl_create_bo()` 的用法:
```c
bo = panfrost_gem_create_with_handle(file, dev, args->size, args->flags, &args->handle);
if (IS_ERR(bo))
    return PTR_ERR(bo);  // ← 直接返回，不访问 bo 指针
```

**目前是安全的**，因为调用者通过 `IS_ERR()` 检查后立即返回，不再访问 `bo`。但这是一个**脆弱的约定**：

- 如果将来有人在 `panfrost_gem_create_with_handle()` 中的 `drm_gem_object_put()` 和 `return ERR_PTR(ret)` 之间插入使用 `bo`/`shmem` 的代码，就会出现 use-after-free
- 如果调用者忘记用 `IS_ERR()` 检查，直接使用返回的 `bo` 指针，就会崩溃

**严重程度**: 🔴 高（虽然当前安全，但极易被破坏）

**建议修复**:
```c
ret = drm_gem_handle_create(file_priv, &shmem->base, handle);
if (ret) {
    drm_gem_object_put(&shmem->base);
    return ERR_PTR(ret);
}
drm_gem_object_put(&shmem->base);
return bo;
```

---

### 🟡 风险 2：HEAP BO 的 size 向上取整导致语义不一致

**位置**: [panfrost_gem.c:249-251](file:///workspace/panfrost/panfrost_gem.c#L249-L251)

**问题代码**:
```c
if (flags & PANFROST_BO_HEAP)
    size = roundup(size, SZ_2M);

shmem = drm_gem_shmem_create(dev, size);
```

**问题分析**:

用户传入 `args->size = 4097`（略大于 4KB），带 `PANFROST_BO_HEAP` 标志。此时:
1. `size` 被向上取整为 `2MB`
2. `drm_gem_shmem_create()` 创建的 BO 实际大小为 2MB
3. 但用户空间持有的 `args->size` 仍然是 4097
4. 用户尝试 mmap 时会使用 4097 作为长度，但实际 BO 是 2MB

**影响**:
- 用户通过 mmap 只能访问前 4097 字节，剩余部分无法通过 CPU 访问
- GPU 端的缺页处理按 2MB 粒度分配，可能分配了用户不需要的物理内存
- 虽然功能上不崩溃，但内存浪费和语义不一致是潜在问题

**严重程度**: 🟡 中（功能正确但浪费内存，语义不透明）

---

### 🟡 风险 3：`drm_gem_shmem_create()` 成功但物理页延迟分配

**位置**: [panfrost_gem.c:253](file:///workspace/panfrost/panfrost_gem.c#L253)

**问题分析**:

`drm_gem_shmem_create()` 只创建 GEM 对象结构体，**不分配物理页**。物理页在首次访问时通过 `drm_gem_shmem_get_pages_sgt()` 延迟分配。

这意味着:
1. `panfrost_gem_create_with_handle()` 成功返回，用户认为 BO 已就绪
2. 当 GPU 提交作业引用该 BO 时，`panfrost_mmu_map()` 调用 `drm_gem_shmem_get_pages_sgt()` 尝试分配物理页
3. 如果此时系统内存不足，分配失败，但 BO 已经创建成功

**失败表现**:
```c
// panfrost_mmu.c:291
sgt = drm_gem_shmem_get_pages_sgt(obj);
if (WARN_ON(IS_ERR(sgt)))
    return PTR_ERR(sgt);  // 返回 -ENOMEM，但 BO 已存在于用户空间
```

**影响**:
- 用户空间拿到有效的 handle，但提交作业时失败
- 错误发生的时间点远离 BO 创建，难以诊断
- 对于 HEAP BO，失败发生在缺页中断上下文中，处理更加复杂

**严重程度**: 🟡 中（延迟失败，诊断困难）

---

### 🟡 风险 4：`panfrost_gem_open()` 隐式副作用 —— 调用者依赖未文档化的行为

**位置**: [panfrost_gem.c:265](file:///workspace/panfrost/panfrost_gem.c#L265) 调用 `drm_gem_handle_create()` → `panfrost_gem_open()`

**问题分析**:

`panfrost_gem_create_with_handle()` 调用 `drm_gem_handle_create()` 时，会**隐式触发** `panfrost_gem_open()`，该函数做了大量工作:

```c
// panfrost_gem.c:121-174
int panfrost_gem_open(...) {
    // 1. 分配 panfrost_gem_mapping
    // 2. 在 drm_mm 中分配虚拟地址 (4GB 空间)
    // 3. 对于非 HEAP BO: 分配物理页 + 建立 IOMMU 映射
    // 4. 将 mapping 加入 bo->mappings.list
}
```

**调用者依赖**: `panfrost_ioctl_create_bo()` 在调用 `panfrost_gem_create_with_handle()` 之后，立即调用 `panfrost_gem_mapping_get()` 来获取刚创建的 mapping。这个调用**隐式依赖于** `drm_gem_handle_create()` 内部调用 `panfrost_gem_open()` 成功创建了 mapping。

**风险**:
- 如果 `drm_gem_handle_create()` 的实现发生变化（例如不再调用 `drm_gem_open_ioctl`），则 `panfrost_gem_mapping_get()` 会返回 NULL
- 这种隐式依赖没有文档化，代码审查时容易遗漏
- 如果 `panfrost_gem_open()` 部分成功（创建了 mapping 但未加入列表），`panfrost_gem_mapping_get()` 也会返回 NULL

**严重程度**: 🟡 中（隐式依赖，可维护性风险）

---

### 🟢 风险 5：`to_panfrost_bo()` 无类型安全检查

**位置**: [panfrost_gem.c:257](file:///workspace/panfrost/panfrost_gem.c#L257)

**问题代码**:
```c
bo = to_panfrost_bo(&shmem->base);
bo->noexec = !!(flags & PANFROST_BO_NOEXEC);
bo->is_heap = !!(flags & PANFROST_BO_HEAP);
```

**`to_panfrost_bo()` 实现**:
```c
// panfrost_gem.h:53-56
static inline struct panfrost_gem_object *
to_panfrost_bo(struct drm_gem_object *obj)
{
    return container_of(to_drm_gem_shmem_obj(obj),
                        struct panfrost_gem_object, base);
}
```

**问题分析**:

`to_panfrost_bo()` 通过 `container_of` 进行强制类型转换，**没有任何类型安全检查**。它假设传入的 `drm_gem_object` 一定是 `panfrost_gem_object`。

在当前上下文中，`drm_gem_shmem_create()` 调用了 `panfrost_gem_create_object()` 来创建对象，所以类型是正确的。但如果:
- 将来使用其他方式创建的 GEM 对象（如 `drm_gem_prime_import_sg_table`）
- 传入的不是 panfrost 的 GEM 对象

`container_of` 仍会返回一个地址，但访问 `bo->noexec` 等字段会修改到错误的内存位置。

**当前安全原因**: `drm_gem_shmem_create()` 通过 `dev->driver->gem_create_object` 回调创建对象，该回调被设置为 `panfrost_gem_create_object()`，确保类型正确。

**严重程度**: 🟢 低（在当前上下文中类型保证正确）

---

### 🟢 风险 6：HEAP 标志下的 2MB 向上取整可能溢出

**位置**: [panfrost_gem.c:250-251](file:///workspace/panfrost/panfrost_gem.c#L250-L251)

**问题代码**:
```c
if (flags & PANFROST_BO_HEAP)
    size = roundup(size, SZ_2M);
```

**问题分析**:

如果用户传入 `size` 接近 `SIZE_MAX`:
- `roundup(SIZE_MAX - 1MB, 2MB)` → 可能溢出回绕到 0 或小值
- 后续 `drm_gem_shmem_create(dev, 0)` 的行为未定义

**实际风险评估**: 用户空间传入的 `size` 来自 `args->size`（`__u64`），在 `panfrost_ioctl_create_bo()` 中已检查 `!args->size`，但不检查上限。不过 `drm_gem_shmem_create()` 内部通常有自己的 size 校验。

**严重程度**: 🟢 低（需要恶意构造，且内核其他层有防护）

---

## 三、风险总结

| 编号 | 风险 | 严重程度 | 触发条件 | 当前状态 |
|------|------|----------|----------|----------|
| 1 | `drm_gem_object_put()` 顺序 —— 错误路径 dangling pointer | 🔴 高 | `drm_gem_handle_create()` 失败 | 调用者做了 IS_ERR 检查，当前安全但脆弱 |
| 2 | HEAP size 向上取整语义不一致 | 🟡 中 | HEAP BO 创建 | 功能正确但浪费内存 |
| 3 | 物理页延迟分配导致失败延后 | 🟡 中 | 系统内存不足 | 诊断困难 |
| 4 | `panfrost_gem_open()` 隐式副作用 | 🟡 中 | 代码变更 | 依赖未文档化 |
| 5 | `to_panfrost_bo()` 无类型检查 | 🟢 低 | 错误类型传入 | 当前安全 |
| 6 | roundup 整数溢出 | 🟢 低 | 恶意构造 | 其他层有防护 |

## 四、改进建议

### 4.1 立即改进：修复 `drm_gem_object_put()` 顺序

```c
// 修改前:
ret = drm_gem_handle_create(file_priv, &shmem->base, handle);
drm_gem_object_put(&shmem->base);
if (ret)
    return ERR_PTR(ret);

// 修改后:
ret = drm_gem_handle_create(file_priv, &shmem->base, handle);
if (ret) {
    drm_gem_object_put(&shmem->base);
    return ERR_PTR(ret);
}
drm_gem_object_put(&shmem->base);
return bo;
```

### 4.2 建议改进：为 HEAP BO 添加 size 校验

```c
if (flags & PANFROST_BO_HEAP) {
    if (size > MAX_HEAP_SIZE)  // 定义合理的上限
        return ERR_PTR(-EINVAL);
    size = roundup(size, SZ_2M);
}
```

### 4.3 建议改进：增加 `to_panfrost_bo()` 的类型安全断言

```c
static inline struct panfrost_gem_object *
to_panfrost_bo(struct drm_gem_object *obj)
{
    if (WARN_ON_ONCE(!obj || obj->funcs != &panfrost_gem_funcs))
        return NULL;
    return container_of(to_drm_gem_shmem_obj(obj),
                        struct panfrost_gem_object, base);
}
```