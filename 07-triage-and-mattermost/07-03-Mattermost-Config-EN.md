# Mattermost Configuration

> **Status**: 20260609 Updated  
> **Series**: AIOps DEMO Center Deploy & Setup Step-by-Step Guide  

## Chapter Overview

This section covers Mattermost UI configuration: how to configure the admin access token for Agentic AI integration, how to configure incoming and outgoing webhooks, how to configure the AIOps Chat bot, and how to integrate AIOps ChatOps for different scenarios into different chat channels.

| # | Topic | Description |
| --- | --- | --- |
| ① | **Admin access token** | Integrate with Agentic AI |
| ② | **Basic configuration** | Mattermost UI baseline settings |
| ③ | **Incoming Webhook** | Receive external messages |
| ④ | **Outgoing Webhook** | Send events outward |
| ⑤ | **N8N_AIOps_Bot** | Create and invite to channels |
| ⑥ | **Troubleshooting** | Common issue resolution |

---

## 1. Admin Access Token Setup

<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/16962767-c667-498f-94ca-d9e950c7f338" />
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/639f97a8-df21-4e35-b4f2-db07dcfd65e0" />


## 2. Basic Configuration

<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/675ba046-8c66-4d87-ad79-2acec1d910c5" />


## 3. Configure Incoming Webhook
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/4a9b2382-f1b1-44e2-b8b6-1f614fde565d" />
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/b0bb63a0-d522-4005-9fc6-729c5e61a861" />
The purpose of this incoming webhook is: n8n workflow A posts the initial diagnosis results to the Mattermost chat tool.
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/7c7006f2-7ea2-4ba7-be6b-7d5a9953bdc8" />


## 4. Configure Outgoing Webhook
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/0988cb77-a877-40ba-9431-f96557d63d0c" />
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/725ba6e3-ef32-4f46-bea2-1ed3f42bbf9b" />
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/26bb10b3-f0af-47d1-9378-8182bd74ae93" />



## 5. Create N8N_AIOps_Bot
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/70c38632-ba60-4b4f-b11e-8a5b1fedf059" />
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/598f955f-af8f-4464-9134-ed2edd3ad8fe" />
<img width="831" height="214" alt="image" src="https://github.com/user-attachments/assets/1e2e15a5-8971-4051-a348-2ce2b0054d5f" />
Make sure to save the access token.This access token is needed for use in the aiops Workflow B
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/8713e594-b297-46e3-b0f3-0fb9901424ba" />


## 6. Invite N8N_AIOps_Bot to Town Square
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/3bef78e2-fa1d-4d3a-a879-7016947f4178" />



## 7. Invite N8N_AIOps_Bot to a New Channel
Same as above


## 8. Error Troubleshooting
Access the server log and perform root cause analysis based on the specific error descriptions.
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/775da060-92cc-4095-b4e0-91aa6d05a4df" />

