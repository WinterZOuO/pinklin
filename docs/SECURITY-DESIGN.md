---
title: Pinklin Phase 1 — Security Design
type: DESIGN
status: draft            # draft → approved
owner: jackylau
prd: docs/project/PRD-project.md
architecture: docs/ARCHITECTURE.md
created: 2026-11-26
updated: 2026-11-26
---

# 安全設計

> 涵蓋 spec §3 安全要求、§8 權限、§11 全部項目，以及 REQ-017 ~ REQ-035、REQ-150 ~ REQ-156。
> 鏈上資金與金鑰的安全另見 `CHAIN-CUSTODY.md`。

## 1. 要保護的資產

| 資產 | 為什麼重要 |
| --- | --- |
| 使用者的 USDT 餘額與平台的鏈上資產 | 真實的錢；外洩或被盜無法追回 |
| 私鑰與簽章服務 | 掌握它就能轉走資金 |
| 後台管理員帳號 | 可以退款、調整點數、改設定 |
| 會員帳號與 session | 可以花用餘額與點數 |
| AI 點數、推薦獎勵 | 可被濫用來換取服務或日後的提領 |
| 個人資料（email、手機、IP、入金來源地址） | 法規與信任 |
| 帳本與稽核記錄的完整性 | 對帳與追責的依據 |

## 2. 威脅模型

| # | 威脅 | 攻擊方式 | 對策 | 位置 |
| --- | --- | --- | --- | --- |
| T1 | 盜用會員帳號 | 撞庫、暴力破解 | 依識別值與 IP 的漸進式鎖定；外洩密碼黑名單；Argon2id | §4.1、§5 |
| T2 | 竊取 token | XSS 取得 token、竊聽 | access token 只在記憶體；refresh token 放 HttpOnly cookie；CSP；HTTPS + HSTS | §4.3、§7 |
| T3 | refresh token 被盜後持續使用 | 重放舊 token | 每次輪替；舊 token 再出現即撤銷整個 session | §4.3 |
| T4 | 探測帳號是否存在 | 比對登入、忘記密碼的回應 | 一致的錯誤訊息與回應；以識別值計數，不分帳號是否存在 | §4.1 |
| T5 | 簡訊灌量（簡訊費用攻擊） | 大量觸發驗證簡訊 | 每號碼、每 IP、每國碼上限；重送冷卻；異常告警 | §5 |
| T6 | 越權存取他人資料 | 改 URL 中的 ID | 會員端 API 只以 token 中的使用者為範圍，不接受外部傳入的 user_id；物件層級檢查 | §6 |
| T7 | 後台帳號被盜或內部人員濫權 | 釣魚、惡意員工 | 強制 TOTP；最小權限；異常入金沖銷與資金提出需雙人核准；退款只進站內餘額；所有寫入操作稽核；後台獨立網域與 IP 白名單 | §3、§4.4 |
| T8 | 竄改帳本或稽核記錄 | SQL、應用程式漏洞 | 只能新增的表以觸發器保護；應用程式資料庫帳號沒有 DDL 與停用觸發器權限；每日對帳 | §10 |
| T9 | 偽造入金 | 仿冒代幣、重組、假 RPC 資料 | 只認可設定的合約；確認數；大額雙來源確認 | `CHAIN-CUSTODY.md` §5 |
| T10 | 盜轉平台資金 | 入侵主系統後要求簽章 | 私鑰只在簽章服務；簽章服務只允許平台地址之間的轉帳，並獨立執行金額上限；使用者的 USDT 在離線冷錢包；暫停開關 | `CHAIN-CUSTODY.md` §11 |
| T11 | 入金地址被調包 | 改掉主系統中的擴展公鑰 | 新地址與簽章服務的衍生結果比對；定期抽查 | `CHAIN-CUSTODY.md` §4 |
| T12 | 管理員濫用退款 | 把錢退給串通的使用者 | 退款只進站內餘額，本期無法出金；必須填原因與工單號；全程稽核並定期抽查 | `PAYMENT-FLOW.md` §10 |
| T13 | 推薦作弊 | 自建假帳號刷獎勵；多層時自建一串假帳號，每層都領 | 驗證後才綁定；結算延遲；同 IP / 裝置自動凍結（每一層分別比對）；退款沖回；層數預設 1，修改需 `referral.rule.manage` 並寫入稽核日誌 | `REFERRAL-FLOW.md` §8 |
| T14 | 重複扣款或重複入帳 | 重送請求、並發 | 冪等鍵；唯一約束；列鎖 | `WALLET-LEDGER.md` |
| T15 | 注入攻擊 | SQL 注入、惡意檔案 | ORM 與參數化查詢；序列化層驗證；頭像重新編碼 | §7 |
| T16 | 密鑰外洩 | 程式碼、日誌、錯誤訊息 | 密鑰只在環境變數或密鑰管理服務；日誌遮罩；錯誤訊息不含內部資訊 | §8、§9 |
| T17 | 依賴套件漏洞 | 供應鏈攻擊 | 鎖定版本；CI 漏洞掃描；最小依賴 | §11 |

