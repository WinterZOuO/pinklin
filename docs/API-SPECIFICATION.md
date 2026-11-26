---
title: Pinklin Phase 1 — API Specification
type: DESIGN
status: draft            # draft → approved
owner: jackylau
prd: docs/project/PRD-project.md
architecture: docs/ARCHITECTURE.md
created: 2026-11-26
updated: 2026-11-26
---

# API 規格

> 本文件是設計階段的 API 契約（REQ-003、REQ-004）。實作後，由 drf-spectacular 產生的 OpenAPI 3 文件
> （`/api/v1/schema`、Swagger UI 在 `/api/v1/docs`）成為欄位層級的唯一依據；兩者不一致時修正其中一方，並在同一個變更中更新另一方。
> 權限代碼見 `SECURITY-DESIGN.md` §3；業務規則見 `BUSINESS-RULES.md`。

## 1. 共通慣例

| 項目 | 規則 |
| --- | --- |
| 基本路徑 | `/api/v1/`；路徑不加結尾斜線 |
| 版本 | 新增欄位或端點屬於相容變更，可直接加入；移除或改變語意需開 `/api/v2/` |
| 格式 | `application/json; charset=utf-8`；頭像上傳用 `multipart/form-data` |
| 驗證 | `Authorization: Bearer <access_token>`；Web 的 refresh token 走 cookie |
| ID | UUID 字串；訂單另有 `order_number` 供人辨識 |
| 時間 | ISO 8601、UTC，例如 `2026-11-26T10:00:00Z` |
| 金額 | 十進位字串加幣別：`{"amount": "30.000000", "currency": "USDT"}`；USDT 固定 6 位小數。前端不得用浮點數運算 |
| 點數 | 整數 |
| 分頁（帳本、紀錄） | cursor：`?limit=20&cursor=…`，回應 `{ "items": [...], "next_cursor": "…" \| null }` |
| 分頁（後台列表） | 頁碼：`?page=1&page_size=50`，回應 `{ "items": [...], "page": 1, "page_size": 50, "total": 1234 }` |
| 冪等 | 建立訂單、付款、後台的寫入操作接受 `Idempotency-Key` header（24 小時內有效） |
| 追蹤 | 每個回應帶 `X-Request-ID`；錯誤回應的 `request_id` 與它相同 |
| 限流 | 超過時回傳 `429` 與 `Retry-After` |

### 1.1 錯誤格式

```json
{
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "USDT 餘額不足",
    "details": { "shortfall": "17.500000" },
    "request_id": "0193a7c2-…"
  }
}
```

- `code` 是給程式判斷的穩定值；`message` 是給人看的說明，可依語系變化。
- `VALIDATION_ERROR` 的 `details` 是欄位對錯誤列表：`{"fields": {"email": ["格式不正確"]}}`。

## 2. 錯誤碼目錄

