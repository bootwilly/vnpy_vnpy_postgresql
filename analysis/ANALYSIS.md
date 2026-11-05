# vnpy_postgresql（PostgreSQL 資料庫驅動）倉庫分析

> 倉庫：`bootwilly/vnpy_vnpy_postgresql`　版本：`1.0.3`
> 定位：VeighNa 的 **資料庫驅動外掛**，基於 peewee ORM 實作 `vnpy.trader.database.BaseDatabase`，把 K 線與 Tick 資料存入 PostgreSQL。不是 App，沒有 UI；由核心的 `get_database()` 依設定動態載入。
> 流程圖：[FLOWCHART.md](./FLOWCHART.md)　整合分析：`vnpy_vnpy/analysis/INTEGRATED_ANALYSIS.md`

## 1. 目錄結構

```
vnpy_postgresql/
├── __init__.py               from .postgresql_database import PostgresqlDatabase as Database
└── postgresql_database.py    peewee 連線、四個資料表 Model、PostgresqlDatabase 實作
```

## 2. 啟用方式

`.vntrader/vt_setting.json`：
```json
{
  "database.name": "postgresql",
  "database.host": "localhost",
  "database.port": 5432,
  "database.database": "vnpy",
  "database.user": "postgres",
  "database.password": "******"
}
```
核心 `get_database()` 會 `import_module("vnpy_postgresql")` 並建立 `module.Database()`。資料庫（database instance）必須事先手動建立；資料表會自動建立。

## 3. 資料表結構

| Model | 主要欄位 | 唯一索引 |
|---|---|---|
| `DbBarData` | symbol, exchange, datetime, interval, volume, turnover, open_interest, OHLC | (symbol, exchange, interval, datetime) |
| `DbTickData` | symbol, exchange, datetime, name, 成交/持倉、last/limit/OHLC/pre_close、五檔買賣價量、localtime | (symbol, exchange, datetime) |
| `DbBarOverview` | symbol, exchange, interval, count, start, end | (symbol, exchange, interval) |
| `DbTickOverview` | symbol, exchange, count, start, end | (symbol, exchange) |

`exchange`、`interval` 以列舉的字串值儲存；`datetime` 以去除時區的 `database.timezone` 時間儲存（`convert_tz`），讀出時再以 `DB_TZ` 還原。

## 4. 方法實作

| 方法 | 實作要點 |
|---|---|
| `__init__` | 共用模組層級的 `db` 物件，`connect(reuse_if_open=True)`，`create_tables(...)` |
| `save_bar_data(bars, stream)` | 轉 dict → 每 100 筆 `insert_many(...).on_conflict(...)` 在 transaction 中執行 → 更新 BarOverview |
| `save_tick_data(ticks, stream)` | 逐筆 `insert(d).on_conflict(update=d)`，**再**每 100 筆 `insert_many(...).on_conflict(...)` → 更新 TickOverview |
| Overview 更新 | 不存在：以本批資料建立；`stream=True`：只更新 end 並 `count += len`；否則：start/end 取 min/max 並重新 `count()` |
| `load_bar_data / load_tick_data` | 依條件查詢並 `order_by(datetime)`，轉回 `BarData / TickData(gateway_name="DB")` |
| `delete_bar_data / delete_tick_data` | 刪除資料並刪除對應 overview，回傳刪除筆數 |
| `get_bar_overview` | 若有 K 線但沒有 overview（舊版資料）會先 `init_bar_overview()` 以 GROUP BY 重建 |

## 5. 依賴

`peewee`（PostgreSQL 驅動另需 `psycopg2`，未列在 `install_requires`）；使用 `list[BarData]` 等內建泛型標註，**需要 Python ≥ 3.9**（README 標示 3.10–3.12），而核心 vnpy 宣稱支援 3.8。

## 6. 觀察與建議

1. **K 線 upsert 實際上不會覆寫舊資料（邏輯缺陷）**：`on_conflict(update={DbBarData.volume: DbBarData.volume, ...})` 經 peewee 產生的 SQL 為 `ON CONFLICT (...) DO UPDATE SET "volume" = "dbbardata"."volume"`，等於把欄位設回**原有值**。重新下載／修正資料時，已存在時間點的 K 線不會被更新。應改用 `EXCLUDED`，例如 `on_conflict(preserve=[DbBarData.volume, ...])`。（已以 peewee 產生 SQL 驗證）
2. **Tick 資料重複寫入**：`save_tick_data` 先逐筆 upsert（`update=d`，會正確覆寫），之後又對同一批資料 `insert_many` 一次（同上，等同 no-op），使寫入成本約加倍，且逐筆 insert 在高頻錄製時是主要瓶頸。建議移除逐筆迴圈，並把 `insert_many` 改為 `preserve=` 寫法。
3. **會修改呼叫端傳入的物件**：`d = bar.__dict__` 直接取用物件本身的 dict，接著 `pop("gateway_name")`、`pop("vt_symbol")`、把 exchange 改成字串、datetime 改成 naive。呼叫 `save_*` 之後，原本的 `BarData / TickData` 已被破壞，若呼叫端再使用（如 `bar.vt_symbol`、`bar.exchange.value`）會出錯。建議改為 `copy(bar.__dict__)`。
4. **stream 模式的 count 可能高估**：`count += len(bars)` 不排除與既有資料衝突的筆數。
5. **模組匯入即建立連線物件**：`db = PeeweePostgresqlDatabase(...)` 在 import 時讀取 `SETTINGS`，設定錯誤時錯誤訊息出現在 `get_database()` 呼叫處，不易排查。
6. 未宣告 `psycopg2` 依賴；Python 版本需求與核心不一致。
