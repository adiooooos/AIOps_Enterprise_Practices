# Mattermost 配置

> **状态**：20260609 Updated  
> **系列**：AIOps DEMO Center 部署与配置分步指南  

## 本章概要

这一节主要概述Mattermost在UI上的配置，如何配置admin访问令牌以集成到Agentic AI，如何配置incoming-webhook和outgoing-webhook，如何配置AIOps Chat bot，以及如何将不同场景AIOps的chatops集成到不同的聊天频道。

| # | 主题 | 说明 |
| --- | --- | --- |
| ① | **Admin 访问令牌** | 集成 Agentic AI |
| ② | **基本配置** | Mattermost UI 基础设置 |
| ③ | **Incoming Webhook** | 接收外部消息 |
| ④ | **Outgoing Webhook** | 向外发送事件 |
| ⑤ | **N8N_AIOps_Bot** | 创建并邀请加入频道 |
| ⑥ | **错误问题处理** | 常见问题排查 |

---

## 1. Admin访问令牌设置

<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/6096e94c-0e23-4b97-a0fd-9b7287d240be" />
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/35aab9a7-30d9-4950-9afc-46b5c2592349" />
**Access Token: dykataxxxxxx878id7f318c**

## 2. 基本配置
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/7654d324-d272-4f42-9d02-0f1bf9b6881e" />



## 3. 配置incoming webhook
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/ae9490a5-54c7-4391-a88a-52240ae7cc28" />
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/7c509322-20f3-4941-bed8-0cec8450180e" />
此incoming webhook的用途为：n8n workflow A，将初次诊断的结果post到聊天工具Mattermost里：
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/4a1a6126-11fe-424c-baa9-49f7d43e0391" />


## 4. 配置Outgoing webhook
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/b5ba7834-6c1c-4321-8f3f-a72ac76ddbc9" />
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/809011cd-8732-4ba5-bca6-d7645b429e65" />
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/1d34f460-fe31-4a44-bd82-6c6fc4de72a0" />



## 5. 创建N8N_AIOps_Bot
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/97eb3e65-35d6-4475-b06e-07f483d909a3" />
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/e25688c1-733a-4be6-ac93-b8bd9c01621a" />
<img width="831" height="214" alt="image" src="https://github.com/user-attachments/assets/bc4b0e65-3241-46ca-87bc-bdda189ee176" />
千万记得保存 access token,此access token，aiops Workflow B里需要使用到
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/3b2e2556-e852-4697-a8f4-2aae2cd53a05" />


## 6. 邀请N8N_AIOps_Bot加入Town Square
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/84a7a4ee-75ee-4d55-a626-90e412e1a916" />


## 7. 邀请N8N_AIOps_Bot加入新channel
Same as above


## 8. 错误问题处理
进入server log,根据具体错误描述，进行错误根因分析
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/905635db-1fe1-4b1d-bd1d-f19a0e598580" />

