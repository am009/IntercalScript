# `types6.ics` 类型系统实现说明

`types6.ics` 是 IntercalScript 编译器第六代类型系统与约束求解器的核心实现，承担类型节点的建模、合并、复制以及约束求解职责。文件中的函数采用基于工厂的“正/负”类型节点设计，并利用图结构来表示类型流动关系。

本文档逐个函数说明其用途，并详细描述内部每段代码的逻辑流程。

## 通用辅助闭包

- `refcb = funct(n) n.ref() end`：简单的回调闭包，调用节点的 `ref()` 方法并返回结果，用于在集合中复用引用。
- `_idcb = funct(n) n._id end`：获取节点 `_id` 的回调，常用于调试输出。

## `TypeSet`

`TypeSet future = funct(s) { ... } end` 构造一个包裹类型节点集合的对象，提供集合运算与遍历能力。

- `copy()`：
  - 基于当前集合 `this.s`，通过 `map(..., refcb)` 为每个元素调用 `ref()`，复制引用计数。
  - 将新集合传入 `TypeSet` 构造函数，返回新的 `TypeSet` 实例。
- `deepCopy(copies)`：
  - 对集合中每个节点调用 `copies._getCopy(n).ref()`，即使用外部的 `Copier` 获取或创建深拷贝节点。
  - 将得到的深拷贝集合封装成新的 `TypeSet`。
- `drop(droplist)`：
  - 将内部集合直接追加到 `droplist`，以便稍后统一调用 `_dropShallow` 回收。
- `debug()`：
  - 利用 `_idcb` 将集合中每个节点映射为 `_id`，便于日志打印。
- `merge(other)`：
  - 逐个遍历 `other` 集合中的节点，如果当前集合缺少该节点，则调用 `n.ref()` 增加引用并插入。
  - 若发生新增则标记 `changed` 为 `true`。
- `_getChildren(out)`：
  - 把集合里的迭代器结果直接追加到 `out`，用于构建整体 GC 扫描列表。
- `iter()`：
  - 返回内部可变集合本身，提供统一的迭代接口。

## `FieldSet`

`FieldSet future = funct(fields, expls) { ... } end` 管理对象字段与其类型集合，同时追踪诊断信息。

- `union(other)`：
  - 对 `other.fields` 逐项遍历，若当前集合已有同名字段则合并 `TypeSet`（调用 `merge`），否则复制 `other` 的字段与解释。
  - 任意修改都会把 `changed` 设为 `true`。
- `intersect(other, droplist, default-expl)`：
  - 遍历当前字段集，若在 `other` 中找到同名字段则合并类型集合，未找到则删除该字段：
    - 先调用 `n.drop(droplist)` 将旧集合加入待回收列表。
    - 从 `this.fields` 删除字段。
    - 使用 `other` 提供的解释或 `default-expl` 记录原因。
  - 若发生删除或合并改动都视为 `changed`。
- `copy()`：
  - 逐字段调用 `n.copy()` 复制 `TypeSet`（浅拷贝），保持 `expls` 共享。
- `deepCopy(copies)`：
  - 与 `copy()` 类似，但改为 `n.deepCopy(copies)` 以创建独立节点。
- `_getChildren(out)`：
  - 遍历所有字段的 `TypeSet`，调用 `_getChildren` 嵌套收集。

`new-field-set = funct() FieldSet(undef, undef) end`：生成一个字段集合，占位字段与解释均为 `undef`。

## 节点结构体

`Type future = (...) end` 内定义多种节点类型，用于建模类型图。

### `SingletonNode`

`SingletonNode = funct(neg) { ... } end` 表示单值类型（null/undef/string/bool）。

- 初始化时根据 `neg` 设置 `present` 缺省值（负节点表示“缺失集合”，正节点表示“存在集合”），并清空诊断信息。
- `mergeFrom(other)`：
  - 当 `other.present` 与当前 `present` 不一致、且与 `neg` 的状态不冲突时，用 `other` 的 `present`、`expl`、`expl-type` 覆盖当前值。
  - 若发生覆盖返回 `true`，否则 `false`。
- `copyFrom(other, copies)`：
  - 简单复制 `other` 的 `present` 与解释信息；单节点不涉及递归结构，因此不需要使用 `copies`。

### `NumberNode`

`NumberNode = funct(neg) { ... } end` 扩展了 `SingletonNode` 的逻辑，通过 `level` 区分 `int`（1）与 `number`（2）。

