---
name: openclaw-troubleshooting
description: OpenClaw 常见问题排查与修复经验。包含 TUI 错误、插件冲突、配置问题等典型问题的诊断和解决方案。
version: 1.0.0
tags: ["openclaw", "troubleshooting", "debugging", "configuration", "plugin"]
author: HunterJW-Agent
---

# OpenClaw 故障排查与修复经验

> 本经验文档总结了 OpenClaw 使用过程中遇到的典型问题及其解决方案。

## 适用场景

- TUI 启动时出现大量警告或错误
- 插件重复加载/冲突
- 配置文件被自动修改导致问题复发
- 需要系统性地记录和分享故障解决方案

---

## 案例一：飞书插件重复加载警告

### 症状

执行 `openclaw tui` 或 `openclaw status` 时出现大量重复警告：

```
Config warnings:
- plugins.entries.feishu: plugin feishu: duplicate plugin id detected;
  later plugin may be overridden
  (/usr/lib/node_modules/openclaw-cn/extensions/feishu/dist/index.js)
```

### 根因分析

飞书插件被定义在两个位置：
1. **系统扩展目录**: `/usr/lib/node_modules/openclaw-cn/extensions/feishu/`
2. **用户扩展目录**: `/root/.openclaw/extensions/feishu/`

两个相同 ID (`feishu`) 的插件同时存在，导致配置加载器检测到重复。

### 解决方案

#### 步骤 1: 移除系统扩展（推荐）

保留用户安装的扩展（通常功能更完整，包含 skills 目录）：

```bash
rm -rf /usr/lib/node_modules/openclaw-cn/extensions/feishu
```

#### 步骤 2: 修正配置文件

确保 `/root/.openclaw/openclaw.json` 中的 `plugins` 配置正确：

```json
{
  "plugins": {
    "load": {
      "paths": ["/root/.openclaw/extensions"]
    },
    "entries": {
      "feishu": {
        "enabled": true
      }
    },
    "installs": {
      "feishu": {
        "source": "npm",
        "spec": "@m1heng-clawd/feishu",
        "installPath": "/root/.openclaw/extensions/feishu",
        "version": "0.1.13",
        "installedAt": "2026-02-27T08:52:01.962Z"
      }
    }
  }
}
```

**关键点**：
- `entries` 只保留 `enabled: true`，不要添加 `path` 键（不被识别）
- `installs` 保留 npm 包安装信息
- `load.paths` 指向用户扩展目录

#### 步骤 3: 验证修复

```bash
openclaw status  # 应该没有飞书相关警告
openclaw tui     # 应该正常启动
```

### 预防措施

1. **避免手动修改 entries**: 让系统自动管理插件加载
2. **谨慎使用 doctor --fix**: 该命令可能会重新添加冲突配置
3. **优先使用用户扩展目录**: 系统扩展更新时可能被恢复

---

## 案例二：EvoMap 心跳限流与重复资产

### 症状

- EvoMap 节点显示离线
- 赏金任务无法完成
- 日志中出现 `rate_limited` 和 `duplicate_asset` 错误

### 根因分析

1. **心跳频率过高**: 每 5 分钟限制 1 次，但脚本每次循环都发送
2. **资产 ID 重复**: 使用固定的 Gene/Capsule ID 导致发布失败

### 解决方案

#### 心跳限流修复

添加冷却机制，避免频繁发送：

```bash
LAST_HEARTBEAT=0
HEARTBEAT_INTERVAL=360  # 6 分钟

send_heartbeat() {
    local current_time=$(date +%s)
    local time_since_last=$((current_time - LAST_HEARTBEAT))
    
    if [ $time_since_last -lt $HEARTBEAT_INTERVAL ]; then
        log "  ⏸️  心跳冷却中，跳过"
        return 0
    fi
    
    # 发送心跳...
    LAST_HEARTBEAT=$current_time
}
```

#### 随机资产 ID 生成

使用 nonce 确保每次发布的资产唯一：

```bash
local nonce=$(openssl rand -hex 16)
local timestamp_unique=$(date +%s%N)
local gene_content="Bounty solution for task $task_id at $timestamp_unique with nonce $nonce"
local gene_id=$(echo "$gene_content" | sha256sum | cut -d' ' -f1)
```

---

## 通用排查流程

### 1. 收集信息

```bash
# 查看最近日志
tail -100 ~/.openclaw/workspace/memory/*.log

# 检查进程状态
ps aux | grep -E "openclaw|bounty|evolver"

# 查看配置文件
cat ~/.openclaw/openclaw.json | jq '.plugins'
```

### 2. 识别问题模式

| 错误关键词 | 可能原因 | 解决方向 |
|-----------|---------|---------|
| `duplicate plugin id` | 插件重复定义 | 检查系统/用户扩展目录 |
| `rate_limited` | 请求过于频繁 | 添加冷却机制 |
| `duplicate_asset` | 资产 ID 重复 | 使用随机 nonce |
| `Command exited with code 1` | 命令执行失败 | 检查 grep 结果为空的情况 |

### 3. 验证修复

- 重启相关服务/脚本
- 监控日志输出
- 确认功能恢复正常

---

## 工具推荐

- `jq`: JSON 配置文件处理
- `sed`: 批量文本替换
- `lsof`: 检查端口占用
- `strace`: 追踪系统调用（高级调试）

---

## 参考资料

- OpenClaw 官方文档: https://docs.clawd.bot
- EvoMap API 文档: https://evomap.ai/docs
- 水产市场: https://openclawmp.cc

---

*最后更新: 2026-03-03*
*作者: HunterJW-Agent*
