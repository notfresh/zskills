---
name: smart
description: Judge task complexity first; plan-and-approve big work.
version: 0.1.0
author: 郑旭 (notfresh), Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [workflow, complexity, planning]
    related_skills: [SEfirst, plan]
---

# smart —— 先评估难度,再决定直做还是报方案

## When to Use

任何任务动手前都过一遍。默认常驻,不用显式加载。

## 判据

**简单**(直接做,别废话):
- 单文件/单点改动
- 一次调用能结束
- 可秒级回滚
- 无不可逆副作用

**复杂**(先报方案,等批准):命中任意一条
- 改动 ≥2 个文件,或跨模块
- 要跑多次外部调用 / 耗时 >1 分钟
- 不可逆(push、删除、部署、迁移、改数据)
- 影响面不明确或结果不确定

## 简单任务 → 直接执行

做完一句话报结果:改了什么、验证了什么。不列清单,不确认。

## 复杂任务 → 先给方案,等批准

只给三行:

```
方案: <分几步,每步做什么>
成本: <时间 / 调用次数 / 改动文件数 / 是否不可逆>
回滚: <怎么撤>
```

写完停下,等用户点头。用户说"对 / 行 / 动手 / 开始"即视为批准,直接执行,不再追问、不再重复方案。

## Pitfalls

- 判成复杂却闷头跑完 —— 高频翻车(2026-10-29 教训:找 token 闷头 10 分钟被骂)。
- 判成简单却反复追问 —— 同样翻车。
- 方案别写成待确认清单,只写一条路径;代价和回滚点写清楚就够了。
