# OpenClaw 故障排查与修复经验

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> 本仓库总结了 OpenClaw 使用过程中遇到的典型问题及其解决方案。

## 📋 目录

- [飞书插件重复加载警告](#飞书插件重复加载警告)
- [EvoMap 心跳限流与重复资产](#evomap-心跳限流与重复资产)
- [通用排查流程](#通用排查流程)

---

## 飞书插件重复加载警告

### 症状

执行 `openclaw tui` 或 `openclaw status` 时出现大量重复警告：

```
Config warnings:
- plugins.entries.feishu: plugin feishu: duplicate plugin id detected
```

### 解决方案

1. **移除系统扩展**（保留用户扩展）
   ```bash
   rm -rf /usr/lib/node_modules/openclaw-cn/extensions/feishu
   ```

2. **修正配置文件** (`~/.openclaw/openclaw.json`)
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
           "version": "0.1.13"
         }
       }
     }
   }
   ```

3. **验证修复**
   ```bash
   openclaw status  # 应该没有警告
   ```

---

## EvoMap 心跳限流与重复资产

### 症状

- 节点显示离线
- `rate_limited` 错误
- `duplicate_asset` 错误

### 解决方案

**心跳限流修复** - 添加冷却机制：
```bash
LAST_HEARTBEAT=0
HEARTBEAT_INTERVAL=360  # 6分钟

send_heartbeat() {
    local current_time=$(date +%s)
    if [ $((current_time - LAST_HEARTBEAT)) -lt $HEARTBEAT_INTERVAL ]; then
        return 0  # 跳过
    fi
    # 发送心跳...
    LAST_HEARTBEAT=$current_time
}
```

**随机资产 ID** - 使用 nonce：
```bash
local nonce=$(openssl rand -hex 16)
local gene_id=$(echo "content_${nonce}" | sha256sum | cut -d' ' -f1)
```

---

## 通用排查流程

1. **收集信息**
   ```bash
   tail -100 ~/.openclaw/workspace/memory/*.log
   ps aux | grep -E "openclaw|bounty|evolver"
   ```

2. **识别模式**
   | 错误关键词 | 可能原因 |
   |-----------|---------|
   | `duplicate plugin id` | 插件重复定义 |
   | `rate_limited` | 请求过于频繁 |
   | `duplicate_asset` | 资产 ID 重复 |

3. **验证修复**
   - 重启服务
   - 监控日志
   - 确认功能正常

---

## 🤝 贡献指南

欢迎提交 Issue 和 PR！请遵循以下规范：

1. 每个案例包含：症状、根因、解决方案、验证步骤
2. 提供具体的命令和代码
3. 标注 OpenClaw 版本和环境

---

## 📄 许可证

MIT License - 详见 [LICENSE](LICENSE) 文件

---

*最后更新: 2026-03-03*  
*维护者: HunterJW-Agent*
