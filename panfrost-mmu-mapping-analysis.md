# Panfrost 模块内存映射详解与风险分析

## 一、数据结构总览

### 1.1 核心数据结构关系

```
panfrost_device
├── features.as_present          # 硬件支持的 AS 槽位数量
├── as_alloc_mask                # 已分配的 AS 位图
├── as_lru_list                  # AS 的 LRU 链表（最近使用）
├── as_lock                      # AS 操作的自旋锁
├── shrinker_lock / shrinker_list # 内存回收器
│
└── panfrost_file_priv (per-FD)
    ├── mmu                      # 该 FD 的 MMU 实例
    │   ├── as                   # 分配的硬件 AS 编号 (-1 表示未分配)
    │   ├── as_count             # AS 引用计数
    │   ├── pgtbl_ops            # io_pgtable 操作接口
    │   └── pgtbl_cfg            # 页表配置 (ias/oas/pgsize)
    │
    ├── mm                       # drm_mm 虚拟地址空间管理器 (4GB)
    ├── mm_lock                  # 虚拟地址空间锁
    └── sched_entity[3]          # 3 个 Job Slot 的调度实体
```

### 1.2 GEM 对象与映射关系

```
panfrost_gem_object (BO)
├── base (drm_gem_shmem_object)  # shmem 后备存储
│   ├── pages[]                  # 物理页数组
│   └── pages_lock               # 页锁
├── sgts[]                       # scatter-gather 表 (每 2MB 一个)
├── mappings                     # 链表: 所有 FD 对该 BO 的映射
│   ├── list
│   └── lock (mutex)
├── gpu_usecount                 # 活跃 GPU 作业引用计数
├── noexec                       # 是否禁止执行
└── is_heap                      # 是否为堆内存 (按需分配)

panfrost_gem_mapping (per-FD, per-BO)
├── node                         # 链入 panfrost_gem_object.mappings.list
├── refcount (kref)              # 引用计数
├── obj                          # 指向所属 BO
├── mmnode (drm_mm_node)         # 虚拟地址区间节点
├── mmu                          # 指向所属 MMU
└── active                       # 是否已建立页表映射
```

---

## 二、内存映射完整流程

### 2.1 阶段一：BO 创建 (CREATE_BO IOCTL)

```
用户空间: PANFROST_IOCTL_CREATE_BO
    │
    ▼
panfrost_ioctl_create_bo()          [panfrost_drv.c:78]
    │
    ├── panfrost_gem_create_with_handle()
    │   ├── drm_gem_shmem_create()  # 创建 shmem GEM 对象
    │   ├── 设置 noexec / is_heap 标志
    │   └── drm_gem_handle_create() # 分配用户空间句柄
    │
    └── panfrost_gem_mapping_get()  # 获取当前 FD 的映射
        └── 遍历 bo->mappings.list 查找匹配的 mmu
```

**关键点**：此时 BO 还没有建立虚拟地址映射，也没有分配物理页。物理页在首次需要时通过 `drm_gem_shmem_get_pages_sgt()` 延迟分配。

---

### 2.2 阶段二：虚拟地址分配 (GEM .open 回调)

当另一个进程通过 `drm_gem_flink` / `drm_gem_open` 打开同一个 BO，或者首次创建 BO 时触发 `panfrost_gem_open()`：

```
panfrost_gem_open()                  [panfrost_gem.c:121]
    │
    ├── 分配 panfrost_gem_mapping 结构
    │
    ├── drm_mm_insert_node_generic() # 在 FD 私有的 4GB 虚拟地址空间中分配
    │   ├── 地址范围: [32MB, 4GB)
    │   ├── 对齐约束:
    │   │   - 可执行 BO: 对齐到 BO 大小 (防止跨 16MB 边界)
    │   │   - 非可执行 BO: 2MB 对齐 (如果 size >= 2MB)
    │   └── color: 可执行用 0, 不可执行用 PANFROST_BO_NOEXEC
    │
    ├── panfrost_mmu_map()            # 建立 IOMMU 页表映射
    │   ├── drm_gem_shmem_get_pages_sgt()  # 获取/分配物理页 + SG 表
    │   ├── mmu_map_sg()
    │   │   ├── 遍历 SG 条目
    │   │   ├── get_pgsize(): 4KB 或 2MB 大页
    │   │   └── ops->map(): 建立 io_pgtable 页表项
    │   └── panfrost_mmu_flush_range()     # 刷新 TLB
    │
    └── 将 mapping 加入 bo->mappings.list
```

