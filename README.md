# BOM 叶件用量数仓

制造企业成本数仓的多级 BOM 叶件用量表：浏览器维护父子关系与用量，整批提交后
沿路径乘用量、多路径相加，把结果**落库**；查看与下载只读已落库的快照。

## 快速启动

```bash
docker compose up --build
```

- 网页 A：<http://localhost:8000>（进程 `web-a`）
- 网页 B：<http://localhost:8001>（进程 `web-b`，与 A 共用同一个 SQLite 文件）
- `seed` 一次性任务会通过 HTTP 载入演示结构，并把叶件 `L-4` 提升为装配件
  （`L-4 → L-6/L-7`），两个样例都可直接看到。

可选：在两个真实容器之间验证交错修改：

```bash
docker compose --profile demo run --rm interleave
```

本地（不使用 Docker）：

```bash
python -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --port 8000            # 第二个进程换端口即可
python scripts/concurrency_demo.py           # 起两个真实进程做交错验证
pytest -q                                    # 64 项测试
```

## 功能与约束

| 要求 | 实现 |
| --- | --- |
| 最多 100 个料号、200 条父子关系 | `validate_limits`（超限返回 409，整批不写） |
| 浏览器成批增改删、正整数用量 | 单条操作先进入「批改清单」，「提交整批」一次提交；用量 1..1e9 |
| 每个装配件展开到叶件、沿路径乘用量、多路径相加 | `BomStore.expand`（按子装配记忆化，共享子装配不会指数爆炸） |
| 结果必须落库 | `leaf_usage` 表中按版本物化，启动/查看/下载都不再展开 |
| 以整批修改后的结构判环，显示路径，遇环整批不改 | `find_cycles` 基于最终结构；409 返回 `paths` 如 `A → B → C → A`，事务回滚 |
| 两个 Web 进程共库 | 同一 `BOM_DB_PATH`，WAL + `busy_timeout`；`docker-compose.yml` 起两份 |
| 递归展开不占写锁，其他修改可先提交 | 展开与判环都在内存快照上完成，之后才 `BEGIN IMMEDIATE` |
| 迟到的展开不能覆盖新结构/漏掉新依赖 | 提交时对 `bom_head.version_seq` 做 CAS；失败则按最新结构**重新规划并重新判环** |
| 旧快照上溢出/成环/超限不算最终结论 | 计算失败先在写锁槽内复核发布版本号；已变化则整批按最新结构**重跑校验与展开**，只有最新结构仍失败才拒绝 |
| 迟到的批改不被误报“用量溢出” | 发布版本已降低用量时，迟到批改重算后成功；最新结构仍溢出则 422 整批拒绝，已发布版本原地保留 |
| 分别无环的并发修改不能合成环 | 复核在最新结构上重跑，合成环会被拒绝 |
| 关系与受影响用量表同时更新，失败保留旧版 | 新版本写入 `bom_edge` + `leaf_usage` + 切换 `bom_head` 全在同一事务 |
| 查看/下载不混两个提交、不临时展开 | 每次请求在其读事务内解析一次 `bom_head`，只 SELECT 物化行 |
| 重启保留结果 | 数据在卷 `bom-data` 的 SQLite 文件中；版本与用量行都在 |
| 共享子装配 / 叶件转装配件 / 可暂停计算的交错修改样例 | 见下文 |

## 并发模型

```
读快照 ──► 应用批改 ──► 判环 ──► 展开（全部在内存，无写锁）
                                        │
                      ┌─────────────────┴─────────────────┐
                成功  ▼                              失败（溢出/环/超限）
      BEGIN IMMEDIATE ──► CAS 版本号                  │
         成功：写新版本 + 用量 + 切 head + 清理旧行     ▼
         失败：ROLLBACK，回到第一步按最新结构重来   BEGIN IMMEDIATE 复核版本号
                                                      │
                                          已变化 ──► ROLLBACK，按最新结构重来
                                          未变化 ──► 报错（422/409），什么都不写
```

- 写锁只在最后一段纯 SQL 事务内持有，毫秒级；期间别的进程可以正常提交。
- CAS 失败说明期间有新提交，此时**旧计划作废**：重新读取结构、重新应用批改、
  重新判环、重新展开。因此迟到的展开不会覆盖新结构，也不会漏掉新依赖。
- **旧快照上的失败也不是最终结论**：溢出、成环、容量超限都是针对旧快照算出的，
  可能已被另一进程的提交修复（例如把 1e9 的用量改成 1）。因此失败时先在写锁
  槽内复核 `bom_head.version_seq`：已变化就把整批按最新结构重新应用、重判环、
  重展开；只有最新结构仍然溢出/成环/超限才返回错误，且此时**什么都不写**，
  已发布版本与物化用量行原样保留。