| 錯誤碼 | HTTP | 說明 |
| --- | --- | --- |
| `VALIDATION_ERROR` | 400 | 請求欄位驗證失敗 |
| `AUTHENTICATION_REQUIRED` | 401 | 未提供 token |
| `TOKEN_EXPIRED` | 401 | access token 過期，請 refresh |
| `TOKEN_INVALID` | 401 | token 無效 |
| `SESSION_REVOKED` | 401 | session 已被撤銷 |
| `REFRESH_TOKEN_REUSED` | 401 | 偵測到舊 refresh token 重放，session 已撤銷 |
| `PERMISSION_DENIED` | 403 | 權限不足 |
| `NOT_FOUND` | 404 | 資源不存在，或不屬於目前使用者 |
| `CONFLICT` | 409 | 一般衝突（例如樂觀鎖失敗，可重試） |
| `REQUEST_IN_PROGRESS` | 409 | 同一個 Idempotency-Key 的請求仍在處理中 |
| `IDEMPOTENCY_KEY_REUSED` | 422 | 同一個 Idempotency-Key 用在不同的請求內容 |
| `RATE_LIMITED` | 429 | 超過限流 |
| `INTERNAL_ERROR` | 500 | 伺服器錯誤 |
| `SERVICE_UNAVAILABLE` | 503 | 暫時無法服務 |
| **身分驗證** | | |
| `EMAIL_TAKEN`、`PHONE_TAKEN`、`USERNAME_TAKEN` | 409 | 已被使用 |
| `PASSWORD_TOO_WEAK` | 400 | 不符合密碼規則；`details.reasons` 列出原因 |
| `INVALID_CREDENTIALS` | 401 | 帳號或密碼錯誤 |
| `ACCOUNT_NOT_VERIFIED` | 403 | 帳號尚未完成驗證 |
| `ACCOUNT_SUSPENDED` | 403 | 帳號已被停權 |
| `ACCOUNT_DISABLED` | 403 | 帳號已停用 |
| `TOO_MANY_ATTEMPTS` | 429 | 登入嘗試過多，暫時鎖定 |
| `VERIFICATION_CODE_INVALID` | 400 | 驗證碼錯誤 |
| `VERIFICATION_CODE_EXPIRED` | 400 | 驗證碼過期 |
| `VERIFICATION_ATTEMPTS_EXCEEDED` | 400 | 驗證碼嘗試次數用盡，請重新發送 |
| `RESEND_COOLDOWN` | 429 | 重送冷卻中 |
| `USERNAME_CHANGE_TOO_SOON` | 409 | username 變更間隔未到 |
| `INVALID_IMAGE` | 400 | 頭像格式或大小不符 |
| `MFA_ENROLLMENT_REQUIRED` | 403 | 後台帳號必須先綁定 TOTP |
| `MFA_INVALID` | 401 | TOTP 或備用碼錯誤 |
| **會員** | | |
| `PLAN_NOT_AVAILABLE` | 409 | 方案未上架 |
| `MEMBERSHIP_ACTION_INVALID` | 409 | 這個操作不符合目前會員狀態；`details.suggested_action` 為 `PURCHASE`、`RENEW`、`UPGRADE`、`DOWNGRADE` 或 `PAY_RENEWAL` |
| `UPGRADE_NOT_ALLOWED` | 409 | 升級應付金額 ≤ 0，或目前在寬限期 |
| `DOWNGRADE_NOT_ALLOWED` | 409 | 目前沒有付費會員，或目標等級不是較低等級 |
| **點數** | | |
| `INSUFFICIENT_CREDITS` | 409 | 點數不足 |
| `ADJUSTMENT_REASON_REQUIRED` | 400 | 調整必須填寫原因與參考編號 |
| **訂單與付款** | | |
| `PRODUCT_NOT_AVAILABLE` | 409 | 商品未上架或數量不允許 |
| `ORDER_STATE_INVALID` | 409 | 訂單目前狀態不允許此操作 |
| `ORDER_NOT_PAYABLE` | 409 | 訂單不能付款 |
| `ORDER_EXPIRED` | 409 | 訂單已逾時 |
| `INSUFFICIENT_BALANCE` | 402 | USDT 餘額不足；`details` 含差額與入金地址（`PAYMENT-FLOW.md` §5） |
| **推薦** | | |
| `REFERRAL_CODE_INVALID` | 400 | 推薦碼無效 |
| `REFERRAL_REGENERATE_TOO_SOON` | 409 | 重新產生間隔未到 |
| `REFERRAL_CODE_SUSPENDED` | 409 | 推薦碼已被管理員停用 |
| `REWARD_STATE_INVALID` | 409 | 獎勵目前狀態不允許此操作 |
| `REVERSAL_EXCEEDS_REWARD` | 409 | 沖回金額超過尚未沖回的部分 |
| **退款與資金** | | |
| `REFUND_NOT_ALLOWED` | 409 | 訂單狀態不可退款 |
| `REFUND_ALREADY_EXISTS` | 409 | 該訂單已退款 |
| `REFUND_AMOUNT_EXCEEDS_PAID` | 400 | 退款金額超過實付 |
| `TX_VERIFICATION_FAILED` | 409 | 登記的鏈上交易驗證失敗（來源、目的或金額不符） |
| `APPROVAL_SELF_FORBIDDEN` | 403 | 需要雙人核准的操作，不能由發起者核准 |
| **後台** | | |
| `LAST_SUPER_ADMIN` | 409 | 不能移除最後一位 Super Administrator |
| `SETTING_INVALID_VALUE` | 400 | 設定值不符合型別或範圍 |

## 3. 會員端 API

「驗證」欄：公開 = 不需登入；會員 = 需要會員 access token。

### 3.1 身分驗證 `/auth`