- 构造时若为负节点则默认 level=2（上界），正节点默认 level=0（下界）。
- `mergeFrom(other)`：
  - 若正节点遇到更高的 level，或负节点遇到更低的 level，则采用 `other` 的 level 与解释，并同步 `present`。
  - 返回布尔值表示是否更新。
- `copyFrom(other, copies)`：
  - 同步 `present`、`level` 以及解释信息。

### `FuncNode`

`FuncNode = funct(neg) { ... } end` 表示函数类型节点，包含参数、返回值、`unsafe` 标记等。

- 构造时创建空的参数 `FieldSet` 与返回值 `TypeSet`，并基于 `neg` 初始化 `present` 和 `is-top`。
- `mergeFrom(other, droplist)`：
  - 首先合并 `present` 和解释开关；`unsafe` 标记也做同样处理。
  - 合并返回类型集合（调用 `TypeSet.merge`）。
  - 对参数字段：
    - 在负节点（即排除语义）下，如果 `other` 不是 “顶部” 函数就要收窄参数集合：
      - 若当前是顶点则直接复制 `other` 的参数；否则执行 `intersect`，并用 `droplist` 记录被删除的 TypeSet。
    - 在正节点下则采取 `union` 扩大参数需求。
  - 根据任何更新返回 `changed`。
- `copyFrom(other, copies)`：
  - 复制 `present`、解释、`unsafe` 状态。
  - 深拷贝参数与返回类型：`args.deepCopy(copies)` 与 `ret.deepCopy(copies)`。
- `_getChildren(out)`：
  - 若不是 `is-top`，将参数字段的子节点加入 `out`；随后无条件添加返回值集合的子节点。

### `ObjNode`

`ObjNode = funct(neg, _id) { ... } end` 表示对象类型节点，包含读字段集合、写字段集合以及解释。

- 构造时根据 `neg` 初始化 `present`，创建读写字段集合。
- `explMissingRead(lbl)` / `explMissingWrite(lbl)`：
  - 优先返回字段级解释，若当前字段没有专属解释则退回到节点级解释。
- `debug()`：
  - 返回 `[节点 ID, neg 标记, 读取字段名列表]`，用于调试日志。
- `mergeFrom(other, droplist)`：
  - 负节点：当当前对象存在时，如果 `other` 也存在，则对读写字段执行 `union`；否则将 `present` 设为 `false` 并继承解释。
  - 正节点：如果 `other` 表示对象存在，则：
    - 当前若已存在则对读写字段执行 `intersect`（利用 `droplist` 记录被丢弃的分支）。
    - 若当前不存在则直接复制 `other` 的读写集合并标记为存在。
  - 返回布尔值指示是否发生变化。
- `copyFrom(other, copies)`：
  - 复制 `present` 与解释，并对 `reads`、`writes` 执行深拷贝。
- `_getChildren(out)`：
  - 将读字段与写字段的子节点递归加入列表。
- `_shallowReadCopy()`：
  - 创建一个新的 `ObjNode`，复制 `present`、解释与读字段集合（浅拷贝），但不复制写集合（默认为空）。主要服务于 `CaseNode`。

### `CaseNode`

`CaseNode = funct(neg) { ... } end` 表示枚举 / case 类型，包含一个标签到 `ObjNode` 的映射。

- 构造时初始化 `present`、`cases`、`itops`（case 可选集），以及解释映射。
- `getExpl(lbl)`：
  - 返回指定标签在 `expls` 中记录的解释，若无则回退到节点级解释。
- `mergeFrom(other, droplist, filter)`：
  - 同步 `present`/解释信息，但不计入 `changed`。
  - 负节点 (`neg == true`)：
    - 根据 `filter.cases` 与 `other.itops`，移除已排除的标签；对需要保留的标签执行合并或删除。
    - 如果标签消失，则调用 `_getChildren` 将原有对象的子节点推入 `droplist` 并删除映射，同时更新解释。
    - 若 `removed_itops` 有补充，则用 `_shallowReadCopy` 拉入 `other` 的对象快照。
  - 正节点：
    - 遍历 `other` 的 case，若过滤器允许则：
      - 已存在就合并对象；
      - 不存在则复制对象快照并保存解释。
  - 返回 `changed` 用于判断是否触发进一步传播。
- `copyFrom(other, copies)`：
  - 复制 `present`、解释、`expls`、`itops`。
  - 逐 case 新建 `ObjNode`，再通过 `copyFrom` 深拷贝。
