# 文档模板与规范

本目录定义 unifabric 各类文档的标准模板。新建或整改文档时,按文档类型选择对应模板,
遵循其骨架与约束。分类体系参照 CNCF TechDocs 对 sandbox 项目的要求
(Conceptual / Task-based / Reference)。

## 文档类型与目录归属

| 类型 | 模板 | 目录 | 面向读者 | 回答的问题 |
|---|---|---|---|---|
| 项目 README | [project-readme.md](./project-readme.md) | 仓库根目录 | 评估者 | 这是什么?值得试吗? |
| 概念/架构 | [concepts.md](./concepts.md) | `docs/concepts/` | 评估者、进阶用户 | 它如何运作? |
| 快速开始 | [getting-started.md](./getting-started.md) | `docs/` | 零基础新用户 | 怎么装起来跑通? |
| 功能使用 | [usage.md](./usage.md) | `docs/usage/` | 已安装的用户 | 这个功能怎么用好? |
| 参考手册 | [reference.md](./reference.md) | `docs/reference/` | 查阅字段/参数的用户 | 这个字段是什么? |
| 设计文档 | [design.md](./design.md) | `docs/design/` | 贡献者、维护者 | 为什么这么设计? |

各类型职责互斥,不混写:

- getting-started / usage 不解释原理(链接 concepts / design)
- concepts / design 不含操作步骤(链接 usage)
- reference 零叙事,只有事实(怎么用链接 usage)

```
getting-started(装起来)→ usage(逐个功能用好)→ reference(查字段)
                                              ↖ concepts / design(理解为什么)
```

## 通用约束(适用于所有类型)

1. **双语同步**:英文为主文档(`xxx.md`),中文为 `xxx.zh.md`,两者章节结构逐节对齐;
   修改一边必须同步另一边。
2. **命令必须带预期输出**:任何可执行命令后紧跟预期输出代码块或截图,
   让读者随时判断自己是否偏航。
3. **验证输出必须真实**:验证章节的输出必须来自真实环境采集,不允许凭记忆编写。
   写作时先确认是否有可用环境:
   - 有 → 在环境中逐条执行命令,采集真实输出填入;
   - 无 → 留占位 `<!-- TODO: 待真实环境采集输出 -->`,由人后续填充。
4. **单一事实源**:同一信息只在一处维护(安装命令只在 getting-started,
   架构图只在 concepts,字段定义以代码为准),其余位置用链接。
5. **YAML 示例完整可用**:可直接 `kubectl apply`,不用 `...` 省略关键字段。
6. **术语统一**:领域术语在 concepts 中定义一次,全部文档复用同一中英文译名。
7. **多 case 同篇**:同一功能/环境的多种做法保留在同一篇内分节,
   开头给出选择依据,不拆分为多个文档。