**注意**：对于 `is_heap` 类型的 BO，此阶段**跳过** `panfrost_mmu_map()`，物理页在 MMU 缺页时按需分配。

---

### 2.3 阶段三：作业提交时的地址空间分配

```
panfrost_job_hw_submit()             [panfrost_job.c:141]
    │
    └── panfrost_mmu_as_get()        [panfrost_mmu.c:145]
        │
        ├── 如果 mmu->as >= 0 (已有 AS)：
        │   ├── atomic_inc(&mmu->as_count)
        │   └── 移动到 LRU 链表头部
        │
        └── 如果 mmu->as == -1 (无 AS)：
            ├── ffz(pfdev->as_alloc_mask)  # 找空闲 AS
            │
            ├── 如果无空闲 AS：
            │   ├── 从 LRU 链表尾部找 as_count==0 的 mmu
            │   ├── 回收其 AS (lru_mmu->as = -1)
            │   └── 分配给当前 mmu
            │
            ├── panfrost_mmu_enable()       # 写 AS_TRANSTAB + AS_MEMATTR
            │   ├── AS_COMMAND_FLUSH_MEM    # 刷新内存
            │   ├── 写 AS_TRANSTAB_LO/HI    # 页表基地址
            │   ├── 写 AS_MEMATTR_LO/HI     # 内存属性
            │   └── AS_COMMAND_UPDATE       # 广播配置
            │
            └── atomic_set(&mmu->as_count, 1)
```

**AS 与 Job Slot 的关系**：
```
AS 是硬件地址空间槽位，每个 AS 有独立的页表基地址
Job Slot 是 GPU 命令提交槽位，3 个 slot 可以共享同一个 AS
同一个 FD 的所有 Job Slot 共享同一个 AS
```

---

### 2.4 阶段四：堆内存按需映射 (缺页处理)

```
GPU 访问未映射的堆地址
    │
    ▼
MMU 硬件触发缺页中断
    │
    ▼
panfrost_mmu_irq_handler()           [panfrost_mmu.c:565]
    │
    └── panfrost_mmu_irq_handler_thread()  [panfrost_mmu.c:576]
        │
        └── panfrost_mmu_map_fault_addr()  [panfrost_mmu.c:446]
            │
            ├── addr_to_mapping()           # 根据 AS+地址查找 mapping
            │   ├── 遍历 as_lru_list 找匹配的 AS
            │   ├── 在 priv->mm 中搜索地址区间
            │   └── kref_get(&mapping->refcount)
            │
            ├── 验证 bo->is_heap == true
            │
            ├── 以 2MB 对齐分配物理页
            │   ├── shmem_read_mapping_page() x 512 页
            │   ├── sg_alloc_table_from_pages()
            │   └── dma_map_sgtable()
            │
            ├── mmu_map_sg()                # 建立 IOMMU 映射
            │
            └── panfrost_gem_mapping_put()  # 释放 addr_to_mapping 的引用
```

---

### 2.5 阶段五：内存解映射

```
BO 销毁时：
panfrost_gem_free_object()           [panfrost_gem.c:17]
    ├── 从 shrinker_list 移除
    ├── 遍历 bo->sgts[]，释放 DMA 映射和 SG 表
    └── drm_gem_shmem_free_object()

mapping 释放时：
panfrost_gem_mapping_release()       [panfrost_gem.c:94]
    ├── panfrost_gem_teardown_mapping()
    │   ├── panfrost_mmu_unmap()     # 遍历页表，逐页 Unmap
    │   │   ├── get_pgsize()
    │   │   ├── ops->iova_to_phys()  # 检查是否已映射
    │   │   ├── ops->unmap()
    │   │   └── panfrost_mmu_flush_range()
    │   └── drm_mm_remove_node()     # 释放虚拟地址
    ├── drm_gem_object_put()
    └── kfree(mapping)
```

