# my Hermes Skills

个人收藏的 Hermes Agent 自用 skills。源材料仓库，每个 skill 收录在这，
实际生效版本在 `~/.hermes/skills/`（Hermes 加载只看那个目录）。

## 结构

每个 skill 一个独立文件夹（自包含）：拷给谁用，谁整个文件夹拷走就行。

```
my-hermes-skills/
├── README.md                          ← 本文件
├── android-build-env/                   ← Android 项目编译环境搭建 skill
│   ├── SKILL.md
│   └── android-build.md
├── hermes-issues/                     ← hermes-agent issue/PR 本地库查询 skill
├── hermes-memory/                     ← 操作 hermes-agent-memories 仓（AX Graph 外部 base）
└── se-first-code-change/              ← 改代码起笔前自审 skill
    ├── SKILL.md
    └── literal-interpretation-verbs.md
```

每个 skill 一个文件夹，直接放仓根，不嵌套 `skills/`。拷给谁用整个文件夹拷走。

## 当前收录

| 文件夹 | 触发场景 |
|---|---|
| `android-build-env/` | 编译 Android 项目从零搭环境，或遇到 AGP / SDK / 代理报错 |
| `se-first-code-change/` | 改代码起笔前自审：单点真相 / 命名单元 / 面积最小，存量改动默认禁止、改之前先打报告 |
| `hermes-issues/` | 查 NousResearch/hermes-agent 本地 issue/PR 库（5 套查询方法 + 7 天热点判据） |
| `hermes-memory/` | 操作 hermes-agent-memories 仓：写/查「第 N 条记忆」，`number` 字段就是本仓唯一扩充 |

## 使用方式

### 加载 skill（生效）
skill 加载器只看 `~/.hermes/skills/<name>/SKILL.md`，所以备份后还要同步过去：

```bash
cp -r android-build-env ~/.hermes/skills/
```

### 添加新 skill
1. 在仓根新建 `<skill-name>/` 文件夹（**不**嵌套 `skills/`）
2. 文件夹里放 `SKILL.md`（主体）+ 任何辅助材料（详情/脚本/参考）。**整个文件夹自包含**
3. `git add -A && git commit -m "..."`

## 维护约定

- **每个 skill = 一个文件夹**，所有材料（SKILL.md + 任何辅助文档）都放一起，拷走即用
- **文件夹直接放仓根**，不嵌套 `skills/` 之类的中间目录
- **description 硬上限 60 字符**，body 精简 —— 上下文宝贵
- **不要在这条仓里塞无关代码** —— 这是 skill 备份仓，不是项目源码仓

## 备份方式
仓库是分两块存的：
1. 这个目录（git 仓库本地）—— 永远在
2. ~/.hermes/skills/（Hermes 实际加载的目录）—— 改动后从这里同步回来
3. 远端（暂未配）—— 后续 push 到 github 备份