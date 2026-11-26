---
title: Pinklin Phase 1 — Business Rules
type: DESIGN
status: draft            # draft → approved
owner: jackylau
prd: docs/project/PRD-project.md
architecture: docs/ARCHITECTURE.md
created: 2026-11-26
updated: 2026-11-26
---

# 業務規則

> 所有業務規則集中在這份文件。凡是數值都是系統設定（§10），這裡寫的是預設值。
> 規則改變時先改這份文件，再改程式；PRD 的待確認決策（D-NN）定案後同步更新。

## 1. 帳號

### 1.1 註冊
- 識別方式：email 或手機號二擇一（D-12），加上 username 與密碼。
- **username**：3–30 字元，只能用英文字母、數字、底線，且必須以字母開頭；不分大小寫唯一；保留字（如 `admin`、`support`、`pinklin`、`system`）不可使用。
- **email 正規化**：去除前後空白、轉小寫。不去除 Gmail 的點與 `+` 標籤，以免誤判不同人為同一人；濫用由推薦風控處理（§6.5）。
- **手機號**：以 `phonenumbers` 函式庫解析並驗證，儲存為 E.164 格式。
- **重複檢查**：

  | 已存在的帳號狀態 | 用同一 email / 手機註冊 | 用同一 username 註冊 |
  | --- | --- | --- |
  | `ACTIVE`、`SUSPENDED`、`DISABLED` | 拒絕（`EMAIL_TAKEN` / `PHONE_TAKEN`） | 拒絕（`USERNAME_TAKEN`） |
  | `PENDING_VERIFICATION` | 覆寫該待驗證帳號的 username 與密碼，重寄驗證碼（能收到驗證碼的人才是真正擁有者） | 拒絕，直到該待驗證帳號過期 |

- **推薦碼**：選填。有填時，註冊當下先檢查是否有效，無效回傳 `REFERRAL_CODE_INVALID`（前端讓使用者移除後重送）；驗證成功時再檢查一次（§6.2）。

### 1.2 驗證
- 驗證碼為 6 位數字，有效 10 分鐘，最多嘗試 5 次，重送冷卻 60 秒。
- 每個 email / 手機號每小時最多發送 5 次；每個 IP 每小時最多 20 次；簡訊另有每個國碼的每日上限。
- 驗證成功時，在同一個資料庫交易內：帳號轉 `ACTIVE`、建立推薦綁定（如有）、產生推薦碼、建立點數錢包與 USDT 帳戶（D-13）。
- 待驗證帳號 72 小時內未完成驗證，由排程硬刪除（此時沒有任何金融資料）。

### 1.3 帳號狀態

| 從 | 到 | 觸發者 | 效果 |
| --- | --- | --- | --- |
| `PENDING_VERIFICATION` | `ACTIVE` | 使用者完成驗證 | 見 §1.2 |
| `ACTIVE` | `SUSPENDED` | Member Support（需填原因） | 立即撤銷所有 session；暫停自動續費；新的推薦獎勵不產生，待結算獎勵轉 `ON_HOLD` |
| `SUSPENDED` | `ACTIVE` | Member Support（需填原因） | 恢復登入；自動續費恢復；`ON_HOLD` 獎勵需人工放行 |
| `ACTIVE` / `SUSPENDED` | `DISABLED` | Super Administrator（需填原因） | 永久停用，不可恢復；所有金融記錄保留；剩餘 USDT 餘額由財務依條款人工處理（D-14） |

- 停權期間不扣自動續費，會員照原到期時間結束。
- 所有轉換寫入 `account_status_history` 與 `security_events`。

### 1.4 密碼
- 長度 10–128 字元；不得出現在常見密碼清單；不得與 username、email 過於相似；不得全為數字。
- 不強制混用大小寫或符號（依 NIST SP 800-63B 建議）。
- 密碼變更或重設後，撤銷該帳號所有 session。

