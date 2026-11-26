---
title: Pinklin Phase 1 — Wallet & Ledger Design
type: DESIGN
status: draft            # draft → approved
owner: jackylau
prd: docs/project/PRD-project.md
architecture: docs/ARCHITECTURE.md
created: 2026-11-26
updated: 2026-11-26
---

# 點數錢包與資金帳本

系統有兩本完全分開的帳：

| | 第一部分：AI 點數錢包 | 第二部分：USDT 資金帳本 |
| --- | --- | --- |
| 模組 | `wallet` | `funds` |
| 記的是什麼 | 平台消費單位，不是錢 | 使用者儲值的 USDT 與平台的鏈上資產 |
| 記帳方式 | 批次 + 單式帳本（每筆記錄異動前後餘額） | 複式記帳（每筆分錄借貸平衡） |
| 相關需求 | REQ-080 ~ REQ-094 | REQ-117、REQ-173、REQ-181、REQ-186 |
| 資料表 | `DATABASE-ERD.md` §7 | `DATABASE-ERD.md` §8 |

兩者共同的原則：
- 記錄只能新增，不能修改或刪除；更正一律新增反向記錄。
- 每次異動都帶冪等鍵；同一個鍵重送時回傳原結果。
- 只有該模組的 `public.py` 可以異動餘額；後台調整也必須經過它。

---

# 第一部分：AI 點數錢包

## A1. 名詞

| 名詞 | 資料表 | 說明 |
| --- | --- | --- |
| 錢包 | `credit_wallets` | 每位使用者一個；`balance` 是所有有效批次剩餘量的快取 |
| 批次 | `credit_lots` | 每次入帳產生一個批次，有類型、剩餘量、到期時間 |
| 帳本 | `credit_ledger` | 每筆異動一列，記錄點數、異動前後餘額與關聯資料（即 spec 的交易記錄） |
| 批次異動 | `credit_lot_movements` | 一筆帳本記錄影響了哪些批次、各多少 |

## A2. 帳目類型

| 類型 | 點數 | 對批次的影響 | 觸發來源 | 冪等鍵格式 |
| --- | --- | --- | --- | --- |
| `GRANT` | + | 建立新批次 | 點數包履約、會員週期發放、促銷發放 | `order:{order_item_id}:grant`、`membership:{membership_id}:cycle:{n}`、`promo:{request_id}` |
| `CONSUME` | − | 依優先序扣多個批次 | 未來 AI 生成等消費者 | 由呼叫方提供，例如 `job:{generation_job_id}:consume` |
| `CONSUME_REVERSAL` | + | 退回原批次；原批次已到期則建立新批次 | 消費失敗沖正 | `reversal:{原帳本 id}:{呼叫方的鍵}` |
| `EXPIRE` | − | 該批次歸零，狀態 `EXPIRED` | 到期處理 | `expire:{lot_id}` |
| `REVOKE` | − | 指定批次的剩餘量歸零，狀態 `REVOKED` | 退款扣回、升級收回舊週期會員點數 | `revoke:{原因代碼}:{參考 id}:{lot_id}` |
| `ADJUST_IN` | + | 建立 `ADMIN` 類型批次 | 後台調整 | `admin:{request_id}` |
| `ADJUST_OUT` | − | 依優先序扣 | 後台調整 | `admin:{request_id}` |

## A3. 公開介面（`wallet.public`）

```python
grant(user_id, amount, credit_type, source_type, source_ref, expires_at,
      idempotency_key, actor, order_id=None) -> LedgerEntry
consume(user_id, amount, idempotency_key, actor, generation_job_id=None) -> LedgerEntry
reverse_consumption(ledger_id, amount, idempotency_key, actor) -> LedgerEntry
revoke_by_source(user_id, source_type, source_ref, reason_code, reference_id, actor) -> list[LedgerEntry]
adjust(user_id, direction, amount, reason, reference, idempotency_key, actor) -> LedgerEntry
get_balance(user_id) -> Balance        # 總額、各類型、即將到期的批次
list_transactions(user_id, filters, cursor) -> Page[LedgerEntry]
ensure_wallet(user_id) -> None         # user.activated 時呼叫，可重複執行
```

