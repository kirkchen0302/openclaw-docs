# 新增測試帳號 SOP

> 最後更新：2026-03-26
> 適用系統：發票存摺 Demo（Firebase RTDB + BigQuery + GitHub Pages）

---

## 概覽

新增測試帳號需要完成三件事：

1. **BigQuery** → 查詢該電話號碼的 member_id 和近 6 個月發票資料
2. **資料處理** → 統編轉店名、計算訂閱/任務/趨勢
3. **Firebase RTDB** → PUT 上傳至 `users/{phone_number}`

---

## 前置條件

- Python venv：`/Users/kirk/.openclaw/workspace/.venv`
- GCP 認證：`/Users/kirk/.openclaw/workspace/.gcp/adc.json`
- BQ Project：`production-379804`
- Firebase RTDB：`https://pm-prototype-a75ce-default-rtdb.asia-southeast1.firebasedatabase.app`

---

## Step 1：查詢 member_id

```python
from google.oauth2.credentials import Credentials
from google.auth.transport.requests import Request
from google.cloud import bigquery
import json

with open('/Users/kirk/.openclaw/workspace/.gcp/adc.json') as f:
    info = json.load(f)
creds = Credentials(
    token=None,
    refresh_token=info['refresh_token'],
    client_id=info['client_id'],
    client_secret=info['client_secret'],
    token_uri='https://oauth2.googleapis.com/token'
)
creds.refresh(Request())
client = bigquery.Client(credentials=creds, project='production-379804')

phones = ['0937508965', '0920330504']  # 修改這裡

q = f"""
WITH latest AS (
  SELECT member_hk, phone_number,
    ROW_NUMBER() OVER (PARTITION BY member_hk ORDER BY effective_from DESC) AS rn
  FROM `production-379804.intermediate.sat__member`
  WHERE phone_number IN ({','.join(repr(p) for p in phones)})
)
SELECT l.phone_number, h.member_id
FROM latest l
JOIN `production-379804.base_marts.base__hub__member` h USING (member_hk)
WHERE l.rn = 1
ORDER BY h.member_id ASC
"""
for row in client.query(q).result():
    print(row.phone_number, row.member_id)
```

---

## Step 2：查詢近 6 個月發票

```python
member_ids = [351987, 5037450]  # 替換為實際 member_id

q = """
SELECT
  h.member_id,
  FORMAT_DATETIME('%Y-%m', i.issued_at) as year_month,
  i.seller_tax_id,
  CAST(i.total_price AS FLOAT64) as amount,
  i.type,
  TO_HEX(i.invoice_hk) as invoice_hk
FROM `production-379804.base_marts.base__link__member_invoice` lmi
JOIN `production-379804.base_marts.base__hub__member` h USING (member_hk)
JOIN `production-379804.base_marts.base__sat__invoice` i ON lmi.invoice_hk = i.invoice_hk
WHERE h.member_id IN ({ids})
  AND i.issued_at >= DATE_SUB(CURRENT_DATETIME('Asia/Taipei'), INTERVAL 6 MONTH)
ORDER BY member_id, i.issued_at DESC
""".format(ids=','.join(str(m) for m in member_ids))

rows = list(client.query(q).result())
```

---

## Step 3：統編轉店名

```python
all_tax_ids = list(set(r.seller_tax_id for r in rows if r.seller_tax_id))
tax_map = {}

# 批次查 psa__mysql_seller（每次最多 100 個）
chunk_size = 100
for i in range(0, len(all_tax_ids), chunk_size):
    chunk = all_tax_ids[i:i+chunk_size]
    ids_str = ','.join(f"'{t}'" for t in chunk)
    q2 = f"SELECT DISTINCT tax_id, name FROM `production-379804.staging.psa__mysql_seller` WHERE tax_id IN ({ids_str})"
    for row in client.query(q2).result():
        if row.tax_id and row.name:
            tax_map[row.tax_id] = row.name
```

### 特殊統編對照（已知）

| 統編 | 店名 |
|------|------|
| 83118125 | UberEats（優步福爾摩沙） |
| 83076928 | foodpanda（優食台灣） |
| 90413278 | 全家便利商店 |
| 80333992 | Apple |
| 84497849 | Google |
| 27492064 | Netflix |

---

## Step 4：訂閱判斷邏輯

### 外送訂閱（UberEats / foodpanda）

