# Firebase RTDB 資料 Schema

> 系統：發票存摺 Demo
> RTDB URL：`https://pm-prototype-a75ce-default-rtdb.asia-southeast1.firebasedatabase.app`
> 節點路徑：`users/{phone_number}`

---

## 完整 Schema

```json
{
  "invoices": [
    {
      "shop": "全家",
      "amount": 65,
      "yearMonth": "2026-03",
      "type": ""
    }
  ],
  "invoiceCount": 570,
  "totalAmount": 366399,
  "monthlyTrend": [
    { "month": "10月", "amount": 74380 },
    { "month": "11月", "amount": 45988 },
    { "month": "12月", "amount": 64794 },
    { "month": "1月",  "amount": 44162 },
    { "month": "2月",  "amount": 85645 },
    { "month": "3月",  "amount": 37533 }
  ],
  "pieData": [
    { "label": "外食",    "pct": 32, "color": "#378ADD" },
    { "label": "百貨購物","pct": 28, "color": "#639922" },
    { "label": "超商",    "pct": 14, "color": "#EF9F27" },
    { "label": "訂閱數位","pct": 12, "color": "#7F77DD" },
    { "label": "交通停車","pct": 8,  "color": "#BA7517" },
    { "label": "其他",    "pct": 6,  "color": "#8E8E93" }
  ],
  "autoTasks": [
    {
      "id": "全家",
      "shop": "全家",
      "title": "全家 消費達標",
      "icon": "🏪",
      "desc": "近6個月已消費99次，本月達標領獎勵",
      "progress": 16,
      "target": 20,
      "unit": "次",
      "reward": 50,
      "daysLeft": 14,
      "urgency": "high",
      "basis": "你在全家的消費頻率名列前茅"
    }
  ],
  "deliverySubs": [
    {
      "id": "ubereats",
      "name": "UberEats+",
      "icon": "🛵",
      "fee": 120,
      "type": "roi",
      "renewDay": 23,
      "months": [
        {
          "m": "01月",
          "orders": 20,
          "feeWaived": 980,
          "extraSpend": 128,
          "totalSpend": 248
        }
      ],
      "roiStatus": "green",
      "roiLabel": "本月划算 +860",
      "roiTip": "本月叫了24次，估計省免運費$1176，訂閱費$120，省了$1056。",
      "actionLabel": "繼續保留",
      "actionColor": "#3B6D11",
      "actionBg": "#EAF3DE"
    }
  ],
  "flatSubs": [
    {
      "id": "apple",
      "name": "Apple",
      "icon": "☁️",
      "fee": 90,
      "renewDay": 24,
      "trend": "stable",
      "months": [90, 90, 90]
    }
  ]
}
```

---

## 欄位說明

### deliverySubs[].months[]

| 欄位 | 型別 | 說明 |
|------|------|------|
| `m` | string | 月份標籤，如「01月」 |
| `orders` | number | 當月訂單筆數（排除月費發票） |
| `feeWaived` | number | 估計省免運費（orders × $49） |
| `extraSpend` | number | 平台額外花費（小額發票加總，不含月費） |
| `totalSpend` | number | 平台總花費（月費 + extraSpend） |

### roiStatus

| 值 | 條件 | 顯示顏色 |
|----|------|----------|
| `green` | roi >= 0 | 綠色 |
| `warn` | -30 < roi < 0 | 橘色 |
| `danger` | roi <= -30 | 紅色 |

### flatSubs[].trend

| 值 | 說明 |
|----|------|
| `stable` | 費用持平 |
| `up` | 費用調漲 |

---

## 正確月費（2026-03 確認）

| 平台 | 月費 |
|------|------|
| UberEats+ | $120 |
| foodpanda Pro | $119 |

---

## 注意

- `deliverySubs` 為 `null` 表示該用戶無外送訂閱（前端不 fallback）
- `flatSubs` 為 `null` 表示該用戶無定額訂閱（前端不 fallback）
- 前端邏輯：`Array.isArray(user?.data?.deliverySubs) ? user.data.deliverySubs : []`
