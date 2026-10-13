---
name: function-explain
description: 给 Python 源码某个方法生成"输入/输出/参数说明"文档。用户显式喊才加载。
version: 0.2
---

# function-explain

触发:用户说「给 X 加注释」「解释 X 里 xx 方法」「function-explain X」。description 无强信号词,必须显式喊才加载。

## 默认值

| 选项 | 默认 | override |
| --- | --- | --- |
| 输出形式 | 独立文档 `<file>.func-explain.md`(不动原文件) | "嵌入源码" |
| 范围 | 单个方法 | "全部方法" |
| 已有 docstring | 合并补充(缺的四段靠后追加) | "强制覆盖"/"跳过" |

回车 = 默认。不列 5 个选项让用户拍。

## 流程

1. **问 4 件事**:文件路径(原话有就跳过) / 方法名(`search_files pattern='^def '` 列方法让用户选) / 输出形式 / 已有 docstring 处理。
2. **抽方法源码**:定位 `def` 起止行(到下个 `def`/`class`/缩进回退/文件底部),扫已有 docstring 四段标题(Args/Returns/Side Effects/Note)。
3. **分析**:沿方法体调用追溯定义;标注入参 / 返回值 / 副作用 / 不变量。
4. **生成四段内容**(顺序固定):
   ```
   ### Args / Returns / Side Effects / Note
   ```
5. **写文档**:文件名 `<file>.func-explain.md`,与原文件同目录。
6. **合并补充**:已有段原文照搬(不改字),缺的段靠后追加并标「本次新增」。

## 产物模板

```markdown
# <文件名>

## [<method_name>](./<basename>#L<def_line>-L<end_line>)

### Args
<Args 段;无则写 `(无)`>

### Returns
<Returns 段;无则写 `(无)`>

### Side Effects
<Side Effects 段;无则写 `(无)`>

### Note
<Note 段;无则写 `(无)`>
```

## 边界

- 不改原文件(默认是独立文档)。
- **不**贴方法体 / def 行 / docstring 全文进产物(只四段说明 + 跳转链接)。
- **不**写顶部导航 / 方法索引 / "重跑其他方法"段(默认就 1 个方法,无需索引)。
- **不**自动触发。