- 同一個冪等鍵、相同參數 → 回傳原本的帳本記錄。
- 同一個冪等鍵、不同參數 → 拋出 `IdempotencyConflict`（API 回 `IDEMPOTENCY_KEY_REUSED`）。

## A4. 扣點演算法

```text
consume(user, amount, key):
  BEGIN
    w = SELECT * FROM credit_wallets WHERE user_id = :user FOR UPDATE
    若 credit_ledger 已有 (w.id, key) → 驗證參數相同後回傳該記錄；結束
    expire_due_lots(w)                                   # 先處理已到期的批次（A6）
    lots = SELECT * FROM credit_lots
           WHERE wallet_id = w.id AND status = 'ACTIVE' AND remaining_amount > 0
           ORDER BY 類型在 wallet.consumption_priority 中的位置,
                    expires_at NULLS LAST, created_at
    若 sum(lots.remaining_amount) < amount → ROLLBACK，回傳 INSUFFICIENT_CREDITS
    依序從 lots 扣，直到扣滿 amount；
      每個被扣的批次：remaining -= x；歸零則 status = 'DEPLETED'
    寫入 credit_ledger（CONSUME，amount = -amount，balance_before = w.balance，balance_after = w.balance - amount）
    每個被扣的批次寫一列 credit_lot_movements（amount = -x）
    UPDATE credit_wallets SET balance = balance - amount, version = version + 1
  COMMIT
```

- 只鎖錢包這一列；批次不另外加鎖，因為同一錢包的所有異動都會先拿到錢包鎖。
- 一個交易只處理一個錢包，避免死結。批次發放促銷點數時，每個錢包各自一個交易。

## A5. 點數退回

```text
reverse_consumption(original_id, amount, key):
  鎖錢包
  驗證 original 是 CONSUME，且 已退回總量 + amount ≤ |original.amount|
  依 original 的 movements，從最後扣的批次開始往回退（後進先出）：
    批次仍有效 → remaining += x；若原為 DEPLETED 則改回 ACTIVE
    批次已到期或已收回 → 建立同類型新批次（source_type = REVERSAL），
                          到期時間 = 現在 + wallet.reversal_new_lot_ttl_days
  寫入 CONSUME_REVERSAL 帳本記錄（reversal_of_id = original_id）與 movements
  更新錢包餘額
```

## A6. 到期

- **惰性處理**：每次異動前，先把該錢包 `expires_at <= now()` 且仍為 `ACTIVE` 的批次逐一做到期處理。
- **排程處理**：每 10 分鐘掃描 `credit_lots` 中已到期的有效批次，依錢包分組，每個錢包一個交易處理。
- **每個到期批次各寫一筆** `EXPIRE` 帳本記錄（冪等鍵 `expire:{lot_id}`），批次 `remaining = 0`、`status = 'EXPIRED'`。
- **查詢餘額時**只計算未到期的批次，所以排程還沒跑到時，使用者也不會看到已到期的點數。

## A7. 後台調整

- `ADJUST_IN`：建立 `ADMIN` 批次，到期時間由管理員指定或不過期。
- `ADJUST_OUT`：依扣點優先序扣；不足時拒絕。
- 兩者都必須帶原因（至少 10 個字）與參考編號，並與 `admin_audit_logs` 在同一個交易寫入。

## A8. 不變式與對帳

每日對帳（`reconciliation_runs.kind = 'CREDITS'`）執行以下檢查，任何一項有差異就標記 `MISMATCH` 並告警，**不自動修正**：

