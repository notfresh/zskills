# my Hermes Skills

个人收藏的 Hermes Agent 自用 skills。源材料仓库，每个 skill 收录在这，
实际生效版本在 `~/.hermes/skills/`（Hermes 加载只看那个目录）。

## 结构

```
my-hermes-skills/
├── README.md                  ← 本文件
├── skills/
│   └── my-hermes-skills/       ← Android 项目编译环境搭建
│       └── SKILL.md
└── docs/
    └── android-build.md       ← skill 引用的详细实战记录
```

## 当前收录

| Skill | 触发场景 |
|---|---|
| `my-hermes-skills` | 编译 Android 项目从零搭环境，或遇到 AGP / SDK / 代理报错 |
| `se-first-code-change` | 改代码起笔前自审：单点真相 / 命名单元 / 面积最小，存量改动默认禁止、改之前先打报告 |

## 使用方式

### 加载 skill（生效）
skill 加载器只看 `~/.hermes/skills/<name>/SKILL.md`，所以备份后还要同步过去：

```bash
cp -r skills/my-hermes-skills ~/.hermes/skills/
```

或者用 Hermes 的 `skill_manage` 工具。

### 添加新 skill
1. 在 `skills/<name>/SKILL.md` 写主体（**保持简短**，description 60 字符硬上限）
3. 详细安排到 `docs/<name>.md`，skill 里只放链接/路径
4. `git add -A && git commit -m "..."`

## 维护约定

- **不在这条复制三里塞无关代码** —— 这是 skill 备份仓，不是项目源码仓
- **description 硬上限 60 字符**，body 精简 —— 上下文宝贵
- **每个 skill 配一个详细文档** —— 详细骨架留项目自带文档，按需查

## 备份方式
仓库是分两块存的：
1. 这个目录（git 仓库本地）—— 永远在
2. ~/.hermes/skills/（Hermes 实际加载的目录）—— 改动后从这里同步回来
3. 远端（暂未配）—— 后续 push 到 github 备份