- `_getChildren(out)`：
  - 将所有 case 对象的子节点加入列表。

## `Type` 主体

`Type` 是对 GC-able 节点的封装，构造时调用 `GCable.prototype` 提供的生命周期方法。

- 初始化字段包括：
  - `_id`：全局唯一 ID。
  - `neg`：标记该节点是否代表“集合补集”。
  - `flow`/`pflow`：保存类型流出节点的集合（完全/部分过滤）。
  - `solve`：记录联合求解图中的相互依赖节点。
  - `signpost`：当节点被 GC 前发现仍在流图中时设为 `true`，以保留最小信息。
  - 六类子节点：`null`、`undef`、`string`、`bool`、`num`、`func`、`obj`、`enum`。

### `mergeFromNoFlow(other, droplist, filter)`

- 若 `this == other`，直接返回 `false`。
- 根据 `filter` 控制选择性地对各子节点调用 `mergeFrom`：
  - 对 `null`/`undef` 单节点根据 `filter.null/undef` 合并。
  - 对其它种类在 `filter.other` 为真时才合并。
- 在函数、对象、case 合并中传递 `droplist` 与 `filter`。
- 返回是否发生变化。

### `mergeFrom(other, droplist, filter)`

- 首先调用 `mergeFromNoFlow` 合并节点内容。
- 然后把 `other.flow` 中的节点逐一加入当前节点的流图（调用 `addPFlow`，传入整型过滤器）。
- 对 `other.pflow`（带过滤器的部分流）则取交集过滤器后加入。

### `mergeIntoOtherFlow(other, droplist)`

用于在求解过程中传播约束：
- 若双方都有流信息，则相互加入 `solve` 集合，表示合并时需同步。
- 使用栈执行 DFS：
  - 初始栈包含 `(true, other, FILTER.ALL)`，表示需要扩展开 `other`。
  - 遇到 `expand == true` 时，将 `other` 的 `pflow` 与 `flow` 邻接节点入栈，标记 `expand=false` 表示后续处理。
  - 当 `expand == false` 时：
    - 若未访问过该节点：
      - 如果过滤器是全量，直接调用 `mergeFromNoFlow`，并在成功或该节点有 `signpost` 时，把其 `solve` 邻居以 `expand=true` 再次入栈。
      - 若过滤器非全量，则跟踪 `seen-partial`，仅当新过滤器提供了额外信息时才合并。
- 算法保证部分过滤器只会逐步扩大，不会重复求解。

### `copyFrom(other, copies)`

- 若 `signpost` 为 `false`，则对所有子节点调用各自的 `copyFrom` 实现；`signpost` 节点避免拷贝以保留惰性状态。
- 对流结构：
  - 通过 `copies._getCopy` 将 `other.flow`、`pflow`（键映射）、`solve` 转换为指向新节点的集合。

### 流图辅助方法

- `_addFlowSingle(other)`：把 `other` 加入 `flow` 并从 `pflow` 删除（全量流优先级更高）。
- `_addPFlowSingle(other, filter)`：若不存在全量流且过滤器非空，则与已存在过滤器并集；若合并后为全量则降级为 `_addFlowSingle`。
- `addFlow(other)`：双向调用 `_addFlowSingle`，建立对称的完全流。
- `addPFlow(other, filter)`：双向建立部分流。
- `hasFlow()`：检测当前节点是否连接任何流或部分流。
- `_getChildren()`：聚合函数/对象/case 节点的子节点引用，用于 GC。
- `_free()`：
  - 如果仍存在 `solve` 或流（包含部分流）关系，则保留节点作为 `signpost`。
  - 否则调用 `_gc_free()` 更新颜色，并清理与其它节点的 `solve`/`flow`/`pflow` 关联。
- 最后把 `GCable` 原型中的 `ref`、`drop` 等方法挂载到对象上。

## `finish-droplist`

`finish-droplist = funct(droplist, gc)`：持续弹出 droplist 中的节点，对每个节点调用 `_dropShallow(droplist, gc)`，以递归方式释放类型节点及其依赖。

## `Copier`

`Copier = funct(factory) { ... } end` 负责在类型工厂中制作深拷贝。

- `_getCopy(node)`：
  - 如果在 `map` 中已有缓存则直接返回；
  - 否则调用 `factory._add(Type(node.neg))` 构建新节点，将 `(node, copy)` 压栈并放入 `droplist`，以等待后续完成。
