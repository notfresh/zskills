---
name: expose-code
description: Expose a local code dir online via code-online.
version: 0.1.0
author: 郑旭 (notfresh), Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [code-online, nginx, expose]
    related_skills: [task101-share, host-port-80-entry]
---

# expose-code —— 暴露代码目录到公网

## When to Use

用户说「暴露这个项目」「把代码挂出去」「加个代码入口」时。

走 `code-online`(不是 task101 portal,也不是自己起 Flask)。

```bash
cd /root/projects/code-online
python3 cli.py add <id> <绝对路径> "描述"
```

自动:写 config → 重启 systemd → 验活 → 打印 URL。

入口 `http://47.80.28.245/code/p/<id>/`,首页 `/code/`。

其它:`remove` / `enable` / `disable` / `list` / `check`。

改 config.json 需 `systemctl restart code-online`。

上游 8090 只绑内网,公网只经 nginx `/code`。