| 方法 | 路徑 | 驗證 | 說明 |
| --- | --- | --- | --- |
| POST | `/auth/register` | 公開 | 註冊（email 或手機號擇一）；回傳驗證碼寄送資訊 |
| POST | `/auth/verify` | 公開 | 提交驗證碼；成功即啟用帳號並登入（回傳 token） |
| POST | `/auth/verification/resend` | 公開 | 重寄驗證碼 |
| POST | `/auth/login` | 公開 | 以 email、手機號或 username 加密碼登入 |
| POST | `/auth/refresh` | refresh token | 取得新的 access 與 refresh token |
| POST | `/auth/logout` | 會員 | 登出目前 session |
| POST | `/auth/logout-all` | 會員 | 登出所有裝置 |
| POST | `/auth/password/reset` | 公開 | 寄送重設碼；一律回 `202` |
| POST | `/auth/password/reset/confirm` | 公開 | 以重設碼設定新密碼；撤銷所有 session |
| POST | `/auth/password/change` | 會員 | 變更密碼；撤銷其他 session |

```json
// POST /auth/register
{ "email": "jane@example.com", "username": "jane_doe", "password": "…", "referral_code": "REF8K29X" }
// 201
{ "verification": { "channel": "EMAIL", "target": "ja***@example.com", "expires_in": 600, "resend_after": 60 } }

// POST /auth/login
{ "identifier": "jane_doe", "password": "…", "client_type": "WEB" }
// 200（WEB：refresh token 以 Set-Cookie 回傳；APP：放在 body 的 refresh_token）
{ "access_token": "eyJ…", "expires_in": 600, "user": { "id": "…", "username": "jane_doe" } }
```

### 3.2 會員資料 `/members`

| 方法 | 路徑 | 驗證 | 說明 |
| --- | --- | --- | --- |
| GET | `/members/me` | 會員 | 個人資料（REQ-022） |
| PATCH | `/members/me` | 會員 | 修改 username |
| PUT | `/members/me/avatar` | 會員 | 上傳頭像 |
| POST | `/members/me/email/change`、`/members/me/email/confirm` | 會員 | 變更 email：寄碼到新 email、提交驗證碼 |
| POST | `/members/me/phone/change`、`/members/me/phone/confirm` | 會員 | 變更手機號 |
| GET | `/members/me/sessions` | 會員 | 已登入的裝置 |
| DELETE | `/members/me/sessions/{session_id}` | 會員 | 登出指定裝置 |
| GET | `/members/me/membership` | 會員 | `/memberships/me` 的別名 |

```json
// GET /members/me
{
  "id": "0193…", "username": "jane_doe", "avatar_url": "https://…",
  "email": "jane@example.com", "phone": null,
  "status": "ACTIVE", "membership_tier": { "code": "VIP", "name": "VIP 會員" },
  "referral_code": "7K3QX9MP", "registered_at": "2026-11-01T08:00:00Z", "last_login_at": "2026-11-26T09:58:00Z"
}
```

### 3.3 會員方案 `/memberships`

| 方法 | 路徑 | 驗證 | 說明 |
| --- | --- | --- | --- |
| GET | `/memberships/tiers` | 公開 | 可購買的等級、方案與價格、權益說明 |
| GET | `/memberships/me` | 會員 | 目前等級、期間、已預付的下一期、自動續費、排定的降級、寬限期、權益 |
| GET | `/memberships/me/history` | 會員 | 會員異動歷史 |
| POST | `/memberships/purchase` | 會員 | 從 Free 購買；回傳訂單 |
| POST | `/memberships/renew` | 會員 | 續費（預設目前方案）；回傳訂單 |
| POST | `/memberships/upgrade/quote` | 會員 | 試算升級應付金額與新期間 |
| POST | `/memberships/upgrade` | 會員 | 升級；回傳訂單 |
| POST | `/memberships/downgrade` | 會員 | 排定降級（不建立訂單） |
| DELETE | `/memberships/downgrade` | 會員 | 取消排定的降級 |
| PATCH | `/memberships/me/auto-renew` | 會員 | `{ "enabled": true }` |

