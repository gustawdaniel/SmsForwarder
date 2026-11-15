![SmsForwarder](pic/SmsForwarder.png)

# SmsForwarder (English Edition & Brain OS Integration)

> **Original Upstream:** Forked from [pppscn/SmsForwarder](https://github.com/pppscn/SmsForwarder) under the **BSD 2-Clause License**.  
> [中文版说明文档 (Chinese Documentation)](README_zh.md)

[![GitHub release](https://img.shields.io/github/release/pppscn/SmsForwarder.svg)](https://github.com/pppscn/SmsForwarder/releases)
[![License: BSD-2-Clause](https://img.shields.io/badge/License-BSD_2--Clause-orange.svg)](https://opensource.org/licenses/BSD-2-Clause)

---

## 📖 Overview

**SmsForwarder** is an open-source Android background utility designed for secondary devices, home test phones, and corporate SIM pools. It captures **SMS messages**, **incoming phone calls**, and **App push notifications**, and automatically forwards them to target channels based on configurable rules.

Supported destinations include:
- **Custom Webhooks** (HTTP POST / JSON / REST APIs)
- **Telegram Bot**
- **Email (SMTP)**
- **Bark / PushPlus / ServerChan**
- **Feishu / Lark / DingTalk / WeCom Bots**
- **SMS forwarding** to another physical phone number

---

## ⚡ Integration with Brain OS & `sms.gustawdaniel.com`

This fork is configured and optimized for central SMS ingestion into **Brain OS** and the standalone Go edge proxy [`apps/sms-ingest`](https://github.com/gustawdaniel/brain/tree/main/apps/sms-ingest).

### 🚀 Setup Steps for Fleet Devices

1. **Install App on Android:**
   Download the latest APK from [Releases](https://github.com/pppscn/SmsForwarder/releases) or GitHub Actions. Grant required permissions (SMS, Notification Access, Battery Optimization exemption).

2. **Add Sender (Webhook):**
   - Open **Senders** (发送通道) $\to$ Tap **Add (+)** $\to$ Choose **Webhook**.
   - **Sender Name:** `Brain OS SMS Ingest`
   - **Webhook URL:**
     ```
     https://sms.gustawdaniel.com/api/v1/telemetry/sms
     ```
     *(Or with token query param if headers are unsupported: `https://sms.gustawdaniel.com/api/v1/telemetry/sms?token=YOUR_TELEMETRY_TOKEN`)*
   - **HTTP Method:** `POST`
   - **Headers:**
     ```
     Content-Type: application/json
     X-Brain-Token: YOUR_TELEMETRY_TOKEN
     ```
   - **Message Template (JSON payload):**
     ```json
     {
       "from": "[from]",
       "content": "[content]",
       "card_slot": "[card_slot]",
       "device_mark": "[device_mark]",
       "timestamp": [timestamp]
     }
     ```

3. **Set Device Mark (Device Identifier):**
   In App **Settings** $\to$ **Device Mark** (设备备注), configure the short device alias matching the inventory in `doc/infra/phones.md`:
   - `rm` $\to$ Nubia RedMagic 9 Pro (`792 579 069`, Orange Flex `of69`)
   - `p8` $\to$ Google Pixel 8 (`504 104 003`, Play `pl03`)
   - `p6` $\to$ Google Pixel 6 (`732 739 456`, Play `pl56`)
   - `moto` $\to$ Motorola Moto E40 (`786 594 593`, Orange `or93`)

4. **Add Forwarding Rule:**
   - Open **Forwarding Rules** (转发规则) $\to$ Tap **Add (+)**.
   - Choose Card Slot (ALL SIMs) and Message Type (SMS).
   - Select the `Brain OS SMS Ingest` sender created above.
   - Tap **Save**.

5. **Test:**
   - Tap **Test** on the Webhook sender.
   - Verify arrival in the central web panel at `http://vision/sms` or `http://brain/sms`.

---

## 🔄 Supported Template Variables

When configuring Webhooks or HTTP notification bodies, the following placeholders can be used:

| Variable | Description | Example Replacement |
| :--- | :--- | :--- |
| `[from]` | Sender phone number or contact name | `+48123456789` or `mBank` |
| `[content]` / `[msg]` | Body of the SMS message | `Your OTP code is 948201` |
| `[card_slot]` | SIM card slot name or number | `SIM1`, `SIM2` |
| `[device_mark]` | User-configured device name / alias | `rm`, `p8`, `p6`, `moto` |
| `[timestamp]` | Millisecond epoch timestamp | `1731671234567` |
| `[receive_time]` | Formatted local date and time | `2026-11-15 10:45:00` |
| `[app_version]` | SmsForwarder application version | `v3.3.0` |

---

## 🛠️ Architecture Workflow

```
               ┌────────────────────────────────────────────────────────┐
               │    Android Fleet Phones (rm, p8, p6, moto)             │
               │    SmsForwarder (SMS_RECEIVED Broadcast Receiver)      │
               └──────────────────────────┬─────────────────────────────┘
                                          │  HTTPS POST
                                          │  https://sms.gustawdaniel.com
                                          ▼
               ┌────────────────────────────────────────────────────────┐
               │  📱 Public Ingress: sms.gustawdaniel.com               │
               │  Cloudflare WAF -> Caddy -> sms-ingest (:8779)         │
               │  - Token verification (X-Brain-Token / ?token=)        │
               │  - Store-and-Forward SQLite buffer if backend is down  │
               │  - Immediate 200 OK (no battery drain on phones)       │
               └──────────────────────────┬─────────────────────────────┘
                                          │  Tailscale Mesh (Private)
                                          ▼
               ┌────────────────────────────────────────────────────────┐
               │  🧠 Private Backend (vision: Oracle ARM Ampere)        │
               │  GTD Engine (:8777) & PostgreSQL 16 (brain_knowledge)  │
               │  - Table: sms_inbox (sender, content, alias, number)   │
               │  - Web UI: http://brain/sms (Search, OTP 1-click copy) │
               └────────────────────────────────────────────────────────┘
```

---

## 🛡️ License & Privacy

- **License:** BSD 2-Clause License (see [LICENSE](LICENSE)).
- **Privacy:** SmsForwarder collects no personal data. All SMS data is transmitted exclusively to the self-hosted webhook endpoints defined by the device owner.
- **Upstream Credit:** Huge thanks to `pppscn` and the open-source contributors of [pppscn/SmsForwarder](https://github.com/pppscn/SmsForwarder).