| # | 不變式 |
| --- | --- |
| I1 | 錢包餘額 = 該錢包 `ACTIVE` 批次的剩餘量總和 |
| I2 | 錢包餘額 = 該錢包所有帳本記錄的點數總和 |
| I3 | 每個批次的剩餘量 = 該批次所有 movements 的總和 |
| I4 | 每筆帳本記錄的點數 = 其 movements 的總和 |
| I5 | 每個錢包最後一筆帳本記錄的 `balance_after` = 錢包餘額 |

```sql
-- I1、I2
SELECT w.id, w.balance,
       COALESCE((SELECT sum(remaining_amount) FROM credit_lots l
                 WHERE l.wallet_id = w.id AND l.status = 'ACTIVE'), 0) AS lots_sum,
       COALESCE((SELECT sum(amount) FROM credit_ledger e WHERE e.wallet_id = w.id), 0) AS ledger_sum
FROM credit_wallets w
WHERE w.balance <> COALESCE((SELECT sum(remaining_amount) FROM credit_lots l
                             WHERE l.wallet_id = w.id AND l.status = 'ACTIVE'), 0)
   OR w.balance <> COALESCE((SELECT sum(amount) FROM credit_ledger e WHERE e.wallet_id = w.id), 0);

-- I3
SELECT l.id, l.remaining_amount, sum(m.amount) AS movements_sum
FROM credit_lots l JOIN credit_lot_movements m ON m.lot_id = l.id
GROUP BY l.id HAVING l.remaining_amount <> sum(m.amount);

-- I4
SELECT e.id, e.amount, sum(m.amount) AS movements_sum
FROM credit_ledger e JOIN credit_lot_movements m ON m.ledger_id = e.id
GROUP BY e.id HAVING e.amount <> sum(m.amount);
```

## A9. 範例

使用者有 100 點促銷點數（7 天後到期）與 1,000 點購買點數，扣 150 點：

| 帳本 | 類型 | 點數 | 前 | 後 | movements |
| --- | --- | --- | --- | --- | --- |
| L1 | `GRANT`（PURCHASED） | +1000 | 0 | 1000 | 批次 P +1000 |
| L2 | `GRANT`（PROMOTIONAL） | +100 | 1000 | 1100 | 批次 R +100 |
| L3 | `CONSUME` | −150 | 1100 | 950 | 批次 R −100（DEPLETED）、批次 P −50 |
| L4 | `CONSUME_REVERSAL`（退 150） | +150 | 950 | 1100 | 批次 P +50、批次 R +100（ACTIVE） |

---

# 第二部分：USDT 資金帳本

## B1. 為什麼用複式記帳

平台代管使用者的 USDT，必須隨時能證明：使用者餘額、平台營收、已轉出的錢，加總起來等於平台在鏈上實際持有的資產。
複式記帳讓「每一筆錢從哪裡來、到哪裡去」都成對記錄，帳本整體自動平衡，鏈上對帳只要比對資產帳戶即可。

## B2. 帳戶表

| 類別 | 類型 | 正常方向 | 幣別 | 粒度 | 說明 |
| --- | --- | --- | --- | --- | --- |
| `USER_BALANCE` | 負債 | 貸 | USDT | 每位使用者 | 使用者的站內餘額；不得為負 |
| `DEPOSIT_POOL` | 資產 | 借 | USDT、TRX、BNB | 每條鏈 × 幣別 | 所有入金地址上的資產總和（含補進去的手續費） |
| `HOT_WALLET` | 資產 | 借 | TRX、BNB（未來出金時加上 USDT） | 每條鏈 × 幣別 | 熱錢包上的資產；本期只放手續費 |
| `COLD_WALLET` | 資產 | 借 | USDT、TRX、BNB | 每條鏈 × 幣別 | 冷錢包上的資產 |
| `STAKED` | 資產 | 借 | TRX | TRON | 為取得能量而質押的 TRX |
| `SALES_REVENUE` | 收入 | 貸 | USDT | 全平台 | 銷售收入 |
| `SALES_REFUNDS` | 收入的減項 | 借 | USDT | 全平台 | 退款 |
| `NETWORK_FEE` | 費用 | 借 | TRX、BNB | 每個幣別 | 鏈上手續費 |
| `SUSPENSE` | 負債 | 貸 | USDT | 全平台 | 異常入金，待人工處理 |
| `PLATFORM_CAPITAL` | 權益 | 貸 | USDT、TRX、BNB | 每個幣別 | 平台自有資金的投入與提出 |