- `_finishPending()`：
  - 不断弹出 `stack` 中的 `(node, copy)`，对于未在 `finished` 集合中的 copy：
    - 调用 `copy.copyFrom(node, this)` 填充内容。
    - 将 copy 标记到 `finished`，避免重复。
- `copy(node)`：
  - 首先通过 `_getCopy` 获得占位 copy，随后 `finishPending()` 以保证所有依赖节点完成。
  - 将原节点压入 `droplist`（表示调用者转移所有权），并调用 `copy.ref()` 返回拥有引用的拷贝。
- `drop()`：
  - 调用 `finish-droplist` 处理内部 `droplist`，释放复制过程中创建的临时对象。

## `Merger`

`Merger = funct(factory) { ... } end` 用于创建合并后的类型节点。

- `mergeOwned(lhs, rhs)`：
  - 把 `lhs`、`rhs` 压入 `droplist`，表示结果将接管所有权。
  - 按照左侧 `neg` 创建新节点。
  - 首先调用 `mergeFrom(lhs, ...)`，再用 `mergeFrom(rhs, ...)`，以便整合两个节点的所有约束。
  - 返回合并后的节点。
- `finish()`：与 `Copier` 类似，调用 `finish-droplist` 清理临时引用。

## `Solver`

`Solver = funct(reporter) { ... } end` 提供类型约束求解器，结合 DFS 和记忆化处理合一。

- `reportError(message, src-expl, src-msg, dest-expl, dest-msg)`：
  - 若任意解释为 `null` 或 `_`，视为内部错误并调用 `reporter.internalError()`。
  - 否则将信息拼接成多行消息，并通过 `reporter.onError` 抛出。
- `checkHeadSub(lhs, rhs, lhs-expl-type)`：
  - 获取两侧的解释，当左值要求存在而右值不存在时，格式化报错消息（说明期望类型和来源）。
  - 若解释缺失则触发内部错误。
- `checkHead(lhs, rhs, lhs-expl-type)`：
  - 在左节点 `present` 但右节点 `present` 为假时调用 `checkHeadSub`。
- `checkHeads(lhs, rhs)`：
  - 对 null/undef/string/bool/func/obj/enum 等所有类型头部执行 `checkHead`。
  - 对数字类型检查 `level`：若左边需求等级高于右边提供，则报错（区分 `number` 与 `int`）。
- `pushSets(ts1, ts2)`：
  - 将 `ts1` 与 `ts2` 的笛卡尔积入栈，以便后续求解时处理每对节点。
- `biunifyObjects(lo, ro)`：
  - 遍历右对象的读字段：
    - 若左对象缺少该字段则报错；
    - 否则将对应的类型集对推入栈。
  - 遍历右对象的写字段：
    - 若左侧缺失，先判断左侧是否完全不存在该字段，分别报缺失或“写入不可变字段”的错误。
    - 若存在，则把右写集合与左写集合的节点对入栈。
- `solve(droplist, pair)`：
  - 将初始对 `(lhs, rhs)` 入栈。
  - 循环弹栈，对于未在记忆表 `memo` 中出现的 pair，记录后调用 `_solvePair`。
- `_solvePair(lhs, rhs, droplist)`：
  - 平行调用 `lhs.mergeIntoOtherFlow(rhs, ...)` 与 `rhs.mergeIntoOtherFlow(lhs, ...)`，触发必要的合并。
  - 调用 `checkHeads` 验证顶层类型是否兼容。
  - 函数类型处理：
    - 如果右侧函数不是“顶对象”且左侧函数存在，则检查安全上下文 (`unsafe`)。
    - 遍历左函数参数，根据标签查找右函数的实际参数：
      - 缺失 `this` 时提示“缺少接收者”，否则提示缺少位置参数；
      - 匹配成功则将类型集合压栈以继续求解。
    - 最后对返回值集合也执行 `pushSets`。
  - 对象类型：若两边都声称存在对象，则调用 `biunifyObjects`。
  - Case 类型：若左侧枚举存在且右侧过滤器并非 `all`，则逐 case 检查，缺失时报错，存在时递归 biunify 对象。

## 其它基础函数

- `_c` / `_count()`：
  - `_c` 是一个带可变字段的包装对象，`_count()` 每次读取 `_c.c` 并自增，用于分配全局递增 ID。