## 3. 授權（RBAC）

### 3.1 權限清單

| 權限代碼 | 允許的操作 |
| --- | --- |
| `admin.access` | 登入後台 |
| `member.view` | 搜尋與檢視會員（個資遮罩） |
| `member.view_pii` | 檢視完整的 email、手機號 |
| `member.update` | 編輯會員允許的欄位 |
| `member.status` | 停權、恢復 |
| `member.disable` | 永久停用 |
| `membership.manage` | 等級、權益、方案設定 |
| `product.manage` | 商品與價格 |
| `referral.view` | 推薦碼、推薦關係、成效 |
| `referral.code.manage` | 停用、恢復推薦碼 |
| `referral.rule.manage` | 獎勵規則、獎勵層數 |
| `referral.reward.review` | 凍結、放行、取消、沖回獎勵 |
| `wallet.view` | 檢視點數錢包與交易 |
| `wallet.grant_promo` | 發放促銷點數 |
| `wallet.adjust` | 點數調整 |
| `funds.view` | 檢視使用者 USDT 餘額與紀錄 |
| `order.view` | 檢視訂單與付款紀錄 |
| `order.retry_fulfillment` | 重試失敗的履約 |
| `refund.manage` | 退款（退回站內餘額） |
| `treasury.view` | 檢視入金、轉出、熱錢包、對帳結果 |
| `treasury.manage` | 處理異常入金、登記平台資金投入與提出 |
| `treasury.limits` | 修改鏈上相關設定與轉出總開關 |
| `report.view` | 儀表板 |
| `report.export` | 匯出報表 |
| `settings.view` | 檢視系統設定 |
| `settings.manage` | 修改一般系統設定 |
| `rbac.manage` | 角色與權限 |
| `admin_user.manage` | 管理員帳號 |
| `audit.view` | 稽核日誌 |

### 3.2 預設權限矩陣

Super Administrator 擁有全部權限。下表是其他三個角色：

| 權限 | Finance | Member Support | Operations |
| --- | :---: | :---: | :---: |
| `admin.access` | ✓ | ✓ | ✓ |
| `member.view` | ✓ | ✓ | ✓ |
| `member.view_pii` | ✓ | ✓ | |
| `member.update`、`member.status` | | ✓ | |
| `membership.manage`、`product.manage` | | | ✓ |
| `referral.view` | ✓ | ✓ | ✓ |
| `referral.code.manage` | | ✓ | ✓ |
| `referral.rule.manage` | | | ✓ |
| `referral.reward.review` | ✓ | | |
| `wallet.view` | ✓ | ✓ | ✓ |
| `wallet.grant_promo` | | | ✓ |
| `wallet.adjust` | ✓ | | |
| `funds.view` | ✓ | ✓ | |
| `order.view` | ✓ | ✓ | ✓ |
| `order.retry_fulfillment` | ✓ | | |
| `refund.manage` | ✓ | | |
| `treasury.view`、`treasury.manage` | ✓ | | |
| `report.view` | ✓ | | ✓ |
| `report.export` | ✓ | | |
| `settings.view` | ✓ | ✓ | ✓ |
| `settings.manage` | | | ✓ |
| `audit.view` | ✓ | | |