- `balance` 欄位以帳戶的正常方向記錄：資產與費用帳戶 = 借方合計 − 貸方合計；負債、權益、收入帳戶 = 貸方合計 − 借方合計。
- 本帳本不做期末結帳；收入結轉、稅務由會計系統處理。

## B3. 分錄範本

| 分錄類型 | 借方 | 貸方 | 觸發時機 |
| --- | --- | --- | --- |
| `DEPOSIT` | `DEPOSIT_POOL`（鏈, USDT） | `USER_BALANCE`（使用者） | 入金確認 |
| `DEPOSIT_TO_SUSPENSE` | `DEPOSIT_POOL`（鏈, USDT） | `SUSPENSE` | 異常入金確認 |
| `SUSPENSE_RESOLVE` | `SUSPENSE` | `USER_BALANCE`（入帳給使用者）或 `PLATFORM_CAPITAL`（沖銷，需雙人核准） | 財務處理異常入金；不在鏈上退回 |
| `PAYMENT` | `USER_BALANCE`（使用者） | `SALES_REVENUE` | 以餘額付款 |
| `REFUND` | `SALES_REFUNDS` | `USER_BALANCE`（使用者） | 管理員退款，退回站內餘額 |
| `SWEEP` | `COLD_WALLET`（鏈, USDT） | `DEPOSIT_POOL`（鏈, USDT） | 歸集在鏈上確認 |
| `GAS_TOPUP` | `DEPOSIT_POOL`（鏈, TRX/BNB） | `HOT_WALLET`（鏈, TRX/BNB） | 補手續費在鏈上確認 |
| `NETWORK_FEE` | `NETWORK_FEE`（TRX/BNB） | 付手續費的地址所屬的帳戶（鏈, TRX/BNB） | 任何轉出確認時，依實際消耗記錄 |
| `CAPITAL_IN` | `HOT_WALLET` 或 `STAKED` | `PLATFORM_CAPITAL` | 平台投入資金（例如購買 TRX、BNB 作為手續費） |
| `CAPITAL_OUT` | `PLATFORM_CAPITAL` | `COLD_WALLET` | 平台從冷錢包提出營收（需雙人核准） |
| `ADJUSTMENT` | 任意 | 任意 | 錯誤更正；只有 Super Administrator 可發起，需雙人核准與原因 |

一筆分錄可以有多組借貸，例如歸集確認時，同一筆分錄同時記 USDT 的歸集與 TRX 的手續費（每個幣別各自平衡）。

未來的出金模組會加上 `WITHDRAWAL`（借 `USER_BALANCE`、貸 `HOT_WALLET`）與熱錢包、冷錢包之間的 USDT 調度分錄。

## B4. 記帳流程（`funds.public.post`）

```text
post(entry_type, reference, lines, idempotency_key, actor):
  # 由呼叫方的交易包住；本函式不開新交易
  若 fund_journal_entries 已有 idempotency_key → 驗證相同後回傳原分錄
  檢查 lines：每個幣別借貸合計相等；金額 > 0；帳戶幣別相符
  依 account_id 排序，逐一 SELECT ... FOR UPDATE 鎖住相關帳戶
  寫入 fund_journal_entries 與 fund_postings
  依正常方向更新每個帳戶的 balance（USER_BALANCE 變負數時，CHECK 約束讓整個交易失敗）
  # 提交時，延遲約束觸發器 assert_entry_balanced() 再檢查一次
```

公開介面：

