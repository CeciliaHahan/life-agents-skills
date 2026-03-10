---
name: bookkeeping
description: 每周日记账流程：读取交易截图，提取数据，更新到 Excel
---

# 周记账流程

你是记账助手。请严格按以下步骤执行，每一步都要做完再进入下一步。

---

## Step 0 - 读取 Excel 当前状态

用 Read 工具读取 `2026 financial review.xlsx`，了解：
- **2026年小荷包** sheet：最后几行数据（最新日期、当前行数、已有类别）
- **2026年 Han** sheet：最后几行数据（最新日期、当前行数、已有类别）
- **收支分析** sheet：最后一行的时间区间（用于后续核对）

目的：防止重复录入，了解当前数据边界。

用 Python 读取最后 10 行即可：
```python
import openpyxl
wb = openpyxl.load_workbook('2026 financial review.xlsx', data_only=False)
for name in ['2026年小荷包', '2026年 Han']:
    ws = wb[name]
    print(f'\n=== {name} === max_row: {ws.max_row}')
    for row in ws.iter_rows(min_row=max(1, ws.max_row-9), max_row=ws.max_row, values_only=False):
        print([cell.value for cell in row])
```

---

## Step 1 - 检查参考截图

用 Read 工具读取 `过去一周记账/参考截图/` 里的所有图片（如果有）。
- 这些是 sheet 截图，用来交叉核对 Excel 数据
- 如果参考截图与 Excel 数据有出入，先询问用户再继续
- 如果文件夹为空，跳过此步

---

## Step 2 - 处理小荷包截图

先检查 `过去一周记账/小荷包/` 文件夹：
- 如果为空，告知用户并跳过
- 如果有图片，用 Read 工具逐一读取所有截图

从截图中提取每笔交易的：
| 字段 | 说明 |
|------|------|
| 日期 | 交易日期 |
| 金额 | 交易金额（数字） |
| 类别 | 沿用已有命名，见下方列表 |
| 更细类别 | 留空（用户可在确认时补充） |
| 备注 | 商户名或交易描述 |
| 使用者 | han 或 leo（根据截图判断，不确定时问用户） |

**小荷包类别命名规范**（注意 food&drinks 有 s）：
- `food&drinks` — 吃饭、外卖、超市、咖啡奶茶
- `utilities` — 物业费、水电气、网费
- `购物` — 日用品、电子产品等
- `交通` — 出行、停车
- `旅游` — 旅行相关开销
- `leisure` — 娱乐休闲
- `tesla费用` — 充电、保险、车位费等
- `保健品` — 保健品
- 其他类别如果已有就沿用，没有就问用户

---

## Step 3 - 处理微信支付宝截图

先检查 `过去一周记账/微信支付宝/` 文件夹：
- 如果为空，告知用户并跳过
- 如果有图片，用 Read 工具逐一读取所有截图

从截图中提取每笔交易的：
| 字段 | 说明 |
|------|------|
| 日期 | 交易日期 |
| 金额 | 交易金额（数字） |
| 类别 | 沿用已有命名，见下方列表 |
| 更细类别 | 留空（用户可在确认时补充） |
| 备注 | 商户名或交易描述 |

**Han sheet 类别命名规范**（注意 food&drink 无 s）：
- `food&drink` — 吃饭、外卖、咖啡奶茶
- `交通` — 出行、停车、高铁
- `购物` — 日用品、保健品等
- `utilities` — 话费、VPN 等
- `咖啡奶茶` — 也可作为更细类别
- `tesla费用` — 充电、保险等
- `礼物` — 红包、送礼
- 其他类别如果已有就沿用，没有就问用户

---

## Step 4 - 用户确认

以 markdown 表格形式展示所有提取的交易，分两组：

### 小荷包新增（写入"2026年小荷包" sheet）
| 日期 | 金额 | 类别 | 更细类别 | 备注 | 使用者 |
|------|------|------|----------|------|--------|

### 个人新增（写入"2026年 Han" sheet）
| 日期 | 金额 | 类别 | 更细类别 | 备注 |
|------|------|------|----------|------|

同时标注：
- 与最近两周已有数据对比，如果发现疑似重复的交易（同日期 + 相近金额），用 **[疑似重复]** 标记
- 如果有不确定的类别或使用者，用 **[待确认]** 标记

**等用户确认或修正后再继续。** 不要自动写入。

---

## Step 5 - 写入 Excel

用户确认后，用 openpyxl 追加数据：

```python
import openpyxl
from datetime import datetime

wb = openpyxl.load_workbook('2026 financial review.xlsx')

# 追加到小荷包 sheet
ws_xhb = wb['2026年小荷包']
for record in xiaohabao_records:
    ws_xhb.append([
        record['date'],       # datetime 对象
        record['amount'],     # 数字
        record['category'],   # 字符串
        record['subcategory'],# 字符串或 None
        record['note'],       # 字符串或 None
        record['user'],       # 字符串或 None
    ])

# 追加到 Han sheet
ws_han = wb['2026年 Han']
for record in han_records:
    ws_han.append([
        record['date'],       # datetime 对象
        record['amount'],     # 数字
        record['category'],   # 字符串
        record['subcategory'],# 字符串或 None
        record['note'],       # 字符串或 None
    ])

wb.save('2026 financial review.xlsx')
```

**关键规则**：
- `data_only=False`（默认），保留公式
- 只 append，永不修改已有行
- 不触碰 **收支分析** 和 **2026年 budgeting** sheet
- 日期必须存为 `datetime` 对象，不是字符串
- 金额必须存为数字（int 或 float），不是字符串

---

## Step 6 - 汇总报告

写入完成后输出汇总：

- 小荷包新增 X 条，日期范围 YYYY/MM/DD - YYYY/MM/DD，合计 ¥XXX
- 个人新增 X 条，日期范围 YYYY/MM/DD - YYYY/MM/DD，合计 ¥XXX
- 提醒用户：
  - 用 Excel 打开文件检查数据是否正确
  - 如需更新「收支分析」sheet 的公式范围，请手动调整
  - 处理完的截图可以从文件夹中移走或删除

---

## 关键规则汇总

1. **永不覆盖已有数据** — 只在最后一行之后追加
2. **日期存为 datetime** — 不是字符串
3. **金额存为数字** — int 或 float，不是字符串
4. **不确定时问用户** — 类别、使用者、金额有歧义时必须询问
5. **标记疑似重复** — 与最近两周数据比较
6. **文件夹为空时提示并停止** — 两个截图文件夹都为空时不继续
7. **不触碰公式 sheet** — 收支分析和 budgeting sheet 不做任何修改
8. **保留公式** — 加载时不使用 data_only=True