### 1.5 登入防暴力破解
- **依識別值計數**：以正規化後的 email / 手機 / username 為鍵計數（不論帳號是否存在，避免藉此探測帳號）。15 分鐘內失敗 5 次即鎖定 15 分鐘；之後每次再被鎖定，時間加倍，最長 24 小時。
- **依 IP 計數**：10 分鐘內失敗 20 次，該 IP 的登入請求封鎖 30 分鐘。
- 鎖定期間一律回傳 `TOO_MANY_ATTEMPTS`，不透露帳號是否存在。

### 1.6 個人資料變更
- username 每 30 天可變更一次。
- 變更 email / 手機號：寄驗證碼到新的 email / 手機號，驗證成功才生效，並通知舊的 email / 手機號。

## 2. 會員

### 2.1 等級與方案
- 每個等級有 `rank`（數字越大等級越高），用來判斷升級或降級。
- Free 是唯一的預設等級：不可停用、沒有付費方案、不發放會員點數（若要贈送點數給 Free 會員，用促銷點數）。
- 付費等級各有月費與年費兩個方案；每個方案對應一個商品，價格存在商品價格表。
- 停用等級：不能再購買；現有會員照常到期。停用方案（下架商品）：自動續費到期時扣款失敗，依 §2.7 處理，並提前通知。

### 2.2 購買情境

| 目前狀態 | 選擇的方案 | 結果 |
| --- | --- | --- |
| Free | 任一付費方案 | 購買，付款後立即生效 |
| 付費等級 A | A 的月費或年費 | 續費：新期間接在最後一期結束之後 |
| 付費等級 A | 更高等級 B | 升級（§2.4） |
| 付費等級 A | 更低等級 C | 排定降級（§2.5），不建立訂單 |
| 寬限期中 | — | 只能完成待付款的續費訂單，或等它到期 |

### 2.3 期間計算
- 月費期間 = 起始時間加 1 個日曆月；年費期間 = 加 12 個日曆月。起始日在下個月不存在時（例如 1 月 31 日），取該月最後一天。
- 所有時間以 UTC 計算。

### 2.4 升級（D-04）
- **應付金額** = 新方案價格 − 抵扣額。
- **抵扣額** = 目前期間的剩餘價值 + 所有已預付、尚未開始期間的完整價值。
  - 剩餘價值 = 目前期間的價值 × 剩餘秒數 ÷ 期間總秒數，無條件捨去到 USDT 最小單位。
  - 期間的價值：一般購買或續費 = 該期間的實付金額；由升級產生的期間 = 新方案的價格。
- **新期間**：從升級當下起算一個新方案期間。被抵扣的期間標為 `SUPERSEDED`。
- **會員點數**：目前週期未用完的會員點數收回；發放新等級本週期的配額。
- **自動續費設定**沿用。
- **不允許**：應付金額 ≤ 0 時拒絕（`UPGRADE_NOT_ALLOWED`），提示選擇較長期間的方案。
- **推薦獎勵**：以實付金額判斷是否合格。

**範例**：1 月 1 日 00:00 以 10 USDT 購買 Standard 月費，1 月 16 日 00:00 升級到 VIP 月費（30 USDT）。
剩餘價值 = 10 × 16 ÷ 31 = 5.161290 USDT；應付 = 30 − 5.161290 = 24.838710 USDT；新期間為 1 月 16 日至 2 月 16 日。

### 2.5 降級
- 降級只是「排定下一期的方案」，記在最後一期會員記錄的 `next_plan_id`，不建立訂單、不退差價。
- 開啟自動續費時，下一次續費以降級後的方案計價；沒開自動續費時，使用者到期後自行購買。
- 生效前可取消。

### 2.6 到期
- 到期時若沒有已預付的下一期，也沒有進行中的自動續費，會員轉為 `EXPIRED`，使用者回到 Free。
- 有已預付的下一期時，下一期直接轉為 `ACTIVE`。

### 2.7 自動續費（D-03）

| 時間點 | 動作 |
| --- | --- |
| 到期前 7 天、1 天 | 若餘額不足以支付下一期，通知儲值 |
| 到期前 24 小時 | 建立續費訂單並從餘額扣款；成功即建立下一期（`SCHEDULED`）。失敗則每 6 小時重試 |
| 到期時仍未付款 | 會員轉 `GRACE`：保留權益，寬限期 3 天，每日重試 |
| 寬限期結束仍未付款 | 會員 `EXPIRED`、續費訂單 `EXPIRED`、使用者回到 Free，通知使用者 |

