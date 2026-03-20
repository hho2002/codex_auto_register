# 改进说明文档

## 改进内容

本次改进针对 ChatGPT/Codex 自动注册工具进行了三项关键优化，以提高注册成功率和绕过反机器人检测。

### 1. 跟随 continue_url - 绕过流程状态机校验

**问题**：OpenAI 的注册流程使用状态机管理，每个步骤完成后会返回 `continue_url` 指示下一步。如果不跟随这个 URL，服务端可能认为流程不完整。

**解决方案**：
- 在 `register()` 和 `create_account()` 方法中，解析响应的 `continue_url` 字段
- 自动发起 GET 请求跟随该 URL，完成流程状态机的转换
- 使用正确的 headers 模拟浏览器导航行为

**修改文件**：
- `chatgpt_register.py`: 第 930-976 行（register 方法）
- `chatgpt_register.py`: 第 963-1039 行（create_account 方法）
- `codex/protocol_keygen.py`: 第 900-941 行（step2_register_user 方法）
- `codex/protocol_keygen.py`: 第 999-1095 行（step5_create_account 方法）

**代码示例**：
```python
# 跟随 continue_url
if r.status_code == 200 and isinstance(data, dict):
    continue_url = data.get("continue_url", "")
    if continue_url:
        if continue_url.startswith("/"):
            continue_url = f"{self.AUTH}{continue_url}"
        follow_r = self.session.get(continue_url, headers={
            "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
            "sec-fetch-dest": "document",
            "sec-fetch-mode": "navigate",
            "sec-fetch-site": "same-origin",
            "sec-fetch-user": "?1",
        }, allow_redirects=True)
```

---

### 2. 添加 Sentinel Token - 绕过 PoW 反机器人校验

**问题**：OpenAI 使用 Sentinel SDK 进行反机器人检测，要求客户端提供包含 PoW（Proof of Work）的 token。

**解决方案**：
- 已有的 `SentinelTokenGenerator` 类实现了完整的 PoW 算法（FNV-1a 哈希 + xorshift）
- 在关键 API 请求中强制添加 `openai-sentinel-token` header
- 使用 `build_sentinel_token()` 函数获取实时的 challenge 和 seed

**修改文件**：
- `chatgpt_register.py`: 第 930-976 行（register 方法）
- `chatgpt_register.py`: 第 963-1039 行（create_account 方法）
- `codex/protocol_keygen.py`: 第 900-941 行（step2_register_user 方法）
- `codex/protocol_keygen.py`: 第 999-1095 行（step5_create_account 方法）

**代码示例**：
```python
# 获取 Sentinel Token
sentinel_token = build_sentinel_token(
    self.session,
    self.device_id,
    flow="user_register",
    user_agent=self.ua,
    sec_ch_ua=self.sec_ch_ua,
    impersonate=self.impersonate
)
if sentinel_token:
    headers["openai-sentinel-token"] = sentinel_token
```

**Sentinel Token 结构**：
```json
{
  "p": "gAAAAAB...",  // PoW 结果
  "t": "",            // 遥测数据（可为空）
  "c": "xxx",         // Challenge token
  "id": "device-id",  // 设备 ID
  "flow": "user_register"  // 业务流程类型
}
```

---

### 3. 补全 sec-fetch-* headers - 模拟真实浏览器指纹

**问题**：现代浏览器会自动添加 `sec-fetch-*` 系列 headers，用于 CORS 和安全策略。缺少这些 headers 会被识别为非浏览器请求。

**解决方案**：
- 在所有 API 请求中添加完整的 `sec-fetch-*` headers
- 根据请求类型（导航、CORS、资源加载）使用不同的值
- 添加 `sec-fetch-user: ?1` 表示用户主动发起

**修改文件**：
- `codex/protocol_keygen.py`: 第 148-161 行（COMMON_HEADERS）
- `chatgpt_register.py`: 第 902-919 行（signin 方法）
- `chatgpt_register.py`: 第 921-928 行（authorize 方法）
- `chatgpt_register.py`: 第 930-976 行（register 方法）
- `chatgpt_register.py`: 第 952-961 行（validate_otp 方法）
- `chatgpt_register.py`: 第 963-1039 行（create_account 方法）

