# Panfrost DRM 驱动崩溃问题排查报告

## 问题描述

在 Spreadtrum UMS9620 平台上运行 `RSRenderThread` 进程时，Panfrost DRM 驱动发生 NULL 指针解引用崩溃。

## 崩溃现场分析

### 1. 错误信息摘要

```
Unable to handle kernel NULL pointer dereference at virtual address 000000000000001e0
...
Internal error: Oops: 96000005 [#1] PREEMPT SMP
PC: mutex_lock+0x30/0x68
LR: panfrost_gem_mapping_get+0x2c/0xd0
```

### 2. 调用栈

| 序号 | 函数地址 | 函数名 |
|------|----------|--------|
| 0 | `0x...` | `mutex_lock+0x30/0x68` |
| 1 | `0x...` | `panfrost_gem_mapping_get+0x2c/0xd0` |
| 2 | `0x...` | `panfrost_ioctl_submit+0x480/0x5d8` |
| 3 | `0x...` | `drm_ioctl_kernel+0xd0/0x120` |
| 4 | `0x...` | `drm_ioctl+0x414/0x7e8` |

### 3. 崩溃位置源码

崩溃发生在 [panfrost_gem_mapping_get](file:///workspace/panfrost/panfrost_gem.c#L56-L72) 函数中调用 `mutex_lock()` 时：

```c
// panfrost_gem.c 第 56-72 行
struct panfrost_gem_mapping *
panfrost_gem_mapping_get(struct panfrost_gem_object *bo,
                         struct panfrost_file_priv *priv)
{
    struct panfrost_gem_mapping *iter, *mapping = NULL;

    mutex_lock(&bo->mappings.lock);  // <-- 崩溃点！
    list_for_each_entry(iter, &bo->mappings.list, node) {
        if (iter->mmu == &priv->mmu) {
            kref_get(&iter->refcount);
            mapping = iter;
            break;
        }
    }
    mutex_unlock(&bo->mappings.lock);

    return mapping;
}
```

## 问题根因分析

### 1. 崩溃触发条件

**NULL 指针解引用** 发生在访问 `bo->mappings.lock` 时，表明传入的 `bo` 指针可能为 NULL，或者 `bo->mappings` 结构中的字段损坏。

### 2. 调用链分析

让我们看一下 [panfrost_ioctl_submit](file:///workspace/panfrost/panfrost_drv.c#L239-L296) 函数是如何调用这个接口的：

```c
// panfrost_drv.c 第 124-173 行
static int
panfrost_lookup_bos(struct drm_device *dev,
                   struct drm_file *file_priv,
                   struct drm_panfrost_submit *args,
                   struct panfrost_job *job)
{
    // ...
    for (i = 0; i < job->bo_count; i++) {
        struct panfrost_gem_mapping *mapping;

        bo = to_panfrost_bo(job->bos[i]);  // <-- 转换
        mapping = panfrost_gem_mapping_get(bo, priv);  // <-- 传入 bo
        if (!mapping) {
            ret = -EINVAL;
            break;
        }
        // ...
    }
```

### 3. 潜在问题场景

1. **场景 A**：传入的 `bo` 指针本身为 NULL
2. **场景 B**：传入的 GEM 对象不是正确的 `panfrost_gem_object` 类型
3. **场景 C**：GEM 对象在被使用之前已经被释放，成为悬挂指针（dangling pointer）
4. **场景 D**：内存池分配失败，导致 GEM 对象部分初始化（从崩溃前的日志看）

## 修复方案

### 修复 1：在 `panfrost_gem_mapping_get` 中增加 NULL 指针检查

修改 [panfrost_gem.c](file:///workspace/panfrost/panfrost_gem.c#L56-L72)，在函数开头增加 NULL 指针检查：

```c
struct panfrost_gem_mapping *
panfrost_gem_mapping_get(struct panfrost_gem_object *bo,
                         struct panfrost_file_priv *priv)
{
    struct panfrost_gem_mapping *iter, *mapping = NULL;

    // 增加 NULL 指针检查
    if (!bo || !priv) {
        pr_err("panfrost: NULL pointer passed to panfrost_gem_mapping_get()\n");
        return NULL;
    }

    mutex_lock(&bo->mappings.lock);
    list_for_each_entry(iter, &bo->mappings.list, node) {
        if (iter->mmu == &priv->mmu) {
            kref_get(&iter->refcount);
            mapping = iter;
            break;
        }
    }
    mutex_unlock(&bo->mappings.lock);

    return mapping;
}
```

### 修复 2：在 `panfrost_lookup_bos` 中增加转换后的检查

修改 [panfrost_drv.c](file:///workspace/panfrost/panfrost_drv.c#L124-L173)，在调用 `to_panfrost_bo()` 后检查转换结果：

```c
static int
panfrost_lookup_bos(struct drm_device *dev,
                   struct drm_file *file_priv,
                   struct drm_panfrost_submit *args,
                   struct panfrost_job *job)
{
    // ...
    for (i = 0; i < job->bo_count; i++) {
        struct panfrost_gem_mapping *mapping;

        // 增加对 job->bos[i] 的 NULL 检查
        if (!job->bos[i]) {
            dev_err(dev, "panfrost: NULL GEM object at index %d\n", i);
            ret = -EINVAL;
            break;
        }

        bo = to_panfrost_bo(job->bos[i]);
        
        // 增加对转换后 bo 的检查
        if (!bo) {
            dev_err(dev, "panfrost: Invalid GEM object type\n");
            ret = -EINVAL;
            break;
        }

        mapping = panfrost_gem_mapping_get(bo, priv);
        if (!mapping) {
            ret = -EINVAL;
            break;
        }
        // ...
    }
```

### 修复 3：增加 `to_panfrost_bo` 的类型安全检查

修改 [panfrost_gem.h](file:///workspace/panfrost/panfrost_gem.h#L53-L56)，增加类型安全的转换宏：

```c
static inline
struct panfrost_gem_object *to_panfrost_bo(struct drm_gem_object *obj)
{
    // 增加对 funcs 的验证，确保这是我们的对象
    if (WARN_ON_ONCE(!obj || obj->funcs != &panfrost_gem_funcs))
        return NULL;
        
    return container_of(to_drm_gem_shmem_obj(obj), struct panfrost_gem_object, base);
}
```

## 建议的完整修复步骤

### 1. 短期修复：立即防止崩溃

实现上述的修复 1（NULL 指针检查），这将防止崩溃并提供错误日志。

### 2. 中期修复：增加调试日志

在关键位置增加更多的日志，帮助定位为什么会出现无效的 BO：

```c
// panfrost_gem.c
struct panfrost_gem_mapping *
panfrost_gem_mapping_get(struct panfrost_gem_object *bo,
                         struct panfrost_file_priv *priv)
{
    struct panfrost_gem_mapping *iter, *mapping = NULL;

    if (!bo) {
        pr_err("panfrost: bo is NULL in panfrost_gem_mapping_get()\n");
        return NULL;
    }

    if (!priv) {
        pr_err("panfrost: priv is NULL in panfrost_gem_mapping_get()\n");
        return NULL;
    }

    mutex_lock(&bo->mappings.lock);
    list_for_each_entry(iter, &bo->mappings.list, node) {
        if (iter->mmu == &priv->mmu) {
            kref_get(&iter->refcount);
            mapping = iter;
            break;
        }
    }
    mutex_unlock(&bo->mappings.lock);

    return mapping;
}
```

### 3. 长期修复：调查并修复 BO 状态问题

从崩溃前的日志看，系统遇到了内存分配问题：

```
ion_system_heap_allocate, size:10162176, time:2180us,
```

这可能表明系统正在经历内存压力，这可能导致 GEM 对象被过早释放或部分初始化。需要配合系统内存管理排查。

## 相关代码文件参考

| 文件 | 路径 | 说明 |
|------|------|------|
| panfrost_gem.c | [panfrost_gem.c](file:///workspace/panfrost/panfrost_gem.c) | GEM 内存对象管理 |
| panfrost_gem.h | [panfrost_gem.h](file:///workspace/panfrost/panfrost_gem.h) | GEM 对象定义 |
| panfrost_drv.c | [panfrost_drv.c](file:///workspace/panfrost/panfrost_drv.c) | 驱动主入口 |