- 續費成功的新期間，一律接在原到期時間之後；在寬限期內付款也一樣，不補寬限的天數。
- **價格**：以續費當下生效的商品價格計算。有自動續費會員的方案要調價時，生效時間至少在 7 天之後，系統在排定時通知受影響的會員。
- 使用者關閉自動續費時，尚未付款的續費訂單轉 `CANCELLED`。
- 停權中的帳號不自動續費。

### 2.8 會員點數
- 每個權益週期為 1 個月，從會員期間開始時間起算；年費方案共 12 個週期。
- 每個週期開始時發放該等級的 `ai.credits.per_cycle` 點數，到期時間 = 週期結束時間。
- 發放以冪等鍵 `membership:{membership_id}:cycle:{n}` 防止重複。
- 寬限期內不發放新週期的點數；續費付款成功後補發。

### 2.9 權益判斷
- 使用者目前的等級 = 狀態為 `ACTIVE` 或 `GRACE` 的會員記錄的等級；沒有就是 Free。
- 權益鍵在程式中註冊型別與預設值；等級沒設定某個鍵時使用預設值（預設值一律是最保守的，例如「不允許」或 0）。
- 每日用量依營運時區的日期計數（§9）。

## 3. AI 點數

### 3.1 類型與來源

| 類型 | 來源 | 到期（D-05 待定，以下為目前設計） |
| --- | --- | --- |
| `PROMOTIONAL` | 後台發放、未來的行銷活動 | 發放時指定；未指定則不過期 |
| `MEMBERSHIP` | 會員權益週期 | 週期結束時 |
| `ADMIN` | 後台調整增加 | 調整時指定；未指定則不過期 |
| `PURCHASED` | 點數包訂單 | 依設定 `wallet.purchased_credit_ttl_days`，預設不過期 |

### 3.2 扣點
- 預設優先序：`PROMOTIONAL` → `MEMBERSHIP` → `ADMIN` → `PURCHASED`（付費買的點數最後扣）。同一類型中，先到期的先扣，再依入帳時間先後。
- 點數為整數；可用餘額不足時整筆拒絕（`INSUFFICIENT_CREDITS`），不做部分扣點。
- 每筆扣點都要帶冪等鍵。

### 3.3 點數退回（沖正）
- 只能針對某一筆扣點交易，累計退回量不超過原扣點量。
- 點數退回原本扣的批次；原批次已到期時，改為新增一個同類型批次，7 天後到期。

### 3.4 後台調整
- 增加：建立 `ADMIN` 類型批次。扣減：依 §3.2 的優先序扣。
- 必須填寫原因（至少 10 個字）與參考編號（例如客服工單號），並寫入稽核日誌。

## 4. 訂單與付款

- 本期每張訂單只有一個品項；會員方案數量固定為 1，點數包數量 1–100。
- 價格由伺服器依商品價格表計算，訂單明細保存價格快照；幣別一律 USDT。
- **訂單編號**：`PL` + UTC 日期（`YYYYMMDD`）+ `-` + 6 個隨機字元（與推薦碼相同的字元集）。
- **付款期限**：一般訂單 30 分鐘；自動續費訂單到寬限期結束為止。
- **取消**：只有 `CREATED`、`PENDING_PAYMENT` 可取消。
- **付款**：以站內 USDT 餘額一次付清，不做部分付款。餘額不足時回傳差額與入金地址。
- **入金後自動付款**：入金入帳後，依建立時間由舊到新檢查使用者的待付款訂單，餘額足夠就付款；每張訂單各自一個交易。

## 5. 退款（D-06、D-18）

