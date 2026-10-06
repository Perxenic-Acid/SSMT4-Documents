# 🧩 保留实例化 Draw 语义：避免特殊材质效果丢失

::: tip 太长不看
如果一个模型原本使用 `DrawIndexedInstanced`，不要在 Mod 的 IB Override 中无条件改成普通 `drawindexed`。

同一个 IB 可能同时参与普通 `DrawIndexed` 和实例化 `DrawIndexedInstanced` 的多个 Pass。对于会读取 `SV_InstanceID` 的 Shader，把实例化绘制降级成普通绘制会直接改变 Shader 的输入语义，常见表现包括毛发、绒毛、Shell、多层材质等效果丢失或只剩一层。

在 GIMI / DBMT 中可以按原始 `DRAW_TYPE` 分流：

```ini
[TextureOverride_IB_xxxxxxxx__Component1]
hash = xxxxxxxx
match_first_index = 0
handling = skip
ib = Resource_xxxxxxxx_Component1

if DRAW_TYPE == 4
    ; 原调用是 DrawIndexedInstanced
    drawindexedinstanced = 3030, instance_count, 0, 0, first_instance
elif DRAW_TYPE == 2
    ; 原调用是普通 DrawIndexed
    drawindexed = 3030, 0, 0
endif
```

这里最重要的不是 `3030` 这个示例数字，而是：**保留原 Draw Call 类型、`instance_count` 和 `first_instance`，只替换 Mod 真正需要修改的资源与索引数量。**

另外，部分旧版 Object 导出流程存在“令 Normal 与 Tangent 一致”一类选项。对于依赖切线空间、毛发或 Shell 效果的对象不要启用这种处理。
:::

## 问题表现

模型替换后，主体看起来可以正常显示，但某些特殊效果异常，例如：

- 毛发、绒毛、皮草明显变薄或只剩一层；
- Shell / 多层透明材质效果消失；
- 某些材质在原模型上正常，在 Mod 模型上表现完全不同；
- Shader Hash 看起来没有变化，但效果仍然不对；
- 同一个模型在某些 Pass 正常、另一些 Pass 异常。

这类问题不一定来自贴图或 Shader 本身。一个很容易被忽略的原因是：

> **Mod 替换 IB 后，把游戏原本的 `DrawIndexedInstanced` 强制改成了普通 `DrawIndexed`。**

## 1. `DrawIndexed` 和 `DrawIndexedInstanced` 不是等价的

假设游戏原始调用是：

```text
DrawIndexedInstanced(
    IndexCountPerInstance = 4344,
    InstanceCount         = 15,
    StartIndexLocation    = 0,
    BaseVertexLocation    = 0,
    StartInstanceLocation = 0
)
```

对应 Vertex Shader 又读取：

```hlsl
uint instance_id : SV_InstanceID;
```

并实际把它用于计算：

```hlsl
index = instance_id + offset;
```

那么这 15 个 Instance 就不是单纯用于减少 Draw Call 的性能优化。Shader 实际看到的是：

```text
SV_InstanceID = 0, 1, 2, ... 14
```

对于毛发 Shell 一类效果，可以近似理解成：

```text
Instance 0  -> 最内层
Instance 1  -> 第二层
Instance 2  -> 第三层
...
Instance 14 -> 最外层
```

如果 Mod 把它改成：

```ini
drawindexed = 3030, 0, 0
```

实际发生的是：

```text
DrawIndexedInstanced(..., InstanceCount = 15)
                    ↓
             DrawIndexed(...)
```

于是原本依赖 `SV_InstanceID` 变化得到的多层效果就被破坏了。

## 2. 常见错误：IB Override 把所有 Pass 都统一成 `drawindexed`

常见的 IB Override 类似：

```ini
[TextureOverride_IB_xxxxxxxx__Component1]
hash = xxxxxxxx
match_first_index = 0
handling = skip
ib = Resource_xxxxxxxx_Component1

drawindexed = 3030, 0, 0
```

如果原始 Pass 本来就是 `DrawIndexed`，当然没有问题。但需要注意：

