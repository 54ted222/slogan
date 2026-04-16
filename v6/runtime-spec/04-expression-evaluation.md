# 04 — Expression Evaluation

本文件定義 CEL 求值器的執行語意、namespace 解析、求值時機與錯誤處理。對應 [dsl-spec/04-expressions.md](../dsl-spec/04-expressions.md)。

---

## CEL Engine

- 採用 **CEL 1.0** 語意（cel.dev），`${ }` 為定界符；定界符外為字面字串。
- 字串中多個 `${ }` 之間的字面字串視為串接（type coerce 至 string）。
- 單一 `${ }` 全字串時保留原型別，不強制轉字串。

### 標準函式集

引擎 MUST 註冊：

| 類別 | 函式 |
|------|------|
| string | `contains` / `startsWith` / `endsWith` / `matches` / `size` |
| list   | `exists` / `all` / `filter` / `map` / `size` / `append` / `concat` / `slice` / `join` |
| util   | `now()` / `uuid()` / `json_encode` / `json_decode` / `default` / `coalesce` / `has` |
| numeric coerce | `int(x)` / `double(x)` / `string(x)` |

`now()` 與 `uuid()` 的副作用（replay 讀回）詳見 `08-persistence.md` 的「Replay」章節。

`s.extractMatches(regex)` 為引擎擴充：回傳完整匹配組的字串陣列，無匹配回 `[]`。

---

## Namespace 結構

求值上下文是一個 map，依 step type 與所在位置動態組裝：

```
{
  input:     map | null,         # workflow / function instance 的初始 input
  steps:     map<step_id, StepRef>,
  prev:      StepRef | null,     # 語法上的前一步
  vars:      map,                # 所在 instance 的 vars
  loop:      LoopFrame | null,
  event:     EventCtx | null,    # 僅 trigger / wait / wait 後續
  error:     ErrorCtx | null,    # 僅 catch 內
  callback:  CallbackCtx | null, # 僅 callback handler 內
  context:   ToolContext | null, # 僅 tool backend resolve 階段
  env:       map<string, string>,
  secret:    SecretAccessor,     # 解密延後到實際讀取時
  project:   map,
  artifacts: ArtifactsCtx,
}
```

### StepRef

```
{
  status: "SUCCEEDED" | "FAILED" | "SKIPPED",
  output: any,
  error:  ErrorCtx | null,        # 僅 FAILED 時
  started_at: timestamp,
  ended_at:   timestamp,
}
```

### LoopFrame

```
{ item: any, index: int, length: int }
```

巢狀 loop 時 `loop` 永遠指最內層 frame；外層需以具名 step 或 `assign` 保留。

---

## 求值時機

| 欄位 | 求值點 |
|------|--------|
| `when` | step 進入時，先於其他欄位 |
| `input` / `args` / `data` / `headers` / 任何字面值欄位 | step 進入 RUNNING 之後、handler 執行之前；一次性求值，求值結果 snapshot 至 step state |
| `match` (wait signals 事件訊號) | event 到達時 |
| `match` (trigger when) | trigger event 到達時，於 instance 建立前 |
| `condition` (if) | if step 進入時，於共通 `when` 通過後 |
| `subject` (switch) | switch step 進入時，於共通 `when` 通過後 |
| `cases[].value` | switch 對應 case 被檢查時（lazy） |
| `items` / `count` (foreach) | foreach 開始時 |
| `concurrency` / `failure_policy` 等控制旗標 | foreach / parallel 開始時 |
| `delay` (emit) | emit step 進入時 |
| `timeout` / `retry.delay` / `backoff` | step 進入時，每次 attempt 重新求值 |
| duration 欄位（`timeout` / `delay` / `duration` / `max_delay`）| CEL 求值後：若非 string → `expression_error.type_error`；若 string 但不符 Duration 格式（見 `dsl-spec/01-overview.md`）→ step FAILED，`error.type: "invalid_duration_format"` |
| `compensate.input` (saga) | 補償觸發時，可引用原 step 的 `output` |
| handler `output` (callback handler return) | handler return 時 |
| Tool backend 的 `command` / `args` / `env` / `stdin.template` / `stdout.mapping` 等 | 每次 tool 呼叫前 |
| Lifecycle `init.backend` 內欄位 | init 觸發時，僅 secret/env/project/artifacts 可用 |

