---
order: 1
date: 2026-04-11
---

# 简介

## 理解

OpenClaw是一款在2026年初爆火的开源AlAgent框架，由奥地利开发者PeterSteinberger创建，它最大的创新在于将大语言模型从"只会聊天"的工具转变为"真正会做事"的管家，其能够直接操作计算机系统，执行跨平台跨应用的复杂任务，像人类一样操作电脑，管理文件、发送消息、浏览网页等。

## 核心概念

- Gateway
- Agent
- Skills
- Memory

## 使用场景

- `办公自动化`：例如：文件管理、邮件处理、日程管理、天气/股票生成日报。
- `浏览器自动化`：数据抓取、表单填写、内容发布。
- `开发与运维`：代码自动生成、Docker自动化运维、跨平台转发消息。

## 安装

```shell
# 1.安装必要依赖
sudo apt update
sudo apt fnstall build-essential -y
sudo apt install cmake curl vim -y

# 2.安装Node.js
https://nodejs.cn/en/download

# 3.安装Git
sudo apt install git -y

# 4.安装OpenClaw
npm install -g openclaw@latest

# 5. 初始化
openclaw onboard --install-daemon
# 交互式配置全跳过，后续配置文件修改
# 其中enable hooks可以全部勾选

# 6.打开仪表盘
openclaw dashboard
```

## 配置

`~/.openclaw/openclaw.json`

```json
{
  "models": {
     "mode": "merge",
     "providers": {
        ""
     }
  }
  "agents": {
    "defaults": {
      "workspace": "/home/molly/.openclaw/workspace"
    }
  },
  "gateway": {
    "mode": "local",
    "auth": {
      "mode": "token",
      "token": "a7a00080b8507432ea45b21c27b4926a086472dc48c97e25"
    },
    "port": 18789,
    "bind": "loopback",
    "tailscale": {
      "mode": "off",
      "resetOnExit": false
    },
    "controlUi": {
      "allowInsecureAuth": true
    },
    "nodes": {
      "denyCommands": [
        "camera.snap",
        "camera.clip",
        "screen.record",
        "contacts.add",
        "calendar.add",
        "reminders.add",
        "sms.send",
        "sms.search"
      ]
    }
  },
  "session": {
    "dmScope": "per-channel-peer"
  },
  "tools": {
    "profile": "coding"
  },
  "wizard": {
    "lastRunAt": "2026-04-10T16:27:51.520Z",
    "lastRunVersion": "2026.4.9",
    "lastRunCommand": "onboard",
    "lastRunMode": "local"
  },
  "meta": {
    "lastTouchedVersion": "2026.4.9",
    "lastTouchedAt": "2026-04-10T16:27:51.617Z"
  },
  "hooks": {
    "internal": {
      "enabled": true,
      "entries": {
        "boot-md": {
          "enabled": true
        },
        "bootstrap-extra-files": {
          "enabled": true
        },
        "command-logger": {
          "enabled": true
        },
        "session-memory": {
          "enabled": true
        }
      }
    }
  }
}
```