只有 Super Administrator 有：`member.disable`、`treasury.limits`、`rbac.manage`、`admin_user.manage`。

### 3.3 規則
- **預設拒絕**：每個 API view 必須宣告 `required_permissions`（會員端宣告 `member_self`）。CI 測試列舉所有 URL，沒有宣告的 view 讓測試失敗。
- **職責分離**：需要雙人核准的操作（異常入金沖銷、從冷錢包提出資金、資金帳本更正），「發起者 ≠ 核准者」由服務層與資料庫 `CHECK` 雙重保證，不靠角色設定。
- **不可移除最後一位 Super Administrator**；也不能移除自己的 `rbac.manage`。
- **會員等級不是授權依據**（REQ-033）：AI 功能等是否可用，由權益服務判斷。

## 4. 身分驗證

### 4.1 密碼與登入
- Argon2id 雜湊（Django `Argon2PasswordHasher`），參數依正式主機壓測調整到單次約 100–300 毫秒；參數升級時，使用者下次登入自動重新雜湊。
- 密碼規則見 `BUSINESS-RULES.md` §1.4；鎖定規則見 §1.5。
- 登入失敗一律回傳 `INVALID_CREDENTIALS`；只有在密碼正確時，才回傳 `ACCOUNT_SUSPENDED`、`ACCOUNT_DISABLED`、`ACCOUNT_NOT_VERIFIED`。
- 忘記密碼一律回傳 `202`，不論帳號是否存在。

### 4.2 驗證碼
- 6 位數字，以 `secrets` 產生；只存 HMAC-SHA256（伺服器密鑰 + 記錄 ID + 驗證碼），避免資料庫外洩後被離線暴力破解。
- 有效期、嘗試次數、冷卻與發送上限見 `BUSINESS-RULES.md` §1.2；比對使用固定時間比較。

### 4.3 Token 與 Session

| 項目 | 會員 | 後台 |
| --- | --- | --- |
| access token | JWT，10 分鐘 | JWT，5 分鐘 |
| refresh token | 256-bit 亂數，只存雜湊；閒置 30 天、絕對 90 天 | 閒置 30 分鐘、絕對 12 小時 |
| 存放（Web） | access 在記憶體；refresh 在 `HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth` cookie | 同左，cookie 路徑為 `/api/v1/admin/auth` |
| 存放（App） | refresh 在回應 body，由 App 存在系統安全儲存區 | 不適用 |

- **JWT**：以 EdDSA（Ed25519）簽署，標頭帶 `kid` 以支援金鑰輪替；claims 包含 `iss`、`aud`（`member` / `admin`）、`sub`、`sid`、`iat`、`exp`、`jti`。
- **每個請求**：驗簽與有效期後，以 `sid` 查 Redis 快取的 session 與帳號狀態（快取 30 秒；撤銷時主動清除）。
- **輪替**：每次 refresh 都換新 token；收到 `previous_refresh_token_hash` 對應的舊 token 時，撤銷整個 session（`REUSE_DETECTED`）並寫入安全事件。
- **撤銷時機**：登出、登出所有裝置、改密碼、重設密碼、停權、管理員強制登出。

### 4.4 後台二階段驗證（REQ-035）
1. `POST /admin/auth/login`：密碼正確後回傳 5 分鐘有效的 `mfa_token`，尚未啟用 TOTP 者要求先綁定。
2. 綁定：回傳 `otpauth://` URI；使用者以驗證器 App 掃描後輸入一次驗證碼確認，同時取得 10 組一次性備用碼（只存雜湊）。
3. `POST /admin/auth/mfa/verify`：驗證 TOTP（容許前後各 1 個時間步），並記錄已使用的時間步，防止同一組驗證碼重複使用。
4. TOTP 秘密以 `FIELD_ENCRYPTION_KEY` 加密儲存。
5. 重設 TOTP 只能由 Super Administrator 執行，並寫入稽核日誌。

## 5. 限流