- **誰可以退款**：只有管理員，需要 `refund.manage` 權限；一位管理員即可完成，必須填寫原因與參考編號（例如客服工單號）。
- **退到哪裡**：一律退回使用者的站內 USDT 餘額，不在鏈上轉帳。使用者要把錢轉出平台，要等未來的出金功能（D-14）。
- **可退款的訂單**：狀態為 `COMPLETED` 或 `FAILED`，且已付款成功；每張訂單本期只能退款一次。
- **金額**：由管理員決定，不超過實付金額。後台會顯示建議金額（點數包 = 實付 × 剩餘點數 ÷ 原始點數，無條件捨去）。
- **時效**：退款、退回餘額與以下的反向處理，在同一個資料庫交易內完成。
- **各商品類型的反向處理**：

  | 商品類型 | 反向處理 |
  | --- | --- |
  | 會員方案（購買、續費） | 該訂單產生的會員期間轉 `REVOKED`，立即失效；該期間發放、尚未用完的會員點數收回 |
  | 會員升級 | 升級產生的期間轉 `REVOKED`；被抵扣的舊期間不恢復（管理員決定金額時應考慮） |
  | 點數包 | 收回該批次目前剩餘的點數；已使用的不收回 |

- **推薦獎勵**：`PENDING` / `ON_HOLD` 的轉 `CANCELLED`；`CONFIRMED` 的按「退款金額 ÷ 實付金額」比例新增反向記錄（全額退款即全額沖回）。

## 6. 推薦

### 6.1 推薦碼
- 字元集：`23456789ABCDEFGHJKMNPQRSTUVWXYZ`（31 個，去掉容易混淆的 0、O、1、I、L）。
- 預設長度 8、無前綴；設定前綴時，隨機部分至少 6 個字元。
- 輸入時去除空白與連字號並轉大寫後再比對。
- 使用者每 30 天可重新產生一次推薦碼；舊碼停用，已建立的推薦關係不受影響。
- 推薦碼被管理員停用時，使用者不能自行重新產生，需由管理員恢復。

### 6.2 綁定條件（驗證成功當下檢查）

| 檢查 | 不符合時 |
| --- | --- |
| 推薦碼存在且為有效狀態 | 不綁定，記錄 `CODE_INVALID` |
| 推薦人帳號為 `ACTIVE`，且推薦資格未被停用 | 不綁定，記錄 `REFERRER_INELIGIBLE` |
| 推薦人不是被推薦人自己 | 不綁定，記錄 `SELF_REFERRAL` |
| 被推薦人尚未綁定 | 不綁定，記錄 `ALREADY_BOUND` |
| 被推薦人不在推薦人的上線鏈中（不會形成循環） | 不綁定，記錄 `CIRCULAR` |

不綁定不會讓驗證失敗；結果記在 `referral_bind_attempts`。
綁定成功時一併記錄被推薦人往上最多 10 層的上線（上線鏈），之後不可修改。

### 6.3 獎勵
- 規則與計算方式見 `REFERRAL-FLOW.md`。
- 層數由管理員設定（`referral.reward_levels`，1 ~ 10，預設 1）：購買者往上第 1 ~ N 層的上線都可以得到獎勵，第 1 層是直接推薦人（D-19）。設為 2 以上之前需法務確認。
- 每條規則分層設定獎勵：每一層各自是固定金額或百分比，可設單筆上限；規則沒設定的層不發。
- 某一層的上線不符資格（帳號不是 `ACTIVE`，或推薦資格被停用）時，這一層不發，也不往上遞補。
- 同一張訂單各層獎勵合計不超過訂單實付金額。
- 修改層數只影響之後完成的訂單。
- 本期沒有預設的獎勵規則，由營運管理員建立。
- 獎勵以 USDT 計價，只記帳、不提領（D-02）。
- 自動續費訂單也是合格訂單（D-07）。

### 6.4 結算
- 獎勵在 `結算時間 = 訂單完成時間 + 規則的結算延遲天數` 之後，且訂單沒有在退款中或已退款，才轉 `CONFIRMED`。

### 6.5 風控
- 購買者的註冊 IP 或裝置指紋，與受益的上線最近 30 天的登入記錄相同時，該層獎勵自動轉 `ON_HOLD`，由財務審核；每一層分別比對。
- 同一推薦人 24 小時內新增的綁定數超過門檻（預設 20）時告警。

## 7. USDT 入金

