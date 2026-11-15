# SQL 筛查规则库（12 条）

> 用途：《审计报告》「规则筛查」一节的可扩展版。所有语句按通用 SQL 写（MySQL 8 / SQLite / PostgreSQL 均可改两处方言后运行）。
> 前提表结构见文末。**每条规则都要先说清「命中之后做什么」，否则跑出来的只是名单。**

## 一、基础表结构（构造数据用）

```sql
-- 销售发票主表
invoice(invoice_no PK, customer_id, invoice_date DATE, amount DECIMAL(14,2),
        tax_rate, contract_no, qty, unit_price, salesperson, status)

-- 销售退回
sales_return(return_no PK, invoice_no, return_date DATE, amount DECIMAL(14,2), reason)

-- 客户 / 供应商（用于同址比对）
customer(id PK, name, addr, phone, bank_acct, legal_person, credit_limit)
vendor(id PK, name, addr, phone, bank_acct, legal_person)

-- 审批流日志
approval_log(id PK, invoice_no, approver, approve_time DATETIME, action, node)
```

## 二、十二条规则

### 规则 1 · 同日同客户拆单，合计跨过审批线

```sql
SELECT customer_id, invoice_date, COUNT(*) AS n, SUM(amount) AS amt
FROM invoice
WHERE invoice_date BETWEEN '2025-01-01' AND '2025-12-31'
GROUP BY customer_id, invoice_date
HAVING COUNT(*) >= 3 AND SUM(amount) >= 50000
ORDER BY amt DESC;
```

**命中后**：重算合计，调审批日志看有没有一笔按合计金额审批；没有就是规避授权。再看这 3—4 笔是否同一天由同一人开票。

### 规则 2 · 整数金额且金额较大

```sql
SELECT invoice_no, customer_id, invoice_date, amount
FROM invoice
WHERE amount >= 100000 AND amount % 10000 = 0
ORDER BY amount DESC;
```

**命中后**：拿合同价比对。合同 48.6 万、发票开 50 万，差额就是问题；合同本来就是 50 万，只是定价习惯，划掉。

### 规则 3 · 年末开票、次年退货

```sql
SELECT i.invoice_no, i.customer_id, i.invoice_date, i.amount,
       r.return_date, r.amount AS ret_amt
FROM invoice i
JOIN sales_return r ON r.invoice_no = i.invoice_no
WHERE i.invoice_date >= '2025-12-27'
  AND r.return_date < '2026-02-01'
ORDER BY i.amount DESC;
```

**命中后**：这是提前确认收入最典型的形态。调出库单与物流签收，看签收日是否落在 2026 年；签收在次年、发票在本年，提请调整。

### 规则 4 · 发票号断号

```sql
WITH seq AS (
  SELECT CAST(SUBSTRING(invoice_no, 9) AS UNSIGNED) AS n
  FROM invoice WHERE invoice_no LIKE 'FP-2025-%'
)
SELECT prev.n + 1 AS gap_start, cur.n - 1 AS gap_end, cur.n - prev.n - 1 AS missing
FROM seq cur JOIN seq prev ON cur.n > prev.n
WHERE cur.n - prev.n > 1;
```

**命中后**：调作废发票清单与税控系统开具记录。作废合规会留痕；没留痕说明有账外收入或撕票。

### 规则 5 · 客户与供应商同址 / 同电话 / 同法人

```sql
SELECT c.name AS cust, v.name AS vend, c.addr, c.phone
FROM customer c JOIN vendor v
  ON c.addr = v.addr OR c.phone = v.phone OR c.legal_person = v.legal_person
WHERE c.addr <> '' AND c.addr IS NOT NULL;
```

**命中后**：企查查/工商信息比对股权关系；再查该客户付款后 3 日内资金是否流向该供应商或股东个人账户（资金回流）。

### 规则 6 · 信用额度超限仍在发货