```python
ensure_user_account(user_id) -> None
post(entry_type, reference_type, reference_id, lines, idempotency_key, actor) -> JournalEntry
charge_user(user_id, amount, reference_type, reference_id, idempotency_key, actor) -> JournalEntry
    # PAYMENT 的捷徑；餘額不足拋出 InsufficientBalance
get_user_balance(user_id) -> Money
list_user_activity(user_id, filters, cursor) -> Page[Activity]
    # 使用者看到的紀錄：入金（含確認中）、付款、退款
get_account_balances(category=None, chain=None, currency=None) -> list[AccountBalance]   # 後台與對帳
```

## B5. 不變式與對帳

| # | 不變式 | 檢查方式 |
| --- | --- | --- |
| F1 | 每筆分錄每個幣別借貸平衡 | 資料庫延遲約束觸發器（即時）＋每日 SQL |
| F2 | 帳戶 `balance` = 該帳戶分錄明細依正常方向的合計 | 每日 SQL |
| F3 | 使用者餘額 ≥ 0 | `CHECK` 約束（即時） |
| F4 | 每個幣別：資產 + 費用 + 收入減項 = 負債 + 權益 + 收入 | 每日 SQL（F1 成立時必然成立，作為額外保險） |
| F5 | 每條鏈每個幣別：`DEPOSIT_POOL` + `HOT_WALLET` + `COLD_WALLET`（+ `STAKED`）= 鏈上實際餘額（扣除轉出中的金額） | 每日鏈上對帳，見 `CHAIN-CUSTODY.md` §8 |

```sql
-- F1
SELECT entry_id, currency
FROM fund_postings
GROUP BY entry_id, currency
HAVING sum(CASE direction WHEN 'DEBIT' THEN amount ELSE -amount END) <> 0;

-- F2
SELECT a.id, a.category, a.balance,
       sum(CASE WHEN p.direction = a.normal_side THEN p.amount ELSE -p.amount END) AS computed
FROM fund_accounts a JOIN fund_postings p ON p.account_id = a.id
GROUP BY a.id
HAVING a.balance <> sum(CASE WHEN p.direction = a.normal_side THEN p.amount ELSE -p.amount END);

-- F4（每個幣別）
SELECT currency,
       sum(CASE WHEN normal_side = 'DEBIT'  THEN balance ELSE 0 END) AS debit_side,
       sum(CASE WHEN normal_side = 'CREDIT' THEN balance ELSE 0 END) AS credit_side
FROM fund_accounts
GROUP BY currency
HAVING sum(CASE WHEN normal_side = 'DEBIT' THEN balance ELSE -balance END) <> 0;
```

## B6. 範例

使用者 U 在 TRON 入金 100 USDT，購買 30 USDT 的會員；之後入金地址被歸集到冷錢包；最後管理員退款 10 USDT。
（金額以 USDT 表示，實際儲存為最小單位。）

| # | 分錄 | 借方 | 貸方 |
| --- | --- | --- | --- |
| 1 | `DEPOSIT` | DEPOSIT_POOL(TRON, USDT) 100 | USER_BALANCE(U) 100 |
| 2 | `PAYMENT` | USER_BALANCE(U) 30 | SALES_REVENUE 30 |
| 3 | `GAS_TOPUP` | DEPOSIT_POOL(TRON, TRX) 15 | HOT_WALLET(TRON, TRX) 15 |
| 4 | `SWEEP` | COLD_WALLET(TRON, USDT) 100 | DEPOSIT_POOL(TRON, USDT) 100 |
| 4 | （同一筆）手續費 | NETWORK_FEE(TRX) 13.5 | DEPOSIT_POOL(TRON, TRX) 13.5 |
| 5 | `REFUND` | SALES_REFUNDS 10 | USER_BALANCE(U) 10 |

結果：U 的餘額 80 USDT；營收 30、退款 10；TRON 冷錢包 USDT 100；TRON 熱錢包 TRX 減少 15。
退款完全在帳本內完成，沒有鏈上轉帳。兩條鏈的 USDT 對使用者來說是同一個餘額。