以 Redis 計數，超過時回傳 `429 RATE_LIMITED` 並帶 `Retry-After`。數值都是系統設定（`ratelimit.*`）。

| 範圍 | 預設上限 |
| --- | --- |
| 匿名，每 IP | 60 次 / 分鐘 |
| 已登入會員，每人 | 300 次 / 分鐘 |
| 後台，每人 | 600 次 / 分鐘 |
| 登入，每 IP | 10 次 / 分鐘（另有依識別值的鎖定） |
| 註冊，每 IP | 5 次 / 小時 |
| 驗證碼發送 | 見 `BUSINESS-RULES.md` §1.2 |
| 推薦碼驗證，每 IP | 20 次 / 分鐘 |
| 推薦連結造訪記錄，每 IP | 30 次 / 分鐘 |
| 付款，每人 | 10 次 / 分鐘 |
| 取得入金地址，每人 | 10 次 / 分鐘 |
| 報表匯出，每位管理員 | 10 次 / 小時 |

## 6. API 授權實作
- **會員端**：所有資源都以 `request.user` 限定範圍；路徑中的 ID 不屬於該使用者時回傳 `404`（不透露資源存在）。
- **後台**：只接受 `aud=admin` 且已完成 TOTP 的 session；權限不足回傳 `403 PERMISSION_DENIED`。
- **後台寫入操作**：必須在同一交易內寫入 `admin_audit_logs`；涉及點數、退款、資金、設定的操作，必須帶原因。

## 7. Web 與輸入安全

| 項目 | 作法 |
| --- | --- |
| HTTPS | 非本地環境一律 HTTPS；HSTS（`max-age=31536000; includeSubDomains`） |
| CORS | 只允許設定中的前台與後台網域；允許 credentials 的端點限定 auth 路徑 |
| CSRF | 一般 API 用 `Authorization` header，不受 CSRF 影響。使用 cookie 的端點（refresh、logout）要求 `SameSite=Strict`、`Origin` 在白名單內，且帶有自訂 header `X-Pinklin-Client` |
| 安全標頭 | CSP（`default-src 'self'`；`frame-ancestors 'none'`；script 使用 nonce）、`X-Content-Type-Options: nosniff`、`Referrer-Policy: strict-origin-when-cross-origin`、`Permissions-Policy` |
| 輸入驗證 | 所有輸入都經過 DRF serializer 驗證型別、長度、格式；未知欄位拒絕 |
| SQL 注入 | 只用 Django ORM；必要的原生 SQL 一律參數化，並在 code review 時特別檢查 |
| 頭像上傳 | 最大 2 MB；只接受 JPEG、PNG、WebP；以 Pillow 實際解碼驗證；重新編碼為 512×512 WebP 並移除 EXIF；檔名使用隨機 key；從物件儲存提供，並設定正確的 Content-Type |
| 轉址 | 前端的 `next` 等轉址參數只接受站內相對路徑 |
| 錯誤訊息 | 正式環境不回傳堆疊或內部資訊，只回傳錯誤碼與 `request_id` |

## 8. 密鑰與設定

| 類別 | 例子 | 存放 |
| --- | --- | --- |
| 應用程式密鑰 | `DJANGO_SECRET_KEY`、`JWT_PRIVATE_KEY`、`JWT_PUBLIC_KEYS`、`FIELD_ENCRYPTION_KEY`、`VERIFICATION_CODE_PEPPER` | 環境變數，正式環境由密鑰管理服務注入 |
| 基礎設施 | 資料庫、Redis 連線字串 | 同上 |
| 外部服務 | 郵件、簡訊、RPC 的 API key | 同上 |
| 鏈上 | 入金擴展公鑰、USDT 合約地址、簽章服務連線憑證 | 同上 |
| 私鑰 | 主種子 | 只在簽章服務，KMS / HSM 或 MPC（D-15） |

- `.env.example` 只列名稱與說明，不含真實值。
- 密鑰要能輪替：JWT 以 `kid` 支援新舊金鑰並存；欄位加密金鑰支援多版本解密。

## 9. 個人資料保護（REQ-154）

