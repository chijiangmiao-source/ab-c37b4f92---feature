# 星载姿控参数页 · ARIES 恢复审计服务

审计断电恢复：浏览器提交**至多 48 个崩溃页像**与**至多 128 条按 LSN 排列的受限 WAL
记录**（仅含 `begin/update/commit/abort/end/checkpoint`），服务严格执行 ARIES
三阶段——**分析**（自检查点建立脏页表 DPT 与事务表 ATT）、**重做**（自最小
recLSN，仅在 `pageLSN < LSN` 即确有缺失时重写）、**撤销**（失败事务沿
prevLSN 前驱链逆序，且只在页 LSN 条件满足时回写 before）——并冻结恢复裁决与
最终页摘要（恢复前后 pageLSN、SHA-256）。

断链、重复 LSN、错误前像、越界区间或已结束事务仍被引用时，一律 **HTTP 422
稳定拒绝**，同一 `auditId` 的旧成功证据会在同一事务中被清除替换。

## 快速开始

```bash
# 默认宿主机端口 8080
docker compose up --build

# 自定义宿主机端口
HOST_PORT=9090 docker compose up --build
```

- 浏览器打开 <http://localhost:8080>
- 健康响应：<http://localhost:8080/healthz> → `{"status":"ok",...}`

`verify` 容器等待 `web` 健康后依次执行：构建检查（py_compile）→
恢复规则单元测试（提交事务补写 / 失败事务回滚 / 损坏前驱链等）→
对 `web` 的 API/HTTP 冒烟，随后**以退出码结束**（0 表示全部通过）：

```bash
# 查看一次性校验结果
docker compose up --build verify
docker compose logs verify

# 或只重跑校验
docker compose run --rm verify
```

## HTTP API

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/healthz` | 健康响应 |
| GET | `/` | 浏览器审计页面 |
| GET | `/api/sample` | 内置演示场景 |
| POST | `/api/recover` | 提交页像 + WAL，返回冻结裁决或 422 拒绝 |
| GET | `/api/audit?auditId=...` | 读取该稳定审计标识的冻结裁决/拒绝 |
| GET | `/api/provenance?auditId=...&page=...&start=...&end=...` | 对**已冻结的成功裁决**追溯页内半开字节区间 `[start,end)` 的最终来源片段（只读） |

### 请求示例

```json
{
  "auditId": "AOC-AUDIT-20261004-01",
  "pages": [
    {"page": 7, "pageLSN": 30, "data": "<4096 字节的十六进制>"}
  ],
  "wal": [
    {"lsn": 10, "type": "begin", "xid": "T1"},
    {"lsn": 20, "type": "update", "xid": "T1", "prevLSN": 10,
     "page": 7, "offset": 0,
     "before": "0000000000000000", "after": "1111111111111111"},
    {"lsn": 30, "type": "checkpoint",
     "transactions": {"T1": 20}, "dirtyPages": {"7": 20}},
    {"lsn": 40, "type": "commit", "xid": "T1", "prevLSN": 20},
    {"lsn": 50, "type": "end", "xid": "T1", "prevLSN": 40}
  ]
}
```

裁决中逐条给出：分析归属、重做/撤销判据、恢复前后 pageLSN；每页给出恢复前后
SHA-256。已提交但未在崩溃页像中的更新被补写且保留；失败事务的更新先重做历史、
再逆序撤销，最终页中不会残留未提交字节。

### 字节来源追溯（`GET /api/provenance`）

审计员在冻结裁决中发现某个参数字节异常时，无需逐条手工比对 WAL：在浏览器成功
结果页填入**页号**与**半开字节区间**（或直接调用接口），即可得到覆盖该区间的
**连续来源片段**，每段给出 `[start,end)`、最终十六进制 `hex`、来源类别
`category`（`crash-image` 崩溃页像 / `redo` 被重做的更新 / `undo` 失败事务撤销
后的回写）以及关联事务 `xid` 与 LSN。

```
GET /api/provenance?auditId=AOC-AUDIT-DEMO-0001&page=7&start=0&end=32
```

追溯严格反映**最后一次实际写入**：只有真正写页的重做/撤销才构成证据，幂等
跳过的重做不会“认领”本属崩溃页像的字节；同一页上重做与撤销无论重叠、包含还是
仅相邻，结果都按请求范围切成**无重叠、无空洞、升序**的片段，边界处按请求边裁剪；
**相邻片段仅在来源证据（类别 + 事务 + LSN）完全相同时才合并**——因此即使两段
最终十六进制相同，只要一为 T1 的重做、一为 T2 撤销的回写，也会分开列示。未被
改写的字节明确归为 `crash-image`（`xid/lsn` 为 `null`）。

错误都是可操作的，且**绝不改写冻结裁决、不清除原有成功证据**（接口只读）：

| 状态码 | 情形 |
| --- | --- |
| 400 | 缺少参数、非整数、start ≥ end（空/反向区间）、`end > 4096` 越界 |
| 404 | 审计不存在（尚未成功建立），或该裁决中无此页（错误信息列出已知页号） |
| 409 | 该审计标识为稳定拒绝，不存在可追溯的成功证据 |

## 无 Docker 时本地运行 / 测试

仅依赖 Python 3.11 标准库：

```bash
python app/server.py                         # 启动服务
python -m unittest discover -s tests -v      # 规则测试
python tests/smoke_http.py                   # 自启服务并跑 API/HTTP 冒烟
```

## 目录结构

```
app/
  recovery.py    # ARIES 分析/重做/撤销引擎与全部稳定拒绝规则
  storage.py     # SQLite 冻结裁决存储（拒绝时清除旧成功证据）
  server.py      # http.server：页面 / API / healthz
  static/        # 浏览器 UI
tests/
  test_recovery.py  # 规则单元测试
  smoke_http.py     # API/HTTP 冒烟
  verify.sh         # verify 容器入口（三阶段）
Dockerfile
docker-compose.yml
```