- 鏈上只用於入金（與未來的出金）；付款、退款、獎勵都在平台帳本內完成（D-18）。
- 支援 TRON TRC-20 USDT 與 BNB Chain BEP-20 USDT；兩條鏈的入金都計入同一個站內 USDT 餘額。
- 每位使用者每條鏈一個固定的入金地址，不會變更。
- 金額為 0 的轉帳（常見於地址投毒攻擊）直接忽略；金額大於 0 的入金不論多少都入帳到使用者餘額。不設最低入金額，以免需要在鏈上退回。
- **確認數**：TRON 預設 20 個區塊；BNB Chain 預設 15 個區塊。
- **小數位**：BEP-20 USDT 鏈上為 18 位小數，換算為站內 6 位時無條件捨去，捨去部分記錄在入金記錄。
- 轉入已停用（`DISABLED`）帳號的入金地址：照常入帳到該帳號的餘額，與該帳號的其他餘額一起由財務依條款處理。
- 風險篩查命中的入金：記到暫記帳戶，由財務決定入帳給使用者或沖銷（沖銷需第二位管理員核准）；不在鏈上退回。
- 單筆入金達 10,000 USDT 時，需第二個 RPC 來源確認後才入帳。

## 8. 捨入規則

| 計算 | 規則 |
| --- | --- |
| 升級的剩餘價值 | 無條件捨去到 0.000001 USDT |
| 百分比推薦獎勵 | 無條件捨去到 0.000001 USDT |
| 退款的獎勵沖回比例 | 無條件捨去；全額退款時等於原獎勵 |
| 點數包建議退款金額 | 無條件捨去 |
| BEP-20 鏈上數量換算 | 無條件捨去到 6 位小數 |
| 點數 | 一律整數，不產生小數 |

## 9. 時間與日界
- 所有時間以 UTC 儲存。
- 「每日」的計算（每日用量上限、每日轉出上限、報表日界）以 `business_timezone` 的 00:00 為界。D-09 定案前預設為 UTC。

## 10. 系統設定註冊表

下表的設定都可在後台調整，每次修改都寫入稽核日誌。「權限」欄是修改所需的權限。
金額單位為 USDT。標「待財務確認」的數值在上線前由財務決定（D-16）。