```json
// GET /memberships/me
{
  "tier": { "code": "VIP", "name": "VIP 會員", "rank": 20 },
  "status": "ACTIVE",
  "current_period": { "plan_id": "…", "billing_period": "MONTHLY", "starts_at": "2026-11-16T00:00:00Z", "ends_at": "2026-12-16T00:00:00Z" },
  "next_period": null,
  "auto_renew": true,
  "scheduled_downgrade": null,
  "grace_ends_at": null,
  "renewal_price": { "amount": "30.000000", "currency": "USDT" },
  "entitlements": { "ai.credits.per_cycle": 3000, "ai.generation.enabled": true, "ai.generation.daily_limit": 50 }
}

// POST /memberships/upgrade/quote  { "plan_id": "…" }
{
  "price": { "amount": "30.000000", "currency": "USDT" },
  "credit": { "amount": "5.161290", "currency": "USDT" },
  "amount_due": { "amount": "24.838710", "currency": "USDT" },
  "new_starts_at": "2026-01-16T00:00:00Z", "new_ends_at": "2026-02-16T00:00:00Z"
}
```

### 3.4 商品、點數錢包 `/products`、`/wallet`

| 方法 | 路徑 | 驗證 | 說明 |
| --- | --- | --- | --- |
| GET | `/products?type=CREDIT_PACK` | 公開 | 上架中的商品與目前價格 |
| GET | `/wallet/credits` | 會員 | 點數餘額：總額、各類型、即將到期 |
| GET | `/wallet/transactions` | 會員 | 點數交易紀錄（cursor 分頁；可依類型、時間篩選） |
| POST | `/wallet/credits/purchase` | 會員 | 購買點數包 `{ product_id, quantity }`；回傳訂單 |

```json
// GET /wallet/credits
{
  "total": 1350,
  "by_type": { "PROMOTIONAL": 0, "MEMBERSHIP": 350, "ADMIN": 0, "PURCHASED": 1000 },
  "expiring_soon": [ { "credit_type": "MEMBERSHIP", "amount": 350, "expires_at": "2026-12-16T00:00:00Z" } ]
}
```

### 3.5 USDT 餘額與入金 `/funds`

| 方法 | 路徑 | 驗證 | 說明 |
| --- | --- | --- | --- |
| GET | `/funds/balance` | 會員 | 站內 USDT 餘額 |
| GET | `/funds/deposit-addresses` | 會員 | 各鏈的入金地址（第一次呼叫時分配）與注意事項 |
| GET | `/funds/deposits` | 會員 | 入金紀錄與確認進度 |
| GET | `/funds/transactions` | 會員 | 餘額異動：入金、付款、退款 |

```json
// GET /funds/deposit-addresses
{
  "addresses": [
    { "chain": "TRON", "token": "USDT (TRC-20)", "address": "T…", "confirmations_required": 20 },
    { "chain": "BSC",  "token": "USDT (BEP-20)", "address": "0x…", "confirmations_required": 15 }
  ],
  "warning_code": "CROSS_CHAIN_NOT_RECOVERABLE"
}

// GET /funds/deposits
{ "items": [ { "id": "…", "chain": "TRON", "tx_hash": "…", "amount": "100.000000", "status": "DETECTED",
               "confirmations": 7, "confirmations_required": 20, "detected_at": "…" } ], "next_cursor": null }
```

### 3.6 訂單與付款 `/orders`、`/payments`

| 方法 | 路徑 | 驗證 | 說明 |
| --- | --- | --- | --- |
| POST | `/orders` | 會員 | 建立訂單 `{ product_id, quantity }`；會員方案會自動判斷是購買、續費或升級（較低等級回傳 `MEMBERSHIP_ACTION_INVALID`） |
| GET | `/orders` | 會員 | 訂單列表 |
| GET | `/orders/{order_id}` | 會員 | 訂單詳情，含狀態歷程與付款、退款紀錄；付款後前端以此查詢狀態 |
| POST | `/orders/{order_id}/cancel` | 會員 | 取消未付款訂單 |
| POST | `/payments/initiate` | 會員 | 以站內餘額付款 `{ order_id, method: "USDT_BALANCE" }`；不足回傳 `402` |
| POST | `/payments/webhooks/{provider}` | — | 本期不實作，保留給未來外部支付供應商 |