> **一个 IB Hash 不等于一种 Draw 类型。**

同一个 IB 完全可能在一帧中被多个 Render Pass 使用，例如：

```text
Pass A:
DrawIndexedInstanced(4344, 15, ...)

Pass B:
DrawIndexedInstanced(4344, 15, ...)

Pass C:
DrawIndexed(4344, ...)

Pass D:
DrawIndexed(4344, ...)

Pass E:
DrawIndexed(4344, ...)
```

如果只根据 IB Hash 写死 `drawindexed = 3030, 0, 0`，那么 A、B 两个 Pass 的实例化信息也会一起被抹掉。

## 3. 正确做法：按照原始 `DRAW_TYPE` 分流

推荐写法：

```ini
[TextureOverride_IB_xxxxxxxx__Component1]
hash = xxxxxxxx
match_first_index = 0
handling = skip
ib = Resource_xxxxxxxx_Component1

if DRAW_TYPE == 4
    ; 原调用是 DrawIndexedInstanced
    drawindexedinstanced = 3030, instance_count, 0, 0, first_instance
elif DRAW_TYPE == 2
    ; 原调用是普通 DrawIndexed
    drawindexed = 3030, 0, 0
endif
```

其中：

- `instance_count`：继承原调用的 Instance 数量；
- `first_instance`：继承原调用的 `StartInstanceLocation`；
- `3030`：示例中 Mod 新 IB 的索引数量，应替换成自己 Component 的实际 Index Count。

因此，原始：

```text
DrawIndexedInstanced(4344, 15, 0, 0, 0)
```

在替换 IB 后会变成：

```text
DrawIndexedInstanced(3030, 15, 0, 0, 0)
```

变化的是 Index Count，而原来的 Instance 数量仍然得到保留。

对于原始普通 Pass：

```text
DrawIndexed(4344, 0, 0)
```

则继续使用：

```text
DrawIndexed(3030, 0, 0)
```

::: warning 兼容性
本文案例在较旧的 GIMI / DBMT 环境中已经验证上述 `DRAW_TYPE` 分支与 `drawindexedinstanced` 写法可用。

如果使用其他 3Dmigoto Fork 或经过大量修改的 Core，建议先通过 Frame Analysis 确认实际 `DRAW_TYPE` 和最终执行的 Draw Call，不要仅凭版本号猜测。
:::

## 4. 为什么不应该把所有 Pass 反过来统一成 `DrawIndexedInstanced`

一个模型可能本来就同时存在 `DrawIndexedInstanced` 和 `DrawIndexed` 两类 Pass。

正确原则不是“毛发模型都应该用 `DrawIndexedInstanced`”，而是：

> **Mod 应尽可能保持游戏原始 Draw Call 的语义，只替换真正需要修改的参数和资源。**

可以概括为：

```text
原始 Draw 类型              -> 保持
原始 InstanceCount          -> 保持
原始 StartInstanceLocation  -> 保持

VB / IB / Texcoord          -> 按 Mod 需要替换
IndexCount                  -> 按新 IB 修改
```

## 5. 如何确认模型是否存在实例化 Pass

推荐使用 Frame Analysis，而不是只靠肉眼判断材质。

先找到目标 IB：

```text
IASetIndexBuffer(...) hash=xxxxxxxx
```

再检查对应 Draw Call。如果看到：

```text
DrawIndexedInstanced(
    IndexCountPerInstance: ...,
    InstanceCount: ...,
    ...
)
```

说明当前 Pass 使用实例化绘制。

接下来检查对应 VS 是否存在并实际使用：

```hlsl
uint v8 : SV_InstanceID;
```

::: info
仅仅看到 `DrawIndexedInstanced`，还不能证明实例化一定影响最终画面。

但如果 VS 明确读取并使用 `SV_InstanceID`，就应该把 Instance 信息视为 Shader 输入的一部分，而不是可以随意删除的绘制优化。
:::

## 6. 如何用 Frame Analysis 验证修复是否成功

修复前可能看到：

```text
原始游戏调用：
DrawIndexedInstanced(4344, 15, 0, 0, 0)

Mod Override 最终执行：
DrawIndexed(3030, 0, 0)
```