---

## 三、内存映射硬件机制

### 3.1 MMU 寄存器布局

```
MMU 寄存器基址: 0x2000
├── MMU_INT_RAWSTAT  (0x2000)  # 中断原始状态
├── MMU_INT_CLEAR    (0x2004)  # 中断清除
├── MMU_INT_MASK     (0x2008)  # 中断屏蔽
└── MMU_INT_STAT     (0x200c)  # 中断状态

每个 AS 的寄存器 (AS_n 基址 = 0x2400 + n * 0x40):
├── AS_TRANSTAB_LO   (+0x00)   # 页表基地址低 32 位
├── AS_TRANSTAB_HI   (+0x04)   # 页表基地址高 32 位
├── AS_MEMATTR_LO    (+0x08)   # 内存属性低 32 位
├── AS_MEMATTR_HI    (+0x0C)   # 内存属性高 32 位
├── AS_LOCKADDR_LO   (+0x10)   # 锁定区域地址低 32 位
├── AS_LOCKADDR_HI   (+0x14)   # 锁定区域地址高 32 位
├── AS_COMMAND       (+0x18)   # 命令寄存器 (WO)
├── AS_FAULTSTATUS   (+0x1C)   # 缺页状态 (RO)
├── AS_FAULTADDRESS_LO(+0x20)  # 缺页地址低 32 位
├── AS_FAULTADDRESS_HI(+0x24)  # 缺页地址高 32 位
└── AS_STATUS        (+0x28)   # 状态寄存器 (RO)
```

### 3.2 MMU 命令

| 命令 | 值 | 说明 |
|------|-----|------|
| `AS_COMMAND_NOP` | 0x00 | 空操作 |
| `AS_COMMAND_UPDATE` | 0x01 | 广播 TRANSTAB/MEMATTR 到所有 MMU |
| `AS_COMMAND_LOCK` | 0x02 | 锁定指定区域 |
| `AS_COMMAND_UNLOCK` | 0x03 | 解锁并刷新区域 |
| `AS_COMMAND_FLUSH_PT` | 0x04 | 刷新 L2 + 刷新区域（页表刷新） |
| `AS_COMMAND_FLUSH_MEM` | 0x05 | 等待内存访问完成 + 刷新 L1 L2 |

### 3.3 页表格式

使用 **ARM Mali LPAE (Long Physical Address Extension)** 格式：
- 输入地址宽度 (ias): 由 `mmu_features[7:0]` 提取
- 输出地址宽度 (oas): 由 `mmu_features[15:8]` 提取
- 支持页大小: 4KB 和 2MB (大页)
- 大页使用条件: 地址和物理地址都 2MB 对齐，且剩余长度 >= 2MB

---

## 四、潜在风险分析

### 🔴 风险 1：mmu_map_sg() 无错误回滚

