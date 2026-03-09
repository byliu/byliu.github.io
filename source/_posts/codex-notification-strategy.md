---
title: 给 Codex 加上系统通知：从沙箱限制到默认授权
date: 2026-03-09 17:05:00
categories:
- AI
- Agent
tags:
- Codex
- Agent
- macOS
- 自动化
- 权限
---

## 背景

我平时把 Codex 跑在 iTerm2 里。一个很实际的问题是：当它跑完任务，或者执行过程中需要我介入时，我不一定一直盯着终端窗口。

需求很简单：

* 任务完成时，给我一个系统级提醒
* 需要我授权、补信息、做决策时，也给我一个系统级提醒
* 最好像微信新消息一样，直接出现在 macOS 通知中心

<!--more-->

## 第一层方案：直接发 macOS 通知

最自然的做法是直接调用 `osascript`：

```sh
osascript -e 'display notification "任务完成" with title "Codex" subtitle "任务完成" sound name "Glass"'
```

这样做在本机上是可行的，但很快遇到了第一个问题：**Codex 的沙箱环境里连不上 macOS Notification Center**。

实际表现是 `osascript` 能执行，但会报 Notification Center 连接无效之类的错误。也就是说，通知能力不是没有，而是**在沙箱里被系统边界挡住了**。

## 第二层方案：改成提权发送

既然沙箱发不了，下一步自然是改成提权发送：

* 正常对话里继续给出文字说明
* 到“任务完成”或“需要你协助”的节点时，额外执行一次提权通知

这解决了沙箱问题，但又出现了第二个矛盾：**如果每次发送通知都先弹一次授权确认，那提醒本身就不再“即时”了。**

换句话说，通知机制本来是为了减少我盯终端的成本，结果如果它自己还要先等我确认，就失去了意义。

## 第三层方案：把约定做成全局规则

另一个现实问题是“新会话不知道这件事”。

当前会话里说好了“完成或阻塞时要提醒我”，并不代表下一个窗口、下一次新开的 Codex 会话也知道这个约定。要想跨会话生效，必须把这条规则写进**会话外的持久位置**。

最终我把规则放到了：

```text
~/.codex/AGENTS.md
```

核心约定是：

* 每个会话都用中文简体回复
* 任务完成或需要用户介入时，除了对话内说明，还要发送 macOS 系统通知

这一步解决的是“**新会话知不知道要通知**”的问题。

## 第四层方案：不要放开任意 `osascript`，只放开专用通知脚本

真正稳定的方案不是“默认允许所有 `osascript -e`”，而是把通知能力收敛到一个固定脚本里。

原因很直接：

* 放开任意 `osascript -e`，授权范围太宽
* 我真正需要默认允许的，其实只有“发一条系统通知”这一件事

所以最终做法是：

1. 写一个专用通知脚本
2. 把这个脚本加入 Codex 的全局批准前缀规则
3. 让全局 `AGENTS.md` 统一要求后续会话调用这个脚本

专用脚本如下：

```sh
#!/bin/zsh
set -euo pipefail

message="${1:-Codex 通知}"
subtitle="${2:-任务完成}"
title="${3:-Codex}"

/usr/bin/osascript - "$message" "$subtitle" "$title" <<'APPLESCRIPT'
on run argv
  set msgText to item 1 of argv
  set subtitleText to item 2 of argv
  set titleText to item 3 of argv
  display notification msgText with title titleText subtitle subtitleText sound name "Glass"
end run
APPLESCRIPT
```

保存路径：

```text
~/.codex/bin/codex_notify.sh
```

这样做的好处是：**授权对象从“任意 AppleScript”收敛成了“一个固定用途的脚本”。**

## 第五层方案：为这个脚本加全局批准前缀

有了专用脚本之后，再把它写进 Codex 的全局规则：

```text
~/.codex/rules/default.rules
```

增加一条：

```text
prefix_rule(pattern=["/Users/baoyiliu/.codex/bin/codex_notify.sh"], decision="allow")
```

这条规则的含义是：

* 后续会话如果要执行这个脚本
* 即使它是提权执行
* 也不需要再为“发送通知”这件事单独弹确认

到这里，问题就被拆解干净了：

* `~/.codex/AGENTS.md` 负责让新会话知道“什么时候该通知”
* `~/.codex/bin/codex_notify.sh` 负责把能力收敛到一个固定动作
* `~/.codex/rules/default.rules` 负责让这个固定动作默认获准

## 最终策略

我最后采用的是下面这套组合：

### 1. 对话内说明仍然保留

通知只是补充，不替代正常的对话说明。这样即使系统通知被专注模式拦掉，终端里仍然有完整上下文。

### 2. 只在两个关键节点发通知

* 任务完成
* 需要我介入

这能避免无意义的通知噪音。

### 3. 默认走专用脚本

统一调用：

```sh
/Users/baoyiliu/.codex/bin/codex_notify.sh "消息内容" "任务完成"
```

或者：

```sh
/Users/baoyiliu/.codex/bin/codex_notify.sh "请确认是否允许执行某操作" "需要你协助"
```

### 4. 让规则全局生效，而不是只对当前会话生效

这样新开窗口、新会话时，不需要重新解释一遍“完成时提醒我”。

## 这套方案的边界

它已经足够实用，但也有边界：

* 如果一个旧会话在规则写入前就已经启动，它不会自动刷新自己读过的指令
* 如果 macOS 开了专注模式，通知可能不会以横幅形式出现
* 如果未来 Codex 的权限模型变化，前缀规则可能需要调整
* 这套方案解决的是“通知授权”和“跨会话约定”，不是任意命令的默认提权

## 总结

这件事里最关键的认知转变是：

**不要追求“默认允许所有提权通知命令”，而是把能力收敛成一个专用脚本，再只对白名单脚本做默认授权。**

这样既保住了体验，也控制住了权限边界。

如果你也把 Agent 长时间跑在终端里，又不想一直盯着窗口，这套方案基本可以直接复用：

* 用全局 `AGENTS.md` 约定什么时候提醒
* 用专用脚本承载通知动作
* 用批准前缀规则给这个动作做默认授权

最后得到的效果就是：**Agent 在该找你的时候，真的能像一条新消息一样把你叫回来。**