---

## 求值流程

```
def evaluate(expr_string, ctx) -> Value:
    ast = parse(expr_string)
    try:
        return run(ast, ctx)
    except IdentifierError as e:
        raise ExpressionError(type="identifier_not_found", message=e.path, fragment=expr_string)
    except TypeError as e:
        raise ExpressionError(type="type_error", message=str(e), fragment=expr_string)
    except OverflowError as e:
        raise ExpressionError(type="overflow", message=str(e), fragment=expr_string)
```

呼叫端責任：

- `when` 求值：catch ExpressionError → step FAILED, `error.type == "expression_error"`
- 其他欄位求值：同上
- handler return output 求值失敗：handler step FAILED；error 向上至 callback step

---

## Namespace 寫入規則

- `input` / `event` / `error` / `callback` / `context` / `project` / `env` / `secret` / `artifacts`：read-only，呼叫端不可修改。
- `vars`：僅 `type: assign` 可寫；其他 step 拋寫入錯誤。
- `steps`：僅 step 終態時由 Engine Loop 寫入。
- `loop`：foreach 進入時 push、退出時 pop；不可由使用者寫。
- `prev`：每進入新 step 時由 Engine Loop 重綁；無使用者寫入。

引擎不允許 expression 產生 namespace 副作用；`now()` / `uuid()` 是純值產生器，但其結果為了 replay 一致性會被記錄（見 `08-persistence.md` 的「Replay」章節）。

---

## Secret 求值

`secret.X` 不在 namespace 物件中持有明文：SecretAccessor 是延遲解密的代理。

- CEL 對 `secret.X` 的存取觸發 SecretAccessor.get("X") → Resource Pool 解密 → 回傳明文 string。
- 命中後的明文 SHOULD 在 expression scope 結束後立即清除（best-effort）。
- log / trace 中 `secret.*` 的值 MUST 被遮蔽為 `"***"`。

---

## Template 求值

Tool backend 的 `stdin.template`（format: text）使用相同 CEL `${ }` 規則，但求值上下文限縮：

```
{ input, env, secret, context }
```

無 `steps` / `prev` / `vars` 等。

`stdout.mapping` 的求值上下文：

```
{ raw, input }
```

`raw` 型別依 `stdout.format` 決定（string / any / list<string>）。

---

## 求值錯誤碼總表

| `error.type` | 觸發 |
|--------------|------|
| `expression_error` | 任何 CEL 異常（identifier / type / overflow） |
| `expression_error.identifier_not_found` | 引用不存在的 namespace 路徑 |
| `expression_error.type_error` | 函式 / 運算子型別不符 |
| `expression_error.template_eval` | template 字串組裝失敗 |
| `expression_error.mapping` | stdout.mapping 求值失敗 |

呼叫端可選擇將子型別併為頂層 `expression_error`；CEL fragment 一律放入 `error.message`。

---

## 多個 `${ }` 字串組裝

```
"order ${ steps.x.output.id } total ${ steps.y.output.amount }"
```

組裝步驟：

```
1. Parse 字面字串為 segments：[literal, expr, literal, expr, ...]
2. 對每個 expr 求值
3. 將所有 expr value 透過 string() 強制轉字串（非字串型別 → string(value)）
4. concatenate
```

任一 segment 求值失敗 → 整段失敗；不做部分組裝。

---

## 求值不變式

- **純函式性**：相同 namespace + 相同 expr → 相同結果，唯獨 `now()` / `uuid()` 例外（記錄至 execution log，replay 時讀回）。
- **無 I/O**：CEL 不能觸發 HTTP / file / database 呼叫；必要時須以 step 形式呼叫 builtin tool。
- **深度限制**：表達式 AST 深度 SHOULD 限制（建議 64），防 stack overflow；過深 → `expression_error.too_deep`。
- **大小限制**：單次求值產生的 string / list / map 大小 SHOULD 限制（建議 1MB），超過 → `expression_error.too_large`。