- `Universe = funct() { ... } end`：
  - 创建类型宇宙环境，包含唯一 `_id`、`GarbageCollector`、以及 `locked` 标记。
- `MONO = Universe()`：
  - 创建一个专用于单态上下文的全局 `Universe`。

## `Factory`

`Factory = funct(uni, expl) { ... } end` 是类型节点的构造器和工具集合。

### 基础构造

- `_add(node)`：在当前工厂中登记一个节点，直接返回该节点（在额外的实现中可以挂接 GC）。
- `_Bot()` / `_Top()`：分别创建正（底类型）与负（顶类型）节点。
- `Bot()` / `Top()`：公开方法，直接转调 `_Bot()`、`_Top()`。

### `_NBot(expl, expl-type)`

- 创建一个“负底”节点，先生成负节点 `Type(true)`。
- 将所有子类型的 `present` 标记设为 `false`，并填入统一解释 `expl` 与解释类型 `expl-type`，表示该节点要求排除对应类型。
- 函数类型额外重置为非 top，并装配空字段集。
- 枚举类型将 `itops` 设为空集且记录解释。

### 原子类型构造

- `PNull/PUndef/PStr/PBool/PInt/PNum`：
  - 先调用 `_Bot()` 得到正节点，再把对应的 `present` 设为 `true` 并写入解释。
- `NStr/NBool/NInt/NNum/NStrOrNum`：
  - 通过 `_NBot()` 创建负节点，再将允许的类型标记为存在（并清除解释，表示已经满足要求）。
  - 对数字、字符串等指定 level 或多字段组合。

### 函数/对象构造

- `_FieldSet(args, expls)`：
  - 将参数映射转换为 `FieldSet`，其中每个值都封装为 `TypeSet`。
- `PFunc(args, ret, expl, unsafe-expl)`：
  - 生成底节点，打开 `func.present`。
  - 调用 `FieldSet`/`TypeSet` 包裹传入的参数、返回值集合。
  - 标记 `unsafe` 并记录解释位（若 `unsafe-expl != null` 则视为不安全函数）。
- `NFunc(args, ret, expl, unsafe-allowed)`：
  - 使用 `_NBot` 创建负节点，允许函数存在但不是 “top”。
  - 参数通过 `_FieldSet` 包装，返回值集合只含单一节点 `ret`。
  - `unsafe-allowed` 决定 `unsafe` 标志；若不允许则沿用 `expl`。
- `_listToFields(fields, expl)`：
  - 若提供 `expl`，为每个字段生成统一的解释映射。
  - 调用 `_FieldSet` 将字段映射转换为 `FieldSet`。
- `PObj(rpairs, wpairs, expl)`：
  - 底节点，标记对象存在；读写字段分别调用 `_listToFields`，读字段解释为 `null`，写字段同样初始化为空解释。
- `NObj(rpairs, wpairs, expl)`：
  - 从 `_NBot` 初始化，允许对象存在，并附带统一解释。

### Case 构造

- `PCase(type, tag, expl)`：
  - 创建底节点，标记 case 类型存在。
  - 将传入的对象类型节点使用 `_shallowReadCopy` 转换为只读对象，并映射到 `tag`。
  - 写入解释后丢弃原对象 (`type.drop`)。
- `NCase(type, tag, expl)`：
  - 同样利用 `_NBot` 创建负节点，将 case 写入 neg 集合。

### 变量与过滤

- `VarPair()`：
  - 创建一个 top 节点 `n` 与一个 bot 节点 `p`。
  - 调用 `p.addFlow(n)` 在两者间建立流动关系，用作多态类型变量的上、下界对。
  - 返回 `{p, n}`。
- `filter(node, filter)`：
  - 创建临时 droplist 和一个与 `node.neg` 相同的空节点 `temp`。
  - 调用 `temp.mergeFrom(node, droplist, filter)` 应用过滤器。
  - 对原节点调用 `drop()`，释放旧引用，并返回过滤后的节点。

### 其它字段

- `poly-count`：记录当前工厂构建的多态变量数量。
- `gc`：引用 `uni.gc`，供 `finish-droplist` 与节点释放使用。

## 总结

`types6.ics` 通过上述函数协同工作，构建了一个带 GC 的图结构类型系统。`Factory` 与 `Type` 负责节点的产生与合并，`Copier`/`Merger` 提供复合操作，`Solver` 负责约束求解与错误诊断，而 `TypeSet`、`FieldSet`、各种节点结构为这些操作提供了底层数据抽象。