| 鍵 | 預設值 | 權限 | 說明 |
| --- | --- | --- | --- |
| `business_timezone` | `UTC` | `settings.manage` | 營運時區（D-09） |
| `auth.access_token_ttl_seconds` | 600 | `settings.manage` | 會員 access token 有效期 |
| `auth.refresh_token_idle_days` | 30 | `settings.manage` | refresh token 閒置期限 |
| `auth.refresh_token_absolute_days` | 90 | `settings.manage` | session 絕對期限 |
| `auth.admin_access_token_ttl_seconds` | 300 | `rbac.manage` | 後台 access token 有效期 |
| `auth.admin_session_idle_minutes` | 30 | `rbac.manage` | 後台 session 閒置期限 |
| `auth.admin_session_absolute_hours` | 12 | `rbac.manage` | 後台 session 絕對期限 |
| `auth.login_lock_threshold` | 5 | `settings.manage` | 鎖定前允許的失敗次數 |
| `auth.login_lock_window_minutes` | 15 | `settings.manage` | 失敗計數的時間窗 |
| `auth.login_lock_base_minutes` | 15 | `settings.manage` | 第一次鎖定時間 |
| `auth.login_lock_max_hours` | 24 | `settings.manage` | 最長鎖定時間 |
| `auth.pending_account_ttl_hours` | 72 | `settings.manage` | 待驗證帳號保留時間 |
| `auth.username_change_cooldown_days` | 30 | `settings.manage` | username 變更間隔 |
| `verification.code_ttl_minutes` | 10 | `settings.manage` | 驗證碼有效期 |
| `verification.max_attempts` | 5 | `settings.manage` | 驗證碼最大嘗試次數 |
| `verification.resend_cooldown_seconds` | 60 | `settings.manage` | 重送冷卻 |
| `verification.per_target_per_hour` | 5 | `settings.manage` | 每個 email / 手機每小時發送上限 |
| `verification.per_ip_per_hour` | 20 | `settings.manage` | 每個 IP 每小時發送上限 |
| `verification.sms_daily_cap_by_country` | `{}` | `settings.manage` | 各國碼每日簡訊上限 |
| `referral.code_length` | 8 | `settings.manage` | 推薦碼隨機部分長度 |
| `referral.code_prefix` | `""` | `settings.manage` | 推薦碼前綴 |
| `referral.regenerate_cooldown_days` | 30 | `settings.manage` | 重新產生推薦碼的間隔 |
| `referral.risk_daily_binding_alert` | 20 | `settings.manage` | 單一推薦人每日綁定數告警門檻 |
| `referral.reward_levels` | 1 | `referral.rule.manage` | 推薦獎勵層數（1 ~ 10）；設為 2 以上前需法務確認（D-19） |
| `membership.renewal_lead_hours` | 24 | `settings.manage` | 到期前多久扣款 |
| `membership.renewal_retry_hours` | 6 | `settings.manage` | 到期前的重試間隔 |
| `membership.grace_days` | 3 | `settings.manage` | 寬限期天數 |
| `membership.reminder_days` | `[7, 1]` | `settings.manage` | 到期提醒時間點 |
| `membership.price_change_notice_days` | 7 | `settings.manage` | 調價對自動續費會員的最短通知期 |
| `wallet.consumption_priority` | `["PROMOTIONAL","MEMBERSHIP","ADMIN","PURCHASED"]` | `settings.manage` | 扣點優先序 |
| `wallet.purchased_credit_ttl_days` | `null` | `settings.manage` | 購買點數有效天數（D-05） |
| `wallet.reversal_new_lot_ttl_days` | 7 | `settings.manage` | 原批次已過期時，退回點數的新批次有效天數 |
| `wallet.expiring_soon_days` | 7 | `settings.manage` | 「即將到期」的顯示範圍 |
| `order.payment_timeout_minutes` | 30 | `settings.manage` | 一般訂單付款期限 |
| `order.auto_pay_on_deposit` | `true` | `settings.manage` | 入金後自動付款 |
| `order.fulfillment_max_attempts` | 5 | `settings.manage` | 履約重試上限 |
| `chain.tron.confirmations` | 20 | `treasury.limits` | TRON 確認數 |
| `chain.bsc.confirmations` | 15 | `treasury.limits` | BNB Chain 確認數 |
| `chain.large_deposit_amount` | 10,000 | `treasury.limits` | 需雙來源確認的入金金額 |
| `chain.sweep_threshold` | 50 | `treasury.limits` | 入金地址歸集門檻 |
| `chain.hot_wallet_low_watermark` | 待財務確認 | `treasury.limits` | 熱錢包手續費資產（TRX、BNB）的低水位 |
| `chain.payout_max_per_tx` | 待財務確認 | `treasury.limits` | 系統發起的單筆轉帳上限（本期為補手續費；簽章服務另有不可由後台修改的上限） |
| `chain.payout_max_daily` | 待財務確認 | `treasury.limits` | 系統發起的每日轉帳上限 |
| `chain.payouts_enabled` | `true` | `treasury.limits` | 系統發起的鏈上轉帳總開關（緊急時關閉；只會延後歸集） |
| `reporting.export_ttl_hours` | 24 | `settings.manage` | 匯出檔下載期限 |
| `ratelimit.*` | 見 `SECURITY-DESIGN.md` §5 | `settings.manage` | 各級限流 |

## 11. 尚待決定

| 決策 | 影響 | 目前作法 |
| --- | --- | --- |
| D-05 點數到期 | §3.1 | 會員點數週期結束到期，其他類型預設不過期 |
| D-09 營運時區 | §9 | UTC |
| D-14 出金 | §1.3 停用帳號的餘額、§5 退款 | 本期不做；未來每筆人工審核、向使用者收固定手續費 |
| D-16 鏈上營運數值 | §10 標「待財務確認」的項目 | 上線前填入 |
| D-17 法遵 | §5、§6.3、§7 | 預留風險篩查掛勾；推薦獎勵層數維持 1，等法務確認多層獎勵的適法性（D-19） |