```json
// GET /orders/{order_id}
{
  "id": "…", "order_number": "PL20261126-7K3QX9", "order_type": "MEMBERSHIP_UPGRADE",
  "status": "COMPLETED", "payment_status": "PAID",
  "item": { "product_id": "…", "product_type": "MEMBERSHIP_PLAN", "name": "VIP 月費", "quantity": 1,
            "unit_price": { "amount": "30.000000", "currency": "USDT" } },
  "subtotal": { "amount": "30.000000", "currency": "USDT" },
  "discount": { "amount": "5.161290", "currency": "USDT" },
  "total":    { "amount": "24.838710", "currency": "USDT" },
  "payment_method": "USDT_BALANCE",
  "created_at": "…", "paid_at": "…", "completed_at": "…", "expires_at": "…",
  "history": [ { "status": "CREATED", "at": "…" }, { "status": "PENDING_PAYMENT", "at": "…" }, { "status": "PAID", "at": "…" } ],
  "refunds": []
}
```

### 3.7 推薦 `/referrals`

| 方法 | 路徑 | 驗證 | 說明 |
| --- | --- | --- | --- |
| GET | `/referrals/me` | 會員 | 推薦碼、推薦連結、狀態、下次可重新產生的時間 |
| POST | `/referrals/me/regenerate` | 會員 | 重新產生推薦碼 |
| GET | `/referrals/me/statistics` | 會員 | 邀請數、註冊數、付費數、各層統計、待結算與已確認金額 |
| GET | `/referrals/me/members` | 會員 | 下線（遮罩 username、註冊日期、是否已付費）；`?level=` 指定第幾層，預設 1，最大為目前的層數 |
| GET | `/referrals/me/rewards` | 會員 | 獎勵與沖回紀錄（含層級） |
| POST | `/referrals/validate` | 公開 | `{ code }` → `{ "valid": true }` |
| POST | `/referrals/visits` | 公開 | `{ code, visitor_id }` → `204`（一律） |

```json
// GET /referrals/me/statistics（目前 referral.reward_levels = 3）
{
  "invited": 120, "registered": 35, "paying": 9,
  "reward_levels": 3,
  "by_level": [
    { "level": 1, "registered": 35, "paying": 9 },
    { "level": 2, "registered": 61, "paying": 14 },
    { "level": 3, "registered": 20, "paying": 2 }
  ],
  "pending_rewards":   { "amount": "15.000000", "currency": "USDT" },
  "confirmed_rewards": { "amount": "42.483871", "currency": "USDT" }
}
```

- `registered`、`paying` 是第 1 層（直接推薦）的人數；`by_level` 一律回傳，層數為 1 時只有一筆。
- 獎勵金額包含所有層。

## 4. 後台 API `/admin`

所有後台端點都需要 `aud=admin` 且已完成 TOTP 的 session，並具備 `admin.access` 權限；下表的「權限」是額外需要的權限。
所有寫入操作都會寫入稽核日誌；標 ★ 的操作必須帶 `reason`。

### 4.1 後台登入 `/admin/auth`

| 方法 | 路徑 | 說明 |
| --- | --- | --- |
| POST | `/admin/auth/login` | 密碼驗證；回傳 `mfa_token` 與 `next_step`（`VERIFY` 或 `ENROLL`） |
| POST | `/admin/auth/mfa/enroll` | 以 `mfa_token` 取得 `otpauth://` URI |
| POST | `/admin/auth/mfa/verify` | 以 `mfa_token` 加 TOTP 或備用碼完成登入；首次綁定時回傳備用碼 |
| POST | `/admin/auth/refresh` | 換新 token |
| POST | `/admin/auth/logout` | 登出 |

### 4.2 會員 `/admin/members`

| 方法 | 路徑 | 權限 | 說明 |
| --- | --- | --- | --- |
| GET | `/admin/members` | `member.view` | 搜尋：email、手機、username、ID、推薦碼；篩選狀態與等級 |
| GET | `/admin/members/{id}` | `member.view` | 個人資料（有 `member.view_pii` 才顯示完整 email、手機） |
| PATCH | `/admin/members/{id}` | `member.update` | 編輯允許的欄位 ★ |
| POST | `/admin/members/{id}/suspend`、`/reactivate` | `member.status` | 停權、恢復 ★ |
| POST | `/admin/members/{id}/disable` | `member.disable` | 永久停用 ★ |
| POST | `/admin/members/{id}/sessions/revoke` | `member.status` | 強制登出所有裝置 ★ |
| GET | `/admin/members/{id}/memberships` | `member.view` | 會員期間與異動歷史 |
| GET | `/admin/members/{id}/referrals` | `referral.view` | 上線鏈、各層下線人數、直接推薦的被推薦人、綁定嘗試 |
| GET | `/admin/members/{id}/credit-transactions` | `wallet.view` | 點數交易 |
| GET | `/admin/members/{id}/funds` | `funds.view` | USDT 餘額、入金、分錄 |
| GET | `/admin/members/{id}/orders` | `order.view` | 訂單 |
| GET | `/admin/members/{id}/security-events` | `member.view_pii` | 登入與安全事件 |