- 月費：UberEats **$120**、foodpanda **$119**
- **訂閱用戶判斷**：連續 2 個月以上有消費，或有月費發票（金額 >= 月費 × 0.7）
- 月費發票：`amount >= 月費 × 0.7`
- 訂單發票：`amount < 月費 × 0.7`（小額，平台服務費）

每個月計算：
```python
fee_invs  = [i for i in invs if i['amount'] >= fee * 0.7]   # 月費發票
order_invs = [i for i in invs if i['amount'] < fee * 0.7]   # 訂單發票

extra_spend = sum(i['amount'] for i in order_invs)   # 平台額外花費
total_spend = fee + extra_spend                       # 平台總花費
orders = len(order_invs)                              # 訂單筆數
fee_waived = orders * 49                              # 估計省免運費（每單 $49）
roi = fee_waived - fee                                # 訂閱 ROI
```

renewDay：從月費發票的 `issued_at` 取日期的日（day）。

### 定額訂閱（Apple / Google / Netflix）

- 直接從發票月份取金額，近 3 個月
- 無額外花費計算

---

## Step 5：建立 Firebase 資料結構

```python
import requests

user_data = {
    'invoices': [
        {'shop': '全家', 'amount': 65, 'yearMonth': '2026-03', 'type': ''}
        # ... 所有發票
    ],
    'invoiceCount': 570,
    'totalAmount': 366399,
    'monthlyTrend': [
        {'month': '10月', 'amount': 74380},
        # ... 近 6 個月
    ],
    'pieData': [
        {'label': '外食', 'pct': 32, 'color': '#378ADD'},
        # ...
    ],
    'autoTasks': [
        {
            'id': '全家', 'shop': '全家', 'title': '全家 消費達標',
            'icon': '🏪', 'desc': '近6個月已消費99次，本月達標領獎勵',
            'progress': 16, 'target': 20, 'unit': '次',
            'reward': 50, 'daysLeft': 14, 'urgency': 'high',
            'basis': '你在全家的消費頻率名列前茅',
        }
    ],
    'deliverySubs': [
        {
            'id': 'ubereats', 'name': 'UberEats+', 'icon': '🛵',
            'fee': 120, 'type': 'roi', 'renewDay': 23,
            'months': [
                {'m': '01月', 'orders': 20, 'feeWaived': 980, 'extraSpend': 128, 'totalSpend': 248},
                # ...
            ],
            'roiStatus': 'green',  # green / warn / danger
            'roiLabel': '本月划算 +860',
            'roiTip': '本月叫了24次，估計省免運費$1176，訂閱費$120，省了$1056。',
            'actionLabel': '繼續保留',
            'actionColor': '#3B6D11',
            'actionBg': '#EAF3DE',
        }
    ],
    'flatSubs': [
        {
            'id': 'apple', 'name': 'Apple', 'icon': '☁️',
            'fee': 90, 'renewDay': 24,
            'trend': 'stable',  # stable / up
            'months': [90, 90, 90],
        }
    ],
}

phone = '0937508965'
resp = requests.put(
    f"https://pm-prototype-a75ce-default-rtdb.asia-southeast1.firebasedatabase.app/users/{phone}.json",
    json=user_data,
    timeout=30
)
print(resp.status_code)  # 200 = 成功
```

---

## 注意事項

1. **去重**：用 `invoice_hk` 去重，避免重複發票
2. **消費分類**：`psa__mysql_seller` 查不到的統編，用店名關鍵字分類
3. **autoTasks 排除**：停車場、加油站、Apple、Google 等不適合做任務
4. **店名正規化**：用 `psa__mysql_seller.name` 去掉「股份有限公司」、分公司後綴後取前 10 字
5. **renewDay**：從月費發票的 `issued_at` 取日，比固定 hardcode 更準確

---

## 常用統計 SQL

```sql
-- 查某帳號所有訂閱的月費發票
SELECT seller_tax_id, FORMAT_DATETIME('%Y-%m', issued_at) as ym,
       COUNT(*) as cnt, SUM(CAST(total_price AS FLOAT64)) as total
FROM `production-379804.base_marts.base__link__member_invoice` lmi
JOIN `production-379804.base_marts.base__hub__member` h USING (member_hk)
JOIN `production-379804.base_marts.base__sat__invoice` i ON lmi.invoice_hk = i.invoice_hk
WHERE h.member_id = {MEMBER_ID}
  AND seller_tax_id IN ('83118125','83076928','80333992','84497849')
GROUP BY 1, 2
ORDER BY 1, 2
```