这说明实例化语义已经丢失。

修复后应该看到：

```text
原始游戏调用：
DrawIndexedInstanced(4344, 15, 0, 0, 0)

进入 TextureOverride：
if draw_type == 4: true

最终执行：
DrawIndexedInstanced(3030, 15, 0, 0, 0)
```

对于普通 Pass：

```text
原始：
DrawIndexed(4344, 0, 0)

进入 TextureOverride：
if draw_type == 4: false
elif draw_type == 2: true

最终：
DrawIndexed(3030, 0, 0)
```

这时才能确认新 IB 已进入目标 Pass，同时保留了正确的绘制方式。

## 7. 不要把 Shader Replacement 当成“给 Mod 套原材质”

从 Frame Analysis 中导出的：

```text
xxxxxxxxxxxxxxxx-vs_replace.txt
yyyyyyyyyyyyyyyy-ps_replace.txt
```

不应该仅仅为了“让修改后的模型继续使用原 Shader”而放进 `ShaderFixes`。

替换 VB / IB 并不意味着当前 Pass 原来的 VS / PS 会自动消失。排查时应先比较原版和 Mod 的 Shader Hash；如果 VS / PS 本来就没有变化，那么问题通常不在“Shader 没有套上”。

部分 GIMI / DBMT Core 还会通过：

```text
ShaderRegex
    ↓
CommandList
    ↓
checktextureoverride
```

触发模型资源替换。用户 Shader Replacement 可能影响这条链，因此没有真正修改 Shader 行为的需求时，建议排查阶段先移除相关 `*_replace.txt`，用游戏原生 Shader 建立基线。

## 8. Object 导出时检查 Normal / Tangent

如果 Draw 类型已经正确，但特殊材质仍然异常，下一步应该检查顶点语义：

- `POSITION`
- `NORMAL`
- `TANGENT`
- `TEXCOORD`
- `COLOR`

部分旧版本的 Object 导出流程中存在类似“令 Normal 与 Tangent 一致”的选项。对于依赖切线空间、法线方向、毛发或 Shell 偏移的材质：

> **不要启用会强制 Normal 与 Tangent 相同的导出选项。**

新版本工具中这个选项的名称、默认值，或者是否仍然存在，可能已经变化。真正需要检查的是导出的最终数据。

一般情况下，下面这种结果应该引起警惕：

```text
NORMAL.xyz == TANGENT.xyz
```

Normal 与 Tangent 表达的是不同方向，不应该被无条件复制成同一个向量。

## 9. 推荐排查顺序

1. **查目标 IB 的全部 Pass**：不要默认一个 IB Hash 只对应一次 Draw。
2. **记录每个 Pass 的原始 Draw 类型**：区分 `DrawIndexed` 与 `DrawIndexedInstanced`。
3. **检查 VS 是否使用 `SV_InstanceID`**：如果实际使用，就必须保留 Instance 语义。
4. **检查 Mod 的 IB Override**：看是否无条件写死了 `drawindexed = ...`。
5. **用 `DRAW_TYPE` 分流**：Instanced Pass 使用 `drawindexedinstanced`，普通 Pass 保持 `drawindexed`。
6. **确认 TextureOverride 真正进入目标 Pass**：Frame Analysis 中应看到相应的 `checktextureoverride` 和 Resource 替换。
7. **最后再检查顶点数据**：特别关注 Normal、Tangent、UV、Vertex Color。

## 10. 核心原则

> **模型 Mod 不应该只保留“画了多少个三角形”，还应该尽可能保留原 Draw Call 的语义。**

当 Shader 使用 `SV_InstanceID` 时，`InstanceCount` 本身就是 Shader 输入的一部分。

对于同一个 IB 同时参与普通与实例化 Pass 的情况，优先使用：

```ini
if DRAW_TYPE == 4
    drawindexedinstanced = ..., instance_count, ..., first_instance
elif DRAW_TYPE == 2
    drawindexed = ...
endif
```

根据调用现场保留原始绘制语义，而不是把所有 Pass 粗暴归一化。