### 4.3 會員等級與商品 `/admin/memberships`、`/admin/products`

| 方法 | 路徑 | 權限 | 說明 |
| --- | --- | --- | --- |
| GET | `/admin/memberships/tiers` | — | 等級、權益、方案 |
| POST | `/admin/memberships/tiers` | `membership.manage` | 建立等級 |
| PATCH | `/admin/memberships/tiers/{id}` | `membership.manage` | 修改名稱、說明、優先序、啟用狀態 ★ |
| PUT | `/admin/memberships/tiers/{id}/entitlements` | `membership.manage` | 設定權益 ★ |
| GET | `/admin/memberships/entitlement-keys` | — | 可用的權益鍵、型別、預設值 |
| POST | `/admin/memberships/tiers/{id}/plans` | `membership.manage` + `product.manage` | 建立月費或年費方案（同時建立商品） |
| PATCH | `/admin/memberships/plans/{id}` | `membership.manage` | 上下架 ★ |
| GET / POST / PATCH | `/admin/products`、`/admin/products/{id}` | `product.manage` | 點數包商品 |
| POST | `/admin/products/{id}/prices` | `product.manage` | 排定新價格 `{ amount, effective_from }` ★ |

### 4.4 推薦 `/admin/referrals`

| 方法 | 路徑 | 權限 | 說明 |
| --- | --- | --- | --- |
| GET | `/admin/referrals/codes` | `referral.view` | 搜尋推薦碼 |
| POST | `/admin/referrals/codes/{id}/suspend`、`/reactivate` | `referral.code.manage` | 停用、恢復 ★ |
| GET | `/admin/referrals/bindings` | `referral.view` | 推薦關係 |
| GET | `/admin/referrals/bind-attempts` | `referral.view` | 綁定嘗試與失敗原因 |
| GET | `/admin/referrals/performance` | `referral.view` | 推薦成效（依推薦人、期間） |
| GET / POST | `/admin/referrals/rules` | `referral.view` / `referral.rule.manage` | 獎勵規則（含各層設定 `levels`） |
| PATCH | `/admin/referrals/rules/{id}` | `referral.rule.manage` | 修改規則 ★ |
| GET | `/admin/referrals/rewards` | `referral.view` | 獎勵列表（可依狀態、受益人、購買者、層級篩選） |
| POST | `/admin/referrals/rewards/{id}/hold`、`/release`、`/cancel` | `referral.reward.review` | 凍結、放行、取消 ★ |
| POST | `/admin/referrals/rewards/{id}/reverse` | `referral.reward.review` | 沖回 `{ amount, reason }` ★ |

獎勵層數用系統設定端點修改：`PATCH /admin/settings/referral.reward_levels { "value": 3, "reason": "..." }`（需 `referral.rule.manage`，1 ~ 10）。

```json
// POST /admin/referrals/rules
{
  "name": "會員方案三層獎勵",
  "priority": 10,
  "eligible_product_types": ["MEMBERSHIP_PLAN"],
  "min_order_amount": "10.000000",
  "settlement_delay_days": 14,
  "first_purchase_only": false,
  "levels": [
    { "level": 1, "reward_type": "PERCENT", "percent_bps": 1000 },
    { "level": 2, "reward_type": "PERCENT", "percent_bps": 500 },
    { "level": 3, "reward_type": "PERCENT", "percent_bps": 200, "max_reward_amount": "5.000000" }
  ]
}
```

- `levels` 必須從 1 開始連續、最多 10 層，各層 `percent_bps` 合計不超過 10000；不符時回 `VALIDATION_ERROR`。
- 層數超過目前 `referral.reward_levels` 的設定會被接受，回應帶 `warning_code: "LEVELS_EXCEED_CURRENT_SETTING"`，提醒多出來的層暫時不發。