**Headers 对照表**：

| Header | 值 | 说明 |
|--------|-----|------|
| `sec-fetch-dest` | `empty` / `document` | 请求目标类型 |
| `sec-fetch-mode` | `cors` / `navigate` | 请求模式 |
| `sec-fetch-site` | `same-origin` / `cross-site` | 请求来源 |
| `sec-fetch-user` | `?1` | 用户主动发起 |

**代码示例**：
```python
# API 请求（CORS）
headers = {
    "sec-fetch-dest": "empty",
    "sec-fetch-mode": "cors",
    "sec-fetch-site": "same-origin",
    "sec-fetch-user": "?1",
}

# 页面导航
headers = {
    "sec-fetch-dest": "document",
    "sec-fetch-mode": "navigate",
    "sec-fetch-site": "same-origin",
    "sec-fetch-user": "?1",
}
```

---

## 测试验证

运行测试脚本验证改进：

```bash
python test_improvements.py
```

**测试结果**：
```
✅ Sentinel Token 生成测试通过
✅ sec-fetch-* headers 测试通过
✅ continue_url 逻辑测试通过
✅ 集成测试完成
```

---

## 使用说明

### 根目录脚本（chatgpt_register.py）

```bash
# 配置 config.json
cp config.example.json config.json
# 编辑 config.json 填入 DuckMail API Key 和代理

# 运行注册
python chatgpt_register.py
```

### Codex 协议脚本（codex/protocol_keygen.py）

```bash
# 配置 codex/config.json
cp codex/config.example.json codex/config.json
# 编辑 codex/config.json

# 运行注册 + OAuth
python codex/protocol_keygen.py
```

---

## 技术细节

### Sentinel Token PoW 算法

1. **FNV-1a 32位哈希**：
   ```python
   h = 2166136261  # offset basis
   for ch in text:
       h ^= ord(ch)
       h = (h * 16777619) & 0xFFFFFFFF
   ```

2. **xorshift 混合**（murmurhash3 finalizer）：
   ```python
   h ^= h >> 16
   h = (h * 2246822507) & 0xFFFFFFFF
   h ^= h >> 13
   h = (h * 3266489909) & 0xFFFFFFFF
   h ^= h >> 16
   ```

3. **PoW 验证**：
   - 暴力搜索 nonce，使得 `hash(seed + base64(config))` 的前缀 ≤ difficulty
   - 典型难度：`"00000"` 需要约 10-50 万次迭代

### 浏览器指纹模拟

完整的浏览器指纹包括：
- User-Agent
- sec-ch-ua（客户端提示）
- sec-ch-ua-mobile
- sec-ch-ua-platform
- sec-fetch-* 系列
- Accept-Language
- Referer
- Origin

---

## 注意事项

1. **代理配置**：建议使用高质量代理，避免 IP 被封
2. **并发控制**：根据代理质量调整并发数（默认 1-3）
3. **延迟设置**：串行模式下每个账号间隔 3-8 秒
4. **Token 保存**：生成的 Token JSON 文件兼容 CLIProxyAPI v6

---

## 改进效果

- ✅ 提高注册成功率（绕过流程状态机检测）
- ✅ 降低被识别为机器人的概率（Sentinel Token + 浏览器指纹）
- ✅ 更稳定的 OAuth 登录流程（完整的 continue_url 跟随）

---

## 相关文件

- `chatgpt_register.py` - 根目录注册脚本
- `codex/protocol_keygen.py` - Codex 协议注册脚本
- `test_improvements.py` - 测试脚本
- `IMPROVEMENTS.md` - 本文档

---

## 参考资料

- [Sentinel SDK 逆向分析](https://sentinel.openai.com/sentinel/20260124ceb8/sdk.js)
- [Fetch Metadata Request Headers](https://developer.mozilla.org/en-US/docs/Glossary/Fetch_metadata_request_header)
- [FNV Hash](http://www.isthe.com/chongo/tech/comp/fnv/)
