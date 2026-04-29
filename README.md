# 基于 One API + LobeChat 搭建私有化 AI 生产力工作站

> **运维笔记**：本指南详细介绍了如何在 Ubuntu 24.04 环境下，通过 Docker 快速构建一套集“多模型中转管理”与“高级智能体对话”于一体的私有化 AI 平台。

---

## 1. 项目架构

本方案采用前后端分离的架构，确保了 API 密钥的安全性和模型调度的灵活性：

* **后端 (One API)**：作为“模型中央处理器”。它将不同厂商（OpenAI, Claude, Gemini, DeepSeek 等）的非标 API 统一转化为 OpenAI 标准协议格式，并负责额度监控、渠道负载均衡及令牌分发。
* **前端 (LobeChat)**：作为“高级交互界面”。提供目前开源界最出色的 UI 体验，支持插件系统（联网、绘图）、智能体 (Agent) 市场及多模态交互。

---

## 2. 核心优势

* **统一协议**：只需一套代码即可调用全球主流大模型。
* **运维友好**：通过 One API 日志系统实时监控请求状态（如 429 限流、401 失效等）。
* **插件扩展**：支持联网搜索、代码运行、数据分析等高级功能。
* **数据隐私**：所有对话记录及 API 配置均私有化存储，不经过第三方中转。

---

## 3. 部署步骤

### 3.1 部署后端 One API

```bash
# 创建数据目录
mkdir -p ~/one-api/data

# 启动容器
docker run -d \
  --name one-api \
  -p 3000:3000 \
  -v ~/one-api/data:/data \
  --restart always \
  justsong/one-api
```

**配置要点：**
1. 访问 `http://服务器公网IP:3000`（默认账号：`root`，密码：`123456`）。
2. 在 **[渠道]** 页面添加你的模型提供商（如 OpenAI, Anthropic, DeepSeek 等）。
3. 在 **[令牌]** 页面创建一个新令牌，获取以 `sk-` 开头的密钥。

### 3.2 部署前端 LobeChat

```bash
# 停止并清理旧容器 (如有)
docker rm -f lobe-chat

# 启动 LobeChat
# 注意：172.26.6.133 为当前服务器私网 IP，请根据实际情况替换
docker run -d \
  --name lobe-chat \
  -p 8080:3210 \
  -e OPENAI_API_KEY=你的OneAPI令牌(sk-xxxx) \
  -e OPENAI_PROXY_URL=[http://172.26.6.133:3000/v1](http://172.26.6.133:3000/v1) \
  --restart always \
  lobehub/lobe-chat
```

---

## 4. 前后端对接校验

1.  登录 LobeChat：`http://服务器IP:8080`。
2.  进入 **[设置] -> [语言模型] -> [OpenAI]**。
3.  **API Key**：填入 One API 生成的令牌。
4.  **接口代理地址 (Proxy URL)**：填入 `http://172.26.6.133:3000/v1`。
5.  点击 **[检查连通性]**。显示“检查通过”即表示链路完整。

---

## 5. 运维手册与 FAQ

### 5.1 常用诊断命令
* **查看实时日志**：`docker logs -f lobe-chat` 或 `docker logs -f one-api`
* **容器互通性测试**：`curl -I http://172.26.6.133:3000/v1/models`
* **重启服务**：`docker restart lobe-chat one-api`

### 5.2 故障排查
* **Failed to fetch / Connection Reset**：通常是网络监听地址问题。确保 One API 没有限制仅 `127.0.0.1` 访问，且防火墙已放行对应端口。
* **请求失败 (404/500)**：检查 `OPENAI_PROXY_URL` 是否漏写了 `/v1`。
* **模型不可用**：在 One API 的 **[日志]** 页面查看报错详情。
    * `429`：厂家限流。
    * `无可用渠道`：模型名称不匹配或渠道未开启。

### 5.3 运维场景推荐助手
在 LobeChat 助手市场中，运维人员可以优先关注以下类型的 Agent：
* **Linux 运维专家**：辅助编写 Shell/Python 自动化脚本。
* **K8s 配置诊断**：快速校验 YAML 逻辑与 Pod 状态排查。
* **SQL/正则助手**：处理复杂的日志提取与数据库性能优化。

---

## 6. 安全建议

* **访问授权**：在 LobeChat 启动命令中增加 `-e ACCESS_CODE=你的密码`，为前端添加访问控制。
* **HTTPS 建议**：生产环境下建议在宿主机使用 Nginx 或 Caddy 进行反向代理，并配置 SSL 证书。