### 4.5 點數錢包 `/admin/wallets`

| 方法 | 路徑 | 權限 | 說明 |
| --- | --- | --- | --- |
| GET | `/admin/wallets` | `wallet.view` | 搜尋錢包 |
| GET | `/admin/wallets/{user_id}` | `wallet.view` | 餘額與批次（含到期資訊） |
| GET | `/admin/wallets/{user_id}/transactions` | `wallet.view` | 交易紀錄 |
| POST | `/admin/wallets/{user_id}/promotional-grants` | `wallet.grant_promo` | 發放促銷點數 `{ amount, expires_at, campaign, reason, reference }` ★ |
| POST | `/admin/wallets/{user_id}/adjustments` | `wallet.adjust` | 調整 `{ direction: "IN" \| "OUT", amount, reason, reference }` ★ |

### 4.6 訂單與退款 `/admin/orders`、`/admin/refunds`

| 方法 | 路徑 | 權限 | 說明 |
| --- | --- | --- | --- |
| GET | `/admin/orders` | `order.view` | 搜尋訂單（編號、會員、狀態、類型、期間） |
| GET | `/admin/orders/{id}` | `order.view` | 詳情：狀態歷程、付款、退款、履約錯誤 |
| POST | `/admin/orders/{id}/retry-fulfillment` | `order.retry_fulfillment` | 重試 `FAILED` 訂單的履約 ★ |
| GET | `/admin/refunds` | `order.view` | 退款列表 |
| POST | `/admin/refunds` | `refund.manage` | 退款：金額退回使用者的站內 USDT 餘額，並在同一交易內完成反向處理 ★ |

```json
// POST /admin/refunds
{
  "order_id": "…",
  "amount": "24.838710",
  "reason": "使用者於客服工單 #1234 申請退款，主管同意",
  "reference": "CS-1234"
}
// 201
{
  "id": "…", "status": "COMPLETED",
  "amount": { "amount": "24.838710", "currency": "USDT" },
  "credited_to": "USDT_BALANCE", "order_status": "REFUNDED",
  "created_by": "…", "completed_at": "…"
}
```

### 4.7 資金管理 `/admin/treasury`

| 方法 | 路徑 | 權限 | 說明 |
| --- | --- | --- | --- |
| GET | `/admin/treasury/overview` | `treasury.view` | 各鏈：入金地址彙總、冷錢包 USDT、熱錢包手續費資產與水位、轉帳中的金額、轉出開關 |
| GET | `/admin/treasury/deposits` | `treasury.view` | 入金列表 |
| GET | `/admin/treasury/held-deposits` | `treasury.view` | 異常入金佇列 |
| POST | `/admin/treasury/held-deposits/{id}/resolve` | `treasury.manage` | `{ action: "CREDIT_USER" \| "WRITE_OFF", reason }`；`WRITE_OFF` 需另一位管理員核准；不在鏈上退回 ★ |
| POST | `/admin/treasury/held-deposits/{id}/resolve/approve` | `treasury.manage` | 核准上述處理（不能是發起者） |
| GET | `/admin/treasury/transfers` | `treasury.view` | 鏈上轉帳列表（歸集、補手續費、平台資金投入與提出） |
| POST | `/admin/treasury/capital-movements` | `treasury.manage` | 登記平台資金投入（購入手續費用的 TRX、BNB）或從冷錢包提出 `{ type, chain, asset, tx_hash }`，系統在鏈上驗證 ★ |
| POST | `/admin/treasury/capital-movements/{id}/approve` | `treasury.manage` | 核准資金提出（不能是發起者） |
| GET | `/admin/treasury/accounts` | `treasury.view` | 平台資金帳戶餘額 |
| GET | `/admin/treasury/reconciliations` | `treasury.view` | 點數、資金帳本、鏈上三種對帳結果 |

### 4.8 報表 `/admin/reports`

| 方法 | 路徑 | 權限 | 說明 |
| --- | --- | --- | --- |
| GET | `/admin/reports/dashboard?from=…&to=…` | `report.view` | 儀表板指標（REQ-120） |
| POST | `/admin/reports/exports` | `report.export` | `{ type: "ORDERS" \| "CREDIT_TRANSACTIONS" \| "FUND_ENTRIES", filters }` → `202 { id }` |
| GET | `/admin/reports/exports/{id}` | `report.export` | 匯出狀態；完成時回傳短時效下載網址 |

