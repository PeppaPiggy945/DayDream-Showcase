# 一、内核–插件边界：Master 的演变史

> 这条线的主题是**持续消除特权通道，把"加一个插件的成本"压到最低**。
> 系统的天花板不取决于代码量，取决于接入成本——如果加插件要改十个文件，上限就是十个插件。

## 阶段总览

| 阶段 | 时间 | 状态 | 问题 |
|---|---|---|---|
| 原型 | 5/26–5/29 | 无插件概念，一个 Python 对象 = 人格 + 记忆 + 调度 | 每加一个功能改十几处 |
| 插件化成型 | 6/16–6/21 | HumanMaster + GATHER→ORDER→EXECUTE 管道，插件用 `tick_intent()` 声明意图 | Master 仍是知道所有插件的上帝对象；自己的动作走特权通道绕过管道 |
| 管道收敛 | 6/23–6/25 | 见下 | — |
| Master 统一 | 7/4 | 见下 | — |
| 声明化终点 | 8 月 | 见下 | — |

## 管道收敛（第 3–4 周，6/23–6/25）

自我行动、外部事件、分类、消费，全部走同一条标准管道：

```
之前：self-action → 特权通道 → handler 直调 → 不经过标准管道
之后：self-action → wrap_inbox → typed_actions → tick_intent → 标准管道
```

所有 inbox 条目先进 wrapper 分类成 `typed_actions`，所有插件通过 `tick_intent()` 在**声明的优先级**上消费，Master 只做排序和执行。Master 不再知道 `move` 是什么、`message` 是什么——这才是真正的插件化。

同时把三种认知模式拆成三路独立的模式与调度策略，而不是一个 LLM 一个 prompt 全包：

- **agency**：只响应外部事件，think-window 门控；
- **autopilot**：独立 LLM，idle-timer 指数退避，产生自发行为；
- **heartbeat_tick**：纯安全网 pacer，不做 LLM 调用。

`Plugin` 基类提取后，写一个状态同步类插件只需声明三四个类属性。这一阶段的性质是**工具建设**：不在给系统加功能，而在降低新功能的接入成本。

## Master 统一（第 6 周，7/4）

原来 `HumanMaster` 和 `WorldMaster` 是两个子类，每加一种实体类型就要复制一条继承树。重构为**单一 Master + slot 变体系统**：

- 行为差异点被抽象成 **9 个 slot**：`event_header`（事件头渲染）、`accept_event`（事件接受）、`self_perspective` / `code_perspective`（视角翻译）、`enact_action`（动作执行）、`route_event`（事件路由）、`before_gather`、`should_skip`、`skip_reason`；
- 每个实体在 `config.json` 里声明 `"slots": {"*": "character"}` **自由组合变体**；
- 未注册的 slot 落到内置 identity / noop 默认。

结果：`Master.on_tick` **零 `entity_type` 分支**。密室辩论场景不开空间 / 生理插件、战斗场景挂载怪物实体，Master 一行不改。

## 声明化终点（8 月）

两步走。

**其一，插件组合成包。** `pack.json` + include 链，如 `fantasy_human = basic_human + 战斗包 + 空间包`。角色创建时按包选择 → 自动展开插件集；插件目录扫描 → 冲突校验 → **深度冻结**成不可变 `PluginCatalog` 再注入运行时，进程级注册表彻底消失，换存档零污染。

**其二，插件语言 v2。** 插件 = 触发谓词 + 动作链的 ECA（Event-Condition-Action）规则声明 + 可组合积木（工厂函数产出闭包）。三层书写复杂度平滑：

| 层级 | 适用 | 写法 |
|---|---|---|
| L1 | 简单插件 | 一条规则声明 |
| L2 | 中等插件 | 拆开积木自由拼 |
| L3 | 复杂插件 | 重写 `tick_intent` 逃逸 |

35 个插件完成迁移，**内核零改动**。

## 跨插件的 pull 钩子

四个跨插件扩展点被统一为同一个"**插件声明、消费者按需拉取**"模式：

| 钩子 | 作用 |
|---|---|
| `outputs` | 维度渲染（4 级信息密度） |
| `dynamic_rule` | 动态规则注入 |
| `action_spec` | 动作菜单 |
| `tools` | Agent 工具面 |

代码里不存在"这个插件需要特殊处理"的分支。

## 结论

演变史里反复验证了同一个规律：

> **每一轮重构都在删除一个特权通道或伪概念，删完之后系统能力反而变强。**

从"一个功能改十几处"到"声明三四个类属性"，接入成本的下降直接抬高了系统的天花板。