**位置**: [panfrost_mmu.c:248-275](file:///workspace/panfrost/panfrost_mmu.c#L248-L275)

**问题描述**:
```c
static int mmu_map_sg(...) {
    for_each_sgtable_dma_sg(sgt, sgl, count) {
        while (len) {
            ops->map(ops, iova, paddr, pgsize, prot, GFP_KERNEL);
            // ↑ 未检查返回值！如果 map 失败，前面已映射的页不会被回滚
        }
    }
    panfrost_mmu_flush_range(...);
    return 0;  // 总是返回成功
}
```

**风险**: 如果 `ops->map()` 在中间某页失败（例如内存不足无法分配页表），之前已映射的页表项会残留，造成 GPU 访问到错误映射，且无法被正常 unmap 清理。

**严重程度**: 🔴 高

---

### 🔴 风险 2：panfrost_mmu_unmap() 不检查 AS 有效性

**位置**: [panfrost_mmu.c:302-333](file:///workspace/panfrost/panfrost_mmu.c#L302-L333)

**问题描述**:
```c
void panfrost_mmu_unmap(struct panfrost_gem_mapping *mapping) {
    struct io_pgtable_ops *ops = mapping->mmu->pgtbl_ops;
    // ↑ 如果 AS 已被回收 (mmu->as == -1)，ops 仍指向旧的页表
    // 但此时硬件 AS 可能已被分配给其他 FD
    
    while (unmapped_len < len) {
        ops->unmap(ops, iova, pgsize, NULL);  // 操作的是软件页表
    }
    
    panfrost_mmu_flush_range(...);  // 如果 as < 0，直接跳过 flush
    // ↑ 但软件页表已修改，硬件 TLB 可能仍有旧条目
}
```

**风险**: 
1. 如果在 AS 被回收后，新的 AS 用户使用了相同的页表内存，此处的 unmap 会破坏新用户的页表
2. 硬件 TLB 刷新被跳过，但软件页表已修改，后续旧 AS 被重新分配时可能出现 TLB 不一致

**严重程度**: 🔴 高

---

### 🔴 风险 3：AS LRU 回收的竞态条件

**位置**: [panfrost_mmu.c:145-196](file:///workspace/panfrost/panfrost_mmu.c#L145-L196)

**问题描述**:
```c
u32 panfrost_mmu_as_get(struct panfrost_device *pfdev, struct panfrost_mmu *mmu) {
    spin_lock(&pfdev->as_lock);
    
    if (mmu->as >= 0) {
        // 已有 AS，移动 LRU 位置
        list_move(&mmu->list, &pfdev->as_lru_list);
        goto out;
    }
    
    // 无空闲 AS，从 LRU 尾部回收
    list_for_each_entry_reverse(lru_mmu, &pfdev->as_lru_list, list) {
        if (!atomic_read(&lru_mmu->as_count))
            break;
    }
    // 这里没有检查 lru_mmu 是否有效！
    // 如果所有 mmu 的 as_count 都 > 0，lru_mmu 指向链表头
    WARN_ON(&lru_mmu->list == &pfdev->as_lru_list);  // 仅 WARN，不阻止
    
    list_del_init(&lru_mmu->list);
    as = lru_mmu->as;
    lru_mmu->as = -1;  // 旧 mmu 的 AS 被剥夺
```

**风险**: 
1. 如果所有 AS 都被活跃作业占用，仍然会回收其中一个 AS
2. 被回收的 AS 对应的旧 mmu 仍持有 `pgtbl_ops`，其页表内容可能被新 mmu 覆盖
3. 旧 mmu 的 `as` 被设为 -1，但 `as_count` 没有被清零，引用计数不一致

**严重程度**: 🔴 高

---

### 🟡 风险 4：Shrinker 与 GPU 作业的竞态

**位置**: [panfrost_gem_shrinker.c:39-63](file:///workspace/panfrost/panfrost_gem_shrinker.c#L39-L63)

**问题描述**:
```c
static bool panfrost_gem_purge(struct drm_gem_object *obj) {
    if (atomic_read(&bo->gpu_usecount))  // 检查是否有活跃 GPU 作业
        return false;
    
    // ⚠️ 在检查 gpu_usecount 和实际 purge 之间存在时间窗口
    // 如果此时有新的作业提交引用了该 BO...
    
    panfrost_gem_teardown_mappings_locked(bo);  // 拆除 IOMMU 映射
    drm_gem_shmem_purge_locked(obj);            // 释放物理页
}
```

**风险**: 在 `gpu_usecount` 检查和实际 purge 之间，GPU 调度器可能提交了引用该 BO 的新作业。此时 GPU 会访问已经被释放的物理页，导致不可预测的行为。

**严重程度**: 🟡 中

---

### 🟡 风险 5：panfrost_mmu_flush_range() 的静默跳过

**位置**: [panfrost_mmu.c:232-246](file:///workspace/panfrost/panfrost_mmu.c#L232-L246)

**问题描述**:
```c
static void panfrost_mmu_flush_range(...) {
    if (mmu->as < 0)
        return;  // AS 未分配，静默跳过

    pm_runtime_get_noresume(pfdev->dev);
    
    if (pm_runtime_active(pfdev->dev))
        mmu_hw_do_operation(...);   // 只有设备活跃时才刷新
    // 如果设备已挂起，不刷新，也不报错！
    
    pm_runtime_put_sync_autosuspend(pfdev->dev);
}
```

**风险**: 
1. 如果设备已 runtime suspend，TLB 刷新被静默跳过
2. 下次设备唤醒时，TLB 中可能有过时的条目，导致 GPU 访问错误地址
3. `pm_runtime_get_noresume()` 只是增加引用计数，不会真正唤醒设备

**严重程度**: 🟡 中

---

### 🟡 风险 6：堆内存缺页处理的错误路径

**位置**: [panfrost_mmu.c:446-542](file:///workspace/panfrost/panfrost_mmu.c#L446-L542)

**问题描述**:
```c
static int panfrost_mmu_map_fault_addr(...) {
    for (i = page_offset; i < page_offset + NUM_FAULT_PAGES; i++) {
        pages[i] = shmem_read_mapping_page(mapping, i);
        if (IS_ERR(pages[i])) {
            mutex_unlock(&bo->base.pages_lock);
            ret = PTR_ERR(pages[i]);
            goto err_pages;  // 跳转到错误处理
        }
    }
    // ...
err_pages:
    drm_gem_shmem_put_pages(&bo->base);  // 释放所有已分配的页
err_bo:
    drm_gem_object_put(&bo->base.base);  // 额外 put，可能导致引用计数不平衡
```

**风险**:
1. 如果 `shmem_read_mapping_page()` 在中间失败，已分配的页通过 `drm_gem_shmem_put_pages()` 释放，但 `bo->base.pages_use_count` 可能不一致
2. 错误路径中 `drm_gem_object_put()` 总是执行，但 `addr_to_mapping()` 获取的引用可能已经被 `panfrost_gem_mapping_put()` 释放（正常路径第 531 行），导致双重释放或引用计数错误
3. 如果 `dma_map_sgtable()` 失败，`sg_free_table()` 后未清除 `sgt->sgl`，如果后续代码再次访问会触发 use-after-free

**严重程度**: 🟡 中

---

### 🟡 风险 7：addr_to_mapping() 的锁顺序问题

**位置**: [panfrost_mmu.c:407-442](file:///workspace/panfrost/panfrost_mmu.c#L407-L442)

**问题描述**:
```c
static struct panfrost_gem_mapping *
addr_to_mapping(struct panfrost_device *pfdev, int as, u64 addr) {
    spin_lock(&pfdev->as_lock);          // 先锁 as_lock
    // ... 遍历找 mmu ...
    spin_lock(&priv->mm_lock);           // 再锁 mm_lock
    // ... 遍历找 mapping ...
    spin_unlock(&priv->mm_lock);
    spin_unlock(&pfdev->as_lock);
}
```

**风险**: 在其他代码路径中（如 `panfrost_gem_open()`），锁顺序是 `mm_lock` → `bo->mappings.lock`。而 `addr_to_mapping()` 是 `as_lock` → `mm_lock`。如果存在需要同时持有 `as_lock` 和 `bo->mappings.lock` 的路径，可能导致死锁。

**严重程度**: 🟡 中

---

### 🟢 风险 8：panfrost_gem_mapping_release() 的潜在 use-after-free

**位置**: [panfrost_gem.c:94-103](file:///workspace/panfrost/panfrost_gem.c#L94-L103)

**问题描述**:
```c
static void panfrost_gem_mapping_release(struct kref *kref) {
    mapping = container_of(kref, struct panfrost_gem_mapping, refcount);
    
    panfrost_gem_teardown_mapping(mapping);
    // teardown 中会访问 mapping->mmu, mapping->mmnode
    
    drm_gem_object_put(&mapping->obj->base.base);
    // ↑ 如果这是最后一个引用，BO 会被释放，包括 mapping->obj 指向的内存
    
    kfree(mapping);  // 此时 mapping 自身被释放
}
```

**风险**: 在 `drm_gem_object_put()` 之后，`mapping->obj` 可能已被释放，但 `kfree(mapping)` 本身不访问 `obj`，所以这里实际安全。但如果将来有人在这两行之间插入使用 `mapping->obj` 的代码，就会出现 use-after-free。

**严重程度**: 🟢 低（当前代码安全，但需要警惕）

---

### 🟢 风险 9：虚拟地址空间耗尽

**位置**: [panfrost_drv.c:478-496](file:///workspace/panfrost/panfrost_drv.c#L478-L496)

**问题描述**:
```c
panfrost_open(struct drm_device *dev, struct drm_file *file) {
    // 每个 FD 分配 4GB 虚拟地址空间
    drm_mm_init(&panfrost_priv->mm, SZ_32M >> PAGE_SHIFT, 
                (SZ_4G - SZ_32M) >> PAGE_SHIFT);
    // 注释说 "4G enough for now. can be 48-bit"
}
```

**风险**: 
1. 虚拟地址空间是 4GB，对于大量 BO 的场景可能不够（特别是碎片化严重时）
2. 每个 FD 独立分配，无法共享虚拟地址空间
3. 可执行 BO 的 16MB 对齐约束进一步加剧碎片化

**严重程度**: 🟢 低

---

### 🟢 风险 10：mmu_hw_do_operation_locked() 的 AS 有效性检查

**位置**: [panfrost_mmu.c:83-97](file:///workspace/panfrost/panfrost_mmu.c#L83-L97)

**问题描述**:
```c
static int mmu_hw_do_operation_locked(...) {
    if (as_nr < 0)
        return 0;  // AS 无效时静默返回 0
    
    lock_region(pfdev, as_nr, iova, size);  // 未检查 as_nr 是否超出硬件支持范围
    write_cmd(pfdev, as_nr, op);
    return wait_ready(pfdev, as_nr);
}
```

**风险**: 虽然检查了 `as_nr < 0`，但没有检查 `as_nr` 是否超出 `features.as_present` 的范围。如果传入的 AS 编号超出硬件支持范围，会写入错误的寄存器偏移，导致不可预测的行为。

**严重程度**: 🟢 低（当前调用者保证了 AS 编号有效）

---

## 五、风险总结与优先级

| 编号 | 风险 | 严重程度 | 影响 | 修复难度 |
|------|------|----------|------|----------|
| 1 | mmu_map_sg() 无错误回滚 | 🔴 高 | 脏页表残留 | 中 |
| 2 | unmap 不检查 AS 有效性 | 🔴 高 | 页表损坏 | 低 |
| 3 | AS LRU 回收竞态 | 🔴 高 | 页表错乱 | 高 |
| 4 | Shrinker 与 GPU 作业竞态 | 🟡 中 | 访问已释放内存 | 中 |
| 5 | TLB 刷新静默跳过 | 🟡 中 | TLB 条目不一致 | 低 |
| 6 | 缺页错误路径引用计数 | 🟡 中 | 内存泄漏/double-free | 中 |
| 7 | 锁顺序不一致 | 🟡 中 | 潜在死锁 | 低 |
| 8 | mapping release use-after-free | 🟢 低 | 暂无 | 低 |
| 9 | 虚拟地址空间耗尽 | 🟢 低 | 分配失败 | 高 |
| 10 | AS 编号越界 | 🟢 低 | 寄存器错写 | 低 |

## 六、建议修复优先级

1. **立即修复**: 风险 2（unmap AS 检查）和风险 5（TLB 跳过告警）—— 改动小，风险低
2. **短期修复**: 风险 1（map 错误回滚）和风险 6（缺页错误路径）—— 需要增加错误处理
3. **中期修复**: 风险 4（shrinker 竞态）—— 需要引入锁机制
4. **长期重构**: 风险 3（AS LRU）—— 需要重新设计 AS 生命周期管理