- 两次分别无环的修改若合起来成环，后提交的一方会在重规划后的结构上发现环而
  整批失败（返回路径）。测试 `test_two_acyclic_edits_cannot_combine_into_cycle`
  与 `scripts/concurrency_demo.py` 场景 3 覆盖此点。
- 只保留当前发布版本的边与用量行（同事务内清理），因此重启后读到的是同一份
  结果，且存储量有界。

## 样例

1. **共享子装配**：`FG-1 → SA-1(2) → L-3(5)` 与 `FG-2 → SA-1(3) → L-3(5)`。
   用量表里 `(FG-1,L-3)=10`、`(FG-2,L-3)=15`，同时 `(SA-1,L-3)=5`
   作为子装配自身的展开行。
2. **叶件转装配件**：`POST /api/demo/promote-leaf`（UI 上「叶件转装配件」按钮）
   在一批中把 `L-4` 改为由 `L-6(3)`、`L-7(1)` 组成；`FG-1` 的 `L-4` 行消失，
   `L-6/L-7` 行出现，全部来自同一次提交。
3. **可暂停计算的交错修改**：演示模式下 `X-Demo-Pause` 头会让进程在展开阶段暂停
   （写锁尚未获取）。此时另一个进程可以提交；暂停方醒来后因 CAS 失败而重规划。
   可用 UI 的「写 A 侧 / 写 B 侧」按钮、`/api/demo/interleave/{a|b}`，或
   `scripts/concurrency_demo.py` 复现。
4. **溢出误拒（迟到批改按最新结构重算）**：`OV-A → OV-B(1e9) → OV-C(1e9)`；
   一边新增 `OV-C → OV-D(10)` 并在展开时暂停，另一边把 `OV-A → OV-B` 改为 `1`
   且先发布。迟到的一边在旧快照上会溢出，但会在最新结构上重算后**成功提交**；
   若最新结构仍溢出，则整批 422 拒绝、已发布版本保留。UI 按钮「溢出误拒样例」、
   `/api/demo/overflow/run`、`scripts/concurrency_demo.py` 场景 2
   与 `scripts/interleave_http.py` 均覆盖。

## HTTP API

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/api/state` | 当前版本的边、用量、版本号、上限 |
| POST | `/api/batch` | `{"ops":[...]}` 顺序执行；`replace` 为整体替换；`{"reset":true,"upserts":[...]}` 直接重建 |
| GET | `/api/leaf-usage?top=&leaf=` | 已落库用量（单快照） |
| GET | `/api/leaf-usage.csv?top=&leaf=` | 下载 CSV，带 `X-Bom-Version` |
| GET | `/api/usage-summary` | 只读汇总 |
| POST | `/api/demo/seed` / `/api/demo/promote-leaf` / `/api/demo/interleave/{a,b}` | 演示样例 |
| POST | `/api/demo/overflow/run` | 溢出误拒样例：迟到的批改重算后成功提交（返回两版本号与重试次数） |
| GET | `/api/demo/scenarios` | 样例说明 |

批改示例：

```json
{"ops": [
  {"op": "upsert", "parent": "FG-1", "child": "SA-9", "qty": 2},
  {"op": "delete", "parent": "FG-1", "child": "L-1"},
  {"op": "upsert", "parent": "SA-9", "child": "L-8", "qty": 3}
]}
```

错误：422 校验失败（非正整数、非法料号、超限）；409 环路（含 `paths`）或容量超限。

## 文件

```
app/store.py        SQLite 版本化存储、展开、判环、并发提交
app/main.py         FastAPI 应用工厂、API 与页面
app/demo.py         内置样例数据与交错修改用例
app/templates/      单页 UI
app/static/         前端脚本与样式
scripts/            两进程演示、HTTP 交错验证、种子任务、worker
tests/              pytest（64 项：展开、判环、原子性、并发、HTTP、UI）
docker-compose.yml  双 Web + 种子任务（+ 可选 interleave）
```

## 测试

```bash
pip install -r requirements.txt pytest httpx
pytest -q
```

覆盖：多路径求和、共享子装配、叶件转装配件、深 DAG 记忆化、整批判环与路径、
`replace`/`reset`/顺序语义、容量上限、展开失败与写入中途失败的回滚、跨实例持久化、
展开期间无写锁、迟到展开重规划、旧快照溢出/成环/超限在发布版本变化后的重算、
最新结构仍溢出时整批拒绝且保留已发布版本、并发合成环、读写快照一致性、CSV 单快照、
重启保留、两 Web 应用共库，以及模板/JS 元素引用一致性。
