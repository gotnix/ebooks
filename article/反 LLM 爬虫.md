# 反爬虫工具技术指南：go-away、Nepenthes 与指纹识别实践

> 本文档整理了关于反爬虫工具选型、指纹识别技术、以及生产环境配置的最佳实践。适用于需要保护 Web 服务和 API 的技术团队参考。

> [!NOTE]
> 文章是 [千问](https://www.qianwen.com/share/chat/e95d330d761b4167af9079a498e9ec32) 写的，我提问的问题：
>> 信息来源：
>> 1. https://sourcehut.org/blog/2025-04-15-you-cannot-have-our-users-data/
>> 2. https://sourcehut.org/blog/2025-05-29-whats-cooking-q2/
>> 3. https://www.theregister.com/2025/07/09/anubis_fighting_the_llm_hordes/
>> 
>> 我的需求：
>> 1. 根据上述信息 1、2、3 分析一下 SourceHut 的反 LLM 爬虫的发展历程。
>> 2. 介绍信息来源 3 里提及的工具的特性和使用场景：
>>     1. Anubis： [官网](https://anubis.techaro.lol/) ， [GitHub仓库](https://github.com/TecharoHQ/anubis)
>>     2. go-away: [GitHub仓库](https://github.com/WeebDataHoarder/go-away/tree/master)
>>     3. [Nepenthes](https://zadzmo.org/code/nepenthes/)
>>     4. [Quixotic Linkmaze](https://marcusb.org/hacks/quixotic.html)
>> 3. 反 LLM 爬虫工具对比：
>>     1. go-away 是否支持根据请求的指纹来识别爬虫，是如何实现的？
>>     2. Nepenthes 和 Quixotic Linkmaze 支持指纹识别吗，相比 go-away 而言水平如何？
>>     3. 如何决策使用 go-away 还是机器人陷阱作为反爬工具？
>> 4. `ja3n`、`ja4` 是如何识别爬虫的？
>> 5. 在实际业务中，合作方的服务端程序需通过 API 合法调用我方接口；面对 go-away 基于`TLS 指纹（JA3/JA4）`和`行为挑战`的严格检测，我们应如何配置策略，以避免这些合法请求被误判为爬虫并遭到封禁？

---

## 目录

- [一、反爬工具选型决策](#一反爬工具选型决策)
- [二、指纹识别技术原理](#二指纹识别技术原理)
- [三、工具能力对比](#三工具能力对比)
- [四、防止误封合法合作方](#四防止误封合法合作方)
- [五、生产环境配置示例](#五生产环境配置示例)
- [六、总结与建议](#六总结与建议)

---

## 一、反爬工具选型决策

### 1.1 决策对比矩阵

| 决策维度 | **go-away** | **机器人陷阱 (Nepenthes)** |
|---------|------------|--------------------------|
| **防御策略** | 主动挑战/阻止 | 被动消耗资源 |
| **用户体验影响** | 可配置，支持非 JS 挑战，影响较小 | 不影响正常用户（需正确配置） |
| **服务器资源消耗** | 中等（挑战验证） | **高**（持续生成内容 + 延迟） |
| **搜索引擎影响** | 可放行合法爬虫 | **可能导致从搜索结果消失** |
| **配置复杂度** | 中等（CEL 规则语言） | 较低（YAML 配置） |
| **防御效果** | 阻止低效爬虫 | 消耗爬虫时间和资源 |
| **适用场景** | 生产环境、公开网站 | 隔离环境、honeypot |

### 1.2 决策流程

```
┌─────────────────────────────────────────────────────────┐
│                    选择反爬工具决策树                     │
└─────────────────────────────────────────────────────────┘
                          │
          ┌───────────────┴───────────────┐
          ▼                               ▼
   【需要保护正常访问？】           【可以接受搜索引擎消失？】
          │                               │
    ┌─────┴─────┐                   ┌─────┴─────┐
    ▼           ▼                   ▼           ▼
   是          否                  是          否
    │           │                   │           │
    ▼           ▼                   ▼           ▼
┌────────┐  ┌────────┐        ┌────────┐  ┌────────┐
│go-away │  │ 混合   │        │Nepenthes│  │ 其他  │
│        │  │ 部署   │        │         │  │ 方案   │
└────────┘  └────────┘        └────────┘  └────────┘
```

### 1.3 场景推荐

| 场景 | 推荐工具 | 理由 |
|------|---------|------|
| **生产网站/服务** | go-away | 可放行合法爬虫，用户影响可控 |
| **API 服务** | go-away | 支持 PASS 规则，可针对 API 放行 |
| **文档/代码仓库** | go-away | SourceHut 已验证有效，支持静态资源放行 |
| **隔离 honeypot** | Nepenthes | 专门用于诱捕和消耗爬虫资源 |
| **高流量网站** | go-away | Nepenthes 可能导致显著 CPU 负载 |
| **需要搜索引擎收录** | go-away | Nepenthes 会导致从搜索结果消失 |
| **深度防御策略** | 两者结合 | go-ahead 前置过滤 + Nepenthes 作为陷阱 |

### 1.4 SourceHut 实践经验

SourceHut 的演进路径提供了重要参考：
1. **最初部署 Anubis** → 有效但用户影响较大
2. **切换到 go-away** → 更可配置，减少对用户影响
3. **未采用陷阱方案** → 需要保持搜索引擎可访问性

---

## 二、指纹识别技术原理

### 2.1 go-away 指纹识别能力

| 指纹类型 | 支持状态 | 实现方式 |
|---------|---------|---------|
| **TLS 指纹 (JA3N)** | ✅ 支持 | `fp.ja3n` |
| **TLS 指纹 (JA4)** | ✅ 支持 | `fp.ja4` |
| **HTTP 请求指纹** | ✅ 支持 | `headers`, `userAgent` |
| **网络指纹** | ✅ 支持 | `remoteAddress.network()` |
| **行为指纹** | ✅ 支持 | 挑战响应行为 |

### 2.2 JA3/JA4 识别原理

**JA3/JA4 是 TLS 客户端指纹技术**，通过分析 TLS 握手过程中的 **Client Hello** 数据包生成唯一标识。

```
┌─────────────────────────────────────────────────────────────────┐
│                    TLS 握手指纹提取流程                          │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │   Client Hello 包     │
              └───────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
   ┌─────────┐      ┌─────────┐       ┌─────────┐
   │TLS 版本 │      │加密套件  │       │扩展字段  │
   │(Version)│      │Ciphers  │       │Extensions│
   └─────────┘      └─────────┘       └─────────┘
        │                 │                 │
        └─────────────────┼─────────────────┘
                          ▼
              ┌───────────────────────┐
              │   字段拼接 + 哈希计算  │
              └───────────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │   JA3/JA4 指纹哈希值   │
              └───────────────────────┘
```

### 2.3 JA3 与 JA4 对比

| 特性 | **JA3** | **JA4** |
|------|---------|---------|
| **推出时间** | 2017 年 | 2023 年 |
| **哈希算法** | MD5 (32 位) | SHA256 |
| **格式** | 连续哈希字符串 | 模块化格式 (tls13_27d66c_3f8e9a) |
| **参数排序** | 顺序敏感 | 智能排序，解决顺序问题 |
| **协议支持** | 主要 TLS | TLS/HTTP/QUIC 多协议 |
| **可读性** | 低 | 高（分段显示关键信息） |

### 2.4 为什么能识别爬虫

**不同 TLS 库实现有独特指纹特征**：

| 客户端类型 | TLS 库 | 典型指纹特征 |
|-----------|--------|-------------|
| **Chrome 浏览器** | BoringSSL | 特定加密套件顺序 + 扩展列表 |
| **Python requests** | OpenSSL | 默认套件组合，易识别 |
| **Go net/http** | Go TLS | 独特扩展支持 |
| **curl** | OpenSSL/cURL | 可识别的默认配置 |
| **LLM 爬虫** | 各种库 | 往往使用默认配置，指纹暴露 |

### 2.5 go-away 指纹配置示例

```yaml
# go-away 配置示例 - 指纹识别规则
rules:
  - name: block-suspicious-tls
    action: DENY
    conditions:
      - 'fp.ja3n == "known_bad_fingerprint"'
      - 'fp.ja4 == "bot_ja4_hash"'
  
  - name: challenge-suspicious
    action: CHALLENGE
    conditions:
      - 'fp.ja3n != "" && fp.ja3n in ["suspicious_1", "suspicious_2"]'
```

**前提条件**：必须启用 TLS（HTTPS），指纹仅在 TLS 连接时可用。

---

## 三、工具能力对比

### 3.1 指纹识别能力对比

| 工具 | **TLS 指纹识别** | **HTTP 指纹识别** | **行为指纹** | **IP 指纹** |
|------|-----------------|------------------|-------------|------------|
| **go-away** | ✅ 支持 (JA3N/JA4) | ✅ 支持 (Headers/UA) | ✅ 支持 (挑战响应) | ✅ 支持 (网络范围) |
| **Nepenthes** | ❌ 不支持 | ❌ 不支持 | ❌ 不支持 | ❌ 不支持 |
| **Quixotic/Linkmaze** | ❌ 不支持 | ❌ 不支持 | ❌ 不支持 | ❌ 不支持 |

### 3.2 各工具技术特性

#### go-away

| 维度 | 详情 |
|------|------|
| **定位** | 主动防御型反爬虫中间件 |
| **核心机制** | 工作量证明挑战 + 指纹识别 |
| **指纹能力** | TLS(JA3N/JA4)、HTTP 头、IP 网络范围 |
| **规则引擎** | CEL (Common Expression Language) |
| **配置复杂度** | 中等（YAML + CEL 规则） |
| **用户影响** | 可配置，支持非 JS 挑战 |
| **适用场景** | 生产环境、API 服务、公开网站 |

#### Nepenthes

| 维度 | 详情 |
|------|------|
| **定位** | 低交互蜜罐 (Low-Interaction Honeypot) |
| **核心机制** | 诱捕 + 资源消耗 |
| **指纹能力** | **无主动指纹识别** |
| **工作原理** | 生成无尽链接迷宫，困住爬虫蜘蛛 |
| **配置复杂度** | 低（易于安装，维护需求少） |
| **用户影响** | 不影响正常用户（需正确隔离部署） |
| **适用场景** | 隔离 honeypot、诱捕研究、补充防御层 |

#### Quixotic / Linkmaze

| 维度 | 详情 |
|------|------|
| **定位** | 链接迷宫陷阱工具 |
| **核心机制** | 制造链接迷宫，扰乱抓取路径 |
| **指纹能力** | **无主动指纹识别** |
| **工作原理** | 与 Nepenthes 类似，生成迷惑性内容 |
| **配置复杂度** | 低 |
| **用户影响** | 不影响正常用户（需正确配置） |
| **适用场景** | 爬虫陷阱、补充防御层 |

### 3.3 综合能力对比矩阵

| 能力维度 | go-away | Nepenthes | Quixotic/Linkmaze |
|----------| --------|-----------|-------------------|
| **主动指纹识别**     | ⭐⭐⭐⭐⭐ | ❌ | ❌ |
| **TLS 指纹 (JA3/4)** | ✅ | ❌ | ❌ |
| **HTTP 指纹**        | ✅ | ❌ | ❌ |
| **行为挑战**         | ✅ | ❌ | ❌ |
| **规则引擎**         | CEL | 简单 | 简单 |
| **可配置性**         | 高 | 低 | 低 |
| **生产环境适用**     | ✅ | ⚠️ | ⚠️ |
| **搜索引擎友好**     | ✅ | ❌ | ❌ |
| **资源消耗**         | 中等 | 高 | 高 |
| **部署复杂度**       | 中等 | 低 | 低 |

---

## 四、防止误封合法合作方

### 4.1 核心思路：多层豁免机制

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        合法 API 调用豁免策略                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐            │
│   │  IP 白名单   │    │  API 密钥认证 │    │  路径排除     │            │
│   │  (最可靠)    │    │  (推荐)       │    │  (辅助)      │            │
│   └──────────────┘    └──────────────┘    └──────────────┘            │
│         │                   │                   │                      │
│         └───────────────────┼───────────────────┘                      │
│                             ▼                                          │
│                   ┌─────────────────┐                                  │
│                   │   go-away PASS   │                                  │
│                   │     规则放行     │                                  │
│                   └─────────────────┘                                  │
│                             │                                          │
│                             ▼                                          │
│                   ┌─────────────────┐                                  │
│                   │   后端服务       │                                  │
│                   └─────────────────┘                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 方案 1：IP 白名单（最可靠）

**适用场景**：合作方有固定出口 IP

```yaml
# go-away 配置示例
network_ranges:
  partner-ips:
    - "203.0.113.0/24"      # 合作方 IP 段
    - "198.51.100.50"       # 单个 IP

policy:
  conditions:
    - name: is-partner-ip
      expression: 'remoteAddress.network("partner-ips")'
  
  rules:
    - name: allow-partner-traffic
      action: PASS
      conditions:
        - 'is-partner-ip'
        - 'path.startsWith("/api/")'
      
    - name: default-challenge
      action: CHALLENGE
      conditions:
        - 'true'  # 其他所有流量
```

**优点**：
- ✅ 最可靠，不依赖请求头（可伪造）
- ✅ 零性能开销
- ✅ 配置简单

**缺点**：
- ❌ 合作方 IP 变更需更新配置
- ❌ 不适合动态 IP 场景

### 4.3 方案 2：API 密钥认证（推荐）

**适用场景**：合作方无固定 IP，但可携带认证信息

```yaml
# go-away 配置示例
policy:
  conditions:
    - name: has-valid-api-key
      expression: >
        headers.get("X-API-Key") in [
          "partner-key-1-abc123",
          "partner-key-2-def456",
          "partner-key-3-ghi789"
        ]
    
    - name: is-api-path
      expression: 'path.startsWith("/api/v1/")'
  
  rules:
    - name: allow-authenticated-api
      action: PASS
      conditions:
        - 'is-api-path'
        - 'has-valid-api-key'
      
    - name: challenge-others
      action: CHALLENGE
      conditions:
        - 'true'
```

**优点**：
- ✅ 不依赖 IP，适合动态 IP 场景
- ✅ 可随时撤销/轮换密钥
- ✅ 可区分不同合作方

**缺点**：
- ❌ 密钥可能泄露
- ❌ 需要与合作方协调密钥管理

### 4.4 方案 3：路径排除 + 指纹组合

**适用场景**：API 路径明确，可结合多种条件

```yaml
# go-away 配置示例
policy:
  conditions:
    - name: is-api-endpoint
      expression: >
        path.startsWith("/api/") || 
        path.startsWith("/webhook/") ||
        path.startsWith("/partner/")
    
    - name: has-authorization-header
      expression: 'headers.get("Authorization") != ""'
    
    - name: is-known-user-agent
      expression: >
        userAgent.contains("PartnerClient/1.0") ||
        userAgent.contains("IntegrationBot/2.0")
  
  rules:
    - name: allow-api-with-auth
      action: PASS
      conditions:
        - 'is-api-endpoint'
        - 'has-authorization-header'
      
    - name: allow-known-partner-ua
      action: PASS
      conditions:
        - 'is-api-endpoint'
        - 'is-known-user-agent'
      
    - name: challenge-api-without-auth
      action: CHALLENGE
      conditions:
        - 'is-api-endpoint'
        - '!has-authorization-header'
      
    - name: default-challenge
      action: CHALLENGE
      conditions:
        - 'true'
```

### 4.5 方案 4：多层组合策略（最佳实践）

```yaml
# go-away 完整配置示例
network_ranges:
  partner-ips:
    - "203.0.113.0/24"
    - "198.51.100.0/24"
  
  known-bot-networks:
    - url: "https://ip-ranges.amazonaws.com/ip-ranges.json"
      jq-path: '(.prefixes[] | select(has("ip_prefix")) | .ip_prefix)'

policy:
  conditions:
    # === 第一层：IP 白名单（最高优先级）===
    - name: is-partner-ip
      expression: 'remoteAddress.network("partner-ips")'
    
    # === 第二层：API 密钥认证 ===
    - name: has-valid-api-key
      expression: >
        headers.get("X-API-Key") in [
          "sk_partner_abc123",
          "sk_partner_def456"
        ]
    
    # === 第三层：标准认证头 ===
    - name: has-bearer-token
      expression: 'headers.get("Authorization").startsWith("Bearer ")'
    
    # === 第四层：路径识别 ===
    - name: is-api-path
      expression: >
        path.startsWith("/api/") ||
        path.startsWith("/webhook/") ||
        path.startsWith("/v1/")
    
    # === 第五层：合法爬虫放行 ===
    - name: is-legitimate-bot
      expression: >
        (userAgent.contains("Googlebot") && remoteAddress.network("googlebot-cidr")) ||
        (userAgent.contains("Bingbot") && remoteAddress.network("bingbot-cidr"))
  
  rules:
    # 规则顺序很重要，从上到下匹配
    - name: allow-partner-ip
      action: PASS
      conditions:
        - 'is-partner-ip'
      log: true
    
    - name: allow-api-with-key
      action: PASS
      conditions:
        - 'is-api-path'
        - 'has-valid-api-key'
    
    - name: allow-api-with-bearer
      action: PASS
      conditions:
        - 'is-api-path'
        - 'has-bearer-token'
    
    - name: allow-legitimate-bots
      action: PASS
      conditions:
        - 'is-legitimate-bot'
    
    - name: challenge-api-without-auth
      action: CHALLENGE
      conditions:
        - 'is-api-path'
    
    - name: default-challenge
      action: CHALLENGE
      conditions:
        - 'true'
```

### 4.6 实施建议与最佳实践

#### 配置管理建议

| 实践 | 说明 |
|------|------|
| **密钥轮换** | 定期更换 API 密钥，配置支持多密钥并存过渡期 |
| **IP 更新机制** | 使用动态 IP 列表（从 URL 加载），合作方 IP 变更自动更新 |
| **灰度发布** | 新配置先在小流量测试，确认无误后全量发布 |
| **监控告警** | 监控 PASS 规则命中率，异常波动及时告警 |

#### 合作方接入检查清单

- [ ] 确认合作方出口 IP（固定/动态）。
- [ ] 分配专用 API 密钥。
- [ ] 约定请求头规范（X-API-Key / Authorization）。
- [ ] 约定 User-Agent 标识。
- [ ] 提供测试环境验证配置。
- [ ] 建立应急联系渠道（配置问题快速响应）。
- [ ] 文档化接入流程和故障排查指南。

#### 应急处理流程

> [!IMPORTANT]
> 目标：5 分钟内恢复访问，24 小时内完成根因修复。

|编号|事件|处理方式|
|----|----|--------|
|1|合作方报告|接收工单|
|2|确认问题|查看日志确认误封|
|3|临时 IP 放行|添加临时 PASS 规则|
|4|根因分析|分析配置问题原因|
|5|永久修复|更新正式配置|

---

## 五、生产环境配置示例

```yaml
# go-away 生产配置示例
# config.yaml

# 网络范围定义
network_ranges:
  partner-acme:
    - "203.0.113.0/24"
  partner-globex:
    - "198.51.100.0/24"
  googlebot:
    - url: "https://developers.google.com/search/apis/ipranges/googlebot.json"
      jq-path: '.prefixes[].prefix'

# TLS 配置（指纹识别必需）
tls:
  enabled: true
  autocert:
    enabled: true
    cache_dir: "/var/lib/go-away/certs"

# 挑战配置
challenges:
  - name: standard-browser
    action: challenge
    settings:
      challenges: [http-cookie-check, preload-link, js-pow-sha256]
      timeout: 30s

# 策略规则
policy:
  conditions:
    # IP 白名单条件
    - name: is-partner-acme
      expression: 'remoteAddress.network("partner-acme")'
    - name: is-partner-globex
      expression: 'remoteAddress.network("partner-globex")'
    
    # API 认证条件
    - name: has-api-key
      expression: >
        headers.get("X-API-Key") in [
          "sk_acme_prod_abc123",
          "sk_globex_prod_def456"
        ]
    
    # 路径条件
    - name: is-api-path
      expression: >
        path.startsWith("/api/") ||
        path.startsWith("/webhook/")
    
    # 合法爬虫
    - name: is-googlebot
      expression: >
        userAgent.contains("Googlebot") && 
        remoteAddress.network("googlebot")

  rules:
    # === 最高优先级：合作伙伴 IP 直接放行 ===
    - name: allow-partner-acme
      action: PASS
      conditions:
        - 'is-partner-acme'
        - 'is-api-path'
      log: true
    
    - name: allow-partner-globex
      action: PASS
      conditions:
        - 'is-partner-globex'
        - 'is-api-path'
      log: true
    
    # === 第二优先级：API 密钥认证放行 ===
    - name: allow-api-with-key
      action: PASS
      conditions:
        - 'is-api-path'
        - 'has-api-key'
      log: true
    
    # === 第三优先级：合法搜索引擎爬虫 ===
    - name: allow-googlebot
      action: PASS
      conditions:
        - 'is-googlebot'
    
    # === 第四优先级：API 路径无认证，给轻量挑战 ===
    - name: challenge-api-no-auth
      action: CHALLENGE
      conditions:
        - 'is-api-path'
        - '!has-api-key'
      settings:
        challenges: [http-cookie-check]  # 最轻量挑战
    
    # === 默认：其他所有流量标准挑战 ===
    - name: default-challenge
      action: CHALLENGE
      conditions:
        - 'true'
```

---

## 六、总结与建议

### 6.1 方案对比总结

| 方案 | 可靠性 | 灵活性 | 维护成本 | 推荐场景 |
|------|--------|--------|----------|----------|
| **IP 白名单** | ⭐⭐⭐⭐⭐ | ⭐⭐ | 中 | 固定 IP 合作方 |
| **API 密钥** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 低 | 动态 IP/多合作方 |
| **认证头** | ⭐⭐⭐ | ⭐⭐⭐⭐ | 低 | 标准 API 集成 |
| **组合策略** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 中 | 生产环境推荐 |

### 6.2 工具选择总结

| 对比维度 | **go-away** | **Nepenthes/Quixotic** |
|---------|------------|----------------------|
| **指纹识别** | ✅ 支持 JA3N/JA4/TLS/HTTP | ❌ 不支持 |
| **防御策略** | 主动挑战 + 识别 | 被动诱捕 + 消耗 |
| **技术复杂度** | 高（规则引擎 + 指纹） | 低（内容生成） |
| **生产适用性** | 高 | 低（需隔离部署） |
| **搜索引擎友好** | 高 | 低 |
| **SourceHut 选择** | ✅ 最终采用 | ❌ 未采用 |

### 6.3 最佳实践建议

1. **生产环境使用组合策略**：IP 白名单 + API 密钥 + 路径识别
2. **配置版本管理**：所有配置纳入 Git 版本控制
3. **定期审计**：每季度审查 PASS 规则，清理过期密钥和 IP
4. **监控告警**：设置 PASS 规则命中率异常告警
5. **文档化**：维护合作方接入文档和应急处理流程
6. **指纹识别必需 TLS**：确保启用 HTTPS 以使用 JA3N/JA4 指纹
7. **深度防御**：可考虑 go-away 前置过滤 + Nepenthes 陷阱的组合策略

### 6.4 核心结论

- **go-away 是唯一支持指纹识别的工具**，具备 JA3N/JA4 TLS 指纹、HTTP 头指纹、IP 网络范围识别等能力
- **Nepenthes 和 Quixotic/Linkmaze 不支持指纹识别**，它们是被动的陷阱工具，依靠消耗爬虫资源而非主动识别
- **防止误封需要多层豁免机制**，推荐 IP 白名单 + API 密钥 + 路径识别的组合策略
- **SourceHut 选择 go-away 而非陷阱方案**，主要是为了保持搜索引擎可访问性和最小化用户影响

---

## 附录：快速参考

### CEL 表达式常用函数

| 函数 | 说明 | 示例 |
|------|------|------|
| `remoteAddress.network()` | IP 网络范围匹配 | `remoteAddress.network("partner-ips")` |
| `headers.get()` | 获取 HTTP 头 | `headers.get("X-API-Key")` |
| `path.startsWith()` | 路径前缀匹配 | `path.startsWith("/api/")` |
| `userAgent.contains()` | User-Agent 包含 | `userAgent.contains("Googlebot")` |
| `fp.ja3n` | JA3N TLS 指纹 | `fp.ja3n == "hash_value"` |
| `fp.ja4` | JA4 TLS 指纹 | `fp.ja4 == "hash_value"` |

### 规则动作类型

| 动作 | 说明 |
|------|------|
| `PASS` | 直接放行，不进行挑战 |
| `CHALLENGE` | 要求进行挑战验证 |
| `DENY` | 直接拒绝请求 |

---

*文档版本：1.0*  
*最后更新：2024 年*  
*适用工具：go-away v1.0+, Nepenthes, Quixotic/Linkmaze*