```json
// GET /admin/reports/dashboard
{
  "members": { "total": 15230, "active_30d": 4120, "new": 320, "vip": 410 },
  "orders": { "total": 980, "paid": 912 },
  "revenue": { "gross": { "amount": "21450.000000", "currency": "USDT" },
               "refunds": { "amount": "120.000000", "currency": "USDT" } },
  "credits_issued": 2850000,
  "referrals": { "bindings": 210, "paying_referees": 64,
                 "rewards_confirmed": { "amount": "830.000000", "currency": "USDT" } },
  "deposits": { "count": 640, "amount": { "amount": "25300.000000", "currency": "USDT" } }
}
```

### 4.9 系統管理 `/admin/settings`、`/admin/roles`、`/admin/admin-users`、`/admin/audit-logs`

| 方法 | 路徑 | 權限 | 說明 |
| --- | --- | --- | --- |
| GET | `/admin/settings` | `settings.view` | 所有設定：目前值、預設值、型別、修改所需權限 |
| PATCH | `/admin/settings/{key}` | 依該設定的權限（`BUSINESS-RULES.md` §10） | 修改設定 ★ |
| GET | `/admin/settings/integrations` | `settings.view` | 郵件、簡訊、RPC、簽章服務的連線狀態（不顯示密鑰） |
| GET / POST / PATCH | `/admin/roles`、`/admin/roles/{id}` | `rbac.manage` | 角色 ★ |
| PUT | `/admin/roles/{id}/permissions` | `rbac.manage` | 設定角色權限 ★ |
| GET | `/admin/permissions` | `rbac.manage` | 所有權限代碼 |
| GET / POST / PATCH | `/admin/admin-users`、`/admin/admin-users/{id}` | `admin_user.manage` | 管理員帳號 ★ |
| PUT | `/admin/admin-users/{id}/roles` | `admin_user.manage` | 指派角色 ★ |
| POST | `/admin/admin-users/{id}/reset-mfa` | `admin_user.manage` | 重設 TOTP ★ |
| GET | `/admin/audit-logs` | `audit.view` | 搜尋稽核日誌（操作者、動作、對象、期間） |

## 5. 系統端點（不在 `/api/v1` 之下）

| 方法 | 路徑 | 說明 |
| --- | --- | --- |
| GET | `/health/live` | 程序存活 |
| GET | `/health/ready` | 資料庫、Redis 可連線 |
| GET | `/api/v1/schema`、`/api/v1/docs` | OpenAPI 與 Swagger UI；正式環境限內部網路或需登入 |

## 6. spec §10 端點對照

| spec 端點 | 本規格 |
| --- | --- |
| `POST /auth/register`、`/login`、`/logout`、`/refresh`、`/password/reset` | 保留；另補 `/auth/verify`、`/auth/verification/resend`、`/auth/logout-all`、`/auth/password/reset/confirm`、`/auth/password/change` |
| `GET / PATCH /members/me`、`GET /members/me/membership` | 保留；後者為 `/memberships/me` 的別名 |
| `GET /referrals/me`、`/me/statistics`、`/me/members`、`/me/rewards`、`POST /referrals/validate` | 保留；另補 `/referrals/me/regenerate`、`/referrals/visits` |
| `GET /wallet/credits`、`/wallet/transactions`、`POST /wallet/credits/purchase` | 保留 |
| `GET /memberships/tiers`、`/memberships/me`、`POST /memberships/purchase`、`/renew` | 保留；另補升級、降級、自動續費 |
| `POST /orders`、`GET /orders`、`GET /orders/{id}`、`POST /orders/{id}/cancel` | 保留 |
| `POST /payments/initiate` | 保留，語意為「以站內餘額付款」 |
| `POST /payments/webhooks/{provider}` | 本期不實作 |
| `/api/v1/admin/members/`、`memberships/`、`referrals/`、`wallets/`、`orders/`、`reports/`、`settings/` | 保留（去掉結尾斜線）；另補 `auth`、`products`、`refunds`、`treasury`、`roles`、`admin-users`、`audit-logs` |
| （spec 未列） | `/funds/*`：USDT 餘額、入金地址、入金紀錄。出金端點留給未來模組（D-14） |