```sql
SELECT c.id, c.name, c.credit_limit,
       SUM(i.amount) AS ytd_sales
FROM customer c JOIN invoice i ON i.customer_id = c.id
WHERE i.invoice_date BETWEEN '2025-01-01' AND '2025-12-31'
GROUP BY c.id, c.name, c.credit_limit
HAVING SUM(i.amount) > c.credit_limit
ORDER BY ytd_sales - c.credit_limit DESC;
```

**命中后**：调授信审批单与超限放行记录。若是系统未配置额度校验，属于 ITGC 缺陷，不只是业务问题。

### 规则 7 · 单一客户集中度

```sql
SELECT customer_id, SUM(amount) AS amt,
       SUM(amount) / (SELECT SUM(amount) FROM invoice) AS pct
FROM invoice GROUP BY customer_id
HAVING pct > 0.10 ORDER BY amt DESC;
```

**命中后**：看合同期限、关联关系、期后回款。集中度高本身不是错报，但影响持续经营假设与收入质量。

### 规则 8 · 毛利率异常月份

```sql
SELECT DATE_FORMAT(i.invoice_date,'%Y-%m') AS ym,
       SUM(i.amount) AS revenue,
       SUM(i.qty * p.std_cost) AS cost,
       ROUND(1 - SUM(i.qty * p.std_cost) / SUM(i.amount), 4) AS gross_margin
FROM invoice i JOIN product p ON p.id = i.product_id
GROUP BY ym ORDER BY ym;
```

**命中后**：某月毛利率显著跳升，先查成本结转是否漏记，再查收入是否虚增。两者都会让毛利率变好看。

### 规则 9 · 同一客户短时间密集开票

```sql
SELECT customer_id, invoice_date, COUNT(*) AS n,
       MIN(amount) AS min_amt, MAX(amount) AS max_amt
FROM invoice
GROUP BY customer_id, invoice_date
HAVING COUNT(*) >= 5
ORDER BY n DESC;
```

**命中后**：与规则 1 互补。规则 1 看「合计跨线」，这条看「笔数异常」，用于发现批量开票或系统重复生成。

### 规则 10 · 销售员代收货款 / 收款人与客户不一致

```sql
SELECT r.invoice_no, i.salesperson, r.payer_name, c.name AS customer
FROM receipt r
JOIN invoice i ON i.invoice_no = r.invoice_no
JOIN customer c ON c.id = i.customer_id
WHERE r.payer_name <> c.name;
```

**命中后**：第三方代付本身可能是正常（集团统一付款），但要看有没有授权书；销售员个人账户收款一律高危。

### 规则 11 · 期后大额红冲

```sql
SELECT invoice_no, return_date, amount
FROM sales_return
WHERE return_date BETWEEN '2026-01-01' AND '2026-03-31'
  AND amount >= 500000
ORDER BY amount DESC;
```

**命中后**：审计报告日前后的大额红冲，直接影响期末收入。逐笔要原因、要审批、要物流。

### 规则 12 · 审批人与开票人同一人

```sql
SELECT a.invoice_no, a.approver, i.salesperson
FROM approval_log a JOIN invoice i ON i.invoice_no = a.invoice_no
WHERE a.node = '审批' AND a.approver = i.salesperson;
```

**命中后**：职责未分离，属于控制设计缺陷。逐笔不算错报，但要在内控缺陷汇总里记一条，并评估是否升级为重要缺陷。

## 三、跑完之后的三件事

1. **去重**：同一笔可能命中多条规则，按 `invoice_no` 去重后再算金额，否则会重复计数。
2. **分层**：按金额降序，先看前 20%。审计资源有限，先查影响大的。
3. **留痕**：每条规则、参数、执行时间、命中数写进底稿。别人要能复现你的结果，否则等于没做。

## 四、规则写不出来的部分

SQL 能筛「形态」，筛不出「意图」。以下三类只能靠人：

- 交易是否真实发生（需要外部证据）；
- 管理层为何这么做（需要询问与判断）；
- 这个错报对报表使用者是否重大（需要职业判断）。

工具把范围从 4,860 缩到 37，剩下的 37 条要一条一条问。
