# 前端常見陷阱與修正記錄

> 專案：發票存摺 Demo
> 前端位置：`/Users/kirk/.openclaw/workspace/invoice-prototype/src/App.jsx`
> GitHub Pages：`https://kirkchen0302.github.io/openclaw-workspace/`

---

## 1. deliverySubs / flatSubs fallback 問題

**症狀**：所有帳號的訂閱顯示一模一樣（demo 資料）

**原因**：
```js
// 舊寫法 — null 或空陣列都會 fallback 到 demo 資料
const liveDeliverySubs = user?.data?.deliverySubs || SUBSCRIPTIONS;
const liveFlatSubs     = user?.data?.flatSubs     || FLAT_SUBS;
```

**修正**：
```js
// 新寫法 — 只有 Array 才用，否則空陣列（不 fallback）
const liveDeliverySubs = Array.isArray(user?.data?.deliverySubs) ? user.data.deliverySubs : [];
const liveFlatSubs     = Array.isArray(user?.data?.flatSubs)     ? user.data.flatSubs     : [];
```

---

## 2. flatSubs 從 invoices 自動重算的 fallback

**症狀**：本月訂閱總支出所有人都是 $417（$327 外送 + $90 Apple）

**原因**：`computedFlatSubs` 有 fallback 邏輯，當 `flatSubs` 為空時會自動從 `invoices` 掃 Apple/Google 重算。

**修正**：
```js
// 只用 Firebase 資料，沒資料就空陣列，不從 invoices 重算
const computedFlatSubs = useMemo(() => {
  if (!flatSubs || flatSubs.length === 0) return [];
  return flatSubs.map(sub => ({ ...sub, renewDay: sub.renewDay ?? "—" }));
}, [flatSubs]);
```

---

## 3. renewDay 固定 hardcode

**症狀**：每個帳號的訂閱續訂日都一樣（UberEats 15日、foodpanda 22日）

**修正**：從 BQ 查每個帳號最後一筆月費發票的 `issued_at` 取日（day），寫入 Firebase：
```python
renew_day = int(fee_invs[0]['day'])  # 從月費發票的日期取
```

---

## 4. 本月訂閱總支出 hardcode 月份標籤

**症狀**：bar chart 月份永遠顯示「12月、1月、2月」

**修正**：
```js
const subMonthLabels = (() => {
  const allLabels = deliverySubs.flatMap(s => (s.months || []).map(m => m.m));
  const unique = [...new Set(allLabels)];
  return unique.length > 0 ? unique : [];
})();
```

---

## 5. 訂閱項目數量 hardcode

**症狀**：訂閱項目數永遠顯示固定數字

**修正**：
```js
const totalSubCount = deliverySubs.length + computedFlatSubs.length;
// 顯示時用 {totalSubCount} 替換原本的 {SUBSCRIPTIONS.length + FLAT_SUBS.length}
```

---

## 部署流程

每次改完 `App.jsx` 後：

```bash
cd /Users/kirk/.openclaw/workspace/invoice-prototype
npm run build
cp -r dist/* /Users/kirk/.openclaw/workspace/docs/
cd /Users/kirk/.openclaw/workspace
git add invoice-prototype/src/App.jsx docs/
git commit -m "fix: 描述"
git push
```

GitHub Pages 約 **1-2 分鐘**後生效。瀏覽器需強制重新整理（`Cmd+Shift+R`）清快取。