| 等級 | 資料 | 處理 |
| --- | --- | --- |
| 機密 | 密碼雜湊、TOTP 秘密、token 雜湊、驗證碼雜湊 | 永不經 API 回傳、永不寫入日誌 |
| 個資 | email、手機號、IP、裝置雜湊、入金來源地址 | 列表與日誌遮罩；完整內容需 `member.view_pii` 或財務權限 |
| 一般 | username、會員等級、訂單編號 | 正常顯示 |

- **遮罩格式**：email `ja***@gmail.com`；手機 `+852 ****5678`；鏈上地址 `T9yDk2…3xKq`（前 6 後 4）。
- **日誌過濾器**：移除 `password`、`token`、`refresh_token`、`code`、`otp`、`secret`、`authorization`、`cookie` 等欄位，並遮罩 email 與手機號格式的字串。
- **靜態加密**：資料庫磁碟與備份加密；TOTP 秘密另做欄位加密。
- **匯出**：需 `report.export` 權限；匯出檔 24 小時後刪除；下載連結為短時效簽章網址；匯出行為寫入稽核日誌。

## 10. 稽核與監控

| 記錄 | 內容 |
| --- | --- |
| `security_events` | 登入成功與失敗、鎖定、token 重放、密碼變更與重設、驗證、狀態變更、TOTP 綁定與失敗 |
| `admin_audit_logs` | 所有後台寫入操作：操作者、session、IP、動作、對象、前後值、原因、參考編號 |
| 簽章日誌 | 簽章服務的每次簽署與拒絕，雜湊串接 |

| 安全告警 | 條件 |
| --- | --- |
| 撞庫 | 登入失敗率或鎖定數突增 |
| token 竊用 | 出現 `REUSE_DETECTED` |
| 後台異常 | 非白名單 IP 嘗試登入後台；TOTP 連續失敗 |
| 資金 | 簽章服務拒絕、對帳不一致、熱錢包異常變動（`CHAIN-CUSTODY.md` §12） |

資料庫帳號：
- `app_rw`：只能讀寫應用程式資料表，不能 `TRUNCATE`、DDL 或停用觸發器。
- `migrator`：執行 migration，只在部署時使用。
- 簽章服務使用自己的資料庫與帳號。

## 11. 供應鏈與部署
- Python 與 Node 依賴都使用鎖定檔；CI 執行 `pip-audit`、`pnpm audit` 與容器映像掃描。
- 容器以非 root 使用者執行、根檔案系統唯讀、移除不需要的 Linux capabilities。
- 後台網域在正式環境加上 IP 白名單或 VPN。
- 每次新增依賴都要說明理由（spec §15 第 12 條）。

## 12. 事故處理
1. **偵測**：告警通知值班人員。
2. **止血**：依情況關閉轉出開關、撤銷 session、停用帳號或角色。
3. **保全證據**：保留日誌、稽核記錄、簽章日誌與資料庫快照。
4. **復原**：依 `CHAIN-CUSTODY.md` §12.3 的事故處理表執行。
5. **檢討**：事後撰寫檢討，必要時新增 ADR 或修改本文件。

## 13. 安全測試（對應 spec §12）

| 類別 | 測試 |
| --- | --- |
| 驗證 | 正確註冊；重複註冊被拒；錯誤帳密；停權帳號無法登入；token 刷新與輪替；舊 refresh token 重放撤銷整個 session；登入鎖定；忘記密碼不透露帳號是否存在 |
| 授權 | 每個角色依矩陣允許與拒絕；未登入回傳 401；權限不足回傳 403；會員存取他人資源回傳 404；所有 view 都有宣告權限 |
| 後台 | 未完成 TOTP 無法使用後台 API；點數調整與退款必須附原因並產生稽核日誌；雙人核准的操作不能由發起者核准 |
| 完整性 | 只能新增的表無法 `UPDATE` / `DELETE`；分錄不平衡時交易失敗；使用者餘額不能為負 |
| 輸入 | 超長或格式錯誤的輸入被拒；偽裝成圖片的檔案被拒 |
