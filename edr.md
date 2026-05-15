# EDR 筆記

## 1. EDR 是什麼

`EDR` 是 `Endpoint Detection and Response`，可以先理解成：

- 裝在端點上的安全監控系統

這裡的端點通常是：

- 員工電腦
- 筆電
- 伺服器
- 虛擬機

它在做的事很簡單：

- 看這台電腦發生了什麼
- 判斷這件事正不正常
- 必要時產生警報

---

## 2. Wazuh 和 OSSEC 是什麼

- `OSSEC` 是比較早期的主機型安全監控系統
- `Wazuh` 是從 `OSSEC` 發展出來的安全平台
- 可以先把 `Wazuh` 想成比 `OSSEC` 功能更多、架構更完整的版本
- 核心沿用了很多 `OSSEC` 的舊架構，所以底層和檔名常會看到 `ossec`

常見例子：

- `ossec.conf`
- `/var/ossec/`
- `manage_agents`

一句話記：

- `Wazuh` 是建立在 `OSSEC` 基礎上的延伸版本

---

## 3. Server 和 Agent

| 角色 | 白話理解 | 主要工作 |
| --- | --- | --- |
| `Server` / `Manager` | 中央控制端 | 收資料、分析資料、套規則、管理 agent |
| `Agent` | 裝在被監控主機上的程式 | 蒐集主機資訊並送回 server |

最簡單的記法：

- `Agent` 收資料
- `Server` 做判斷

你課堂上看到的情況大概是：

- 一台電腦是 `server`
- 另一台電腦裝了 `agent`
- 指令在 `server` 下
- 動作或回報發生在 `agent` 那台主機

所以重點是：

- `server` 負責管理與分析
- `agent` 負責現場收集與回報

---

## 4. Wazuh 大致怎麼運作

可以先把整個流程記成：

1. `Agent` 收到 log 或事件
2. 資料透過安全連線送到 `Server`
3. `Server` 依設定先決定哪些資料要收
4. `decoder` 把資料拆成欄位
5. `XML rule` 根據欄位做判斷
6. 命中後產生 alert
7. 必要時再去查外部 API

你可以把它背成三層：

- 第一層：`config`
- 第二層：`decoder`
- 第三層：`rule`

各層大意：

- `config`：先決定哪些 log / event 要收進來，算是第一階段過濾
- `decoder`：把原始事件拆成欄位
- `rule`：根據欄位判斷這代表什麼事件

### 要特別分清楚：加密 和 decoder 不是同一件事

- `加密`：是 agent 和 server 傳輸資料時的保護，避免通訊內容被直接看懂
- `decoder`：不是拿來解密通訊，而是把收到的 log 內容拆成 Wazuh 能判斷的欄位

所以比較正確的理解是：

1. agent 把資料透過安全通道送到 server
2. server 收到後，`decoder` 再去解析內容
3. 解析出來的欄位交給 `rule` 判斷

---

## 5. 規則判斷的兩種方式

課堂上提到的做法可以先分成兩種：

### 1. 基於規則比對

這是 Wazuh 很核心的做法。

- 用本地的 `XML rules` 來判斷事件
- 速度快
- 結果固定
- 通常能直接得到對應的 `rule level`

### 2. 基於外接 API

這比較像補充查詢或情報 enrichment。

- 拿事件裡的 `hash`、`IP`、`domain` 去查外部服務
- 用來補更多背景資料

課堂的意思可以先記成：

- 通常先做本地規則比對
- 沒有比對到，或想查更詳細資訊時，再去查外部 API

---

## 6. XML 規則怎麼看

你可以把每一條 XML 規則都當成一個 `if`。

在看這段前要先記一件事：

- `decoder` 做的是「解析欄位」
- 不是「解密通訊」

常見標籤意思：

- `<rule id="...">`：這是一條規則，`id` 是規則編號
- `<if_sid>`：前提是某條前置規則先命中
- `<field name="...">`：某個欄位要符合指定值
- `<match>` / `<regex>`：比對文字或模式
- `<description>`：命中後顯示的說明
- `<group>`：這條規則屬於哪一類

白話讀法：

- 如果前面的條件成立
- 而且某個欄位符合條件
- 就觸發這條規則

示意：

```xml
<rule id="100100" level="5">
  <if_sid>base_rule_id</if_sid>
  <field name="win.system.eventID">^4625$</field>
  <description>Windows failed logon</description>
  <group>authentication_failed,</group>
</rule>
```

這段的意思就是：

- 前置規則先命中
- 而且事件欄位 `win.system.eventID` 是 `4625`
- 就把它判成登入失敗事件

---

## 7. Event ID 和 Rule ID 不一樣

這裡很容易混掉。

- `event ID`：原始事件自己的編號，例如 Windows event ID
- `rule ID`：Wazuh 規則自己的編號

所以像你上課聽到的：

- 密碼錯誤
- 得到一個 id
- 去對照 XML

比較正確的理解是：

1. 原始事件先進來
2. `decoder` 把 `event ID`、`user`、`srcip`、`status` 等欄位拆出來
3. `XML rule` 根據這些欄位做比對
4. 命中後，Wazuh 才產生自己的 `rule ID` 和描述

重點：

- 不是只看一個 `id`
- 是看整筆事件拆出來的欄位有沒有符合規則

---

## 8. CDB 是什麼

`CDB` 可以先理解成：

- 快速資料庫比對
- 很適合做清單查詢

常拿來比對：

- `hash`
- `IP`
- `user`
- `domain`

流程通常是：

1. 事件先被 decode 出欄位
2. XML 規則用 `<list>` 去查 `CDB list`
3. 如果值存在於清單裡，就觸發規則

例如 `md5` 比對：

```xml
<rule id="110002" level="13">
  <if_sid>554, 550</if_sid>
  <list field="md5" lookup="match_key">etc/lists/malware-hashes</list>
  <description>File with known malware hash detected: $(file)</description>
</rule>
```

白話就是：

- 先前的檔案事件規則已命中
- 再去查這個檔案的 `md5`
- 如果它出現在 `malware-hashes` 清單裡
- 就觸發惡意檔案警報

CDB 清單大概長這樣：

```text
e0ec2cd43f71c80d42cd7b0f17802c73:mirai
55142f1d393c5ba7405239f232a6c059:Xbash
```

---

## 9. 外接 API 和橋接腳本

如果要查外部 API，通常要寫一個橋接腳本。

原因是 Wazuh 不會自動知道：

- 要把哪個欄位送出去
- 要送到哪個 API
- API 怎麼認證
- 回來的資料要怎麼處理

所以中間通常需要：

- `integration script`
- 也就是你說的橋接腳本

它的工作大致是：

1. 接收 Wazuh alert
2. 取出需要的欄位，例如 `hash`、`srcip`、`domain`
3. 呼叫外部 API
4. 把結果整理後回傳或寫成新的事件

---

## 10. 自訂規則放哪裡

- 內建規則在 `/var/ossec/ruleset/`
- 自訂規則通常放在 `/var/ossec/etc/rules/local_rules.xml`
- 自訂 decoder 通常放在 `/var/ossec/etc/decoders/local_decoder.xml`

如果只是自己做實驗，通常改 `etc` 下面，不要直接改內建 ruleset。

### 自訂規則的 Rule ID

這點也要分清楚：

- 不是所有 Wazuh / OSSEC 規則都要 `100000` 以上
- 是你自己新增的 `custom rule`，建議用 `100000` 到 `120000`

最簡單的記法：

- `自己寫的 rule ID，從 100000 開始最安全`

---

## 11. 課堂上可能看到的指令

### `manage_agents`

用途：

- 新增 agent
- 列出 agent
- 匯入或匯出 key
- 管理哪些 agent 可以和 server 溝通

白話就是：

- 管理 agent 身分的工具

### `agent_control`

用途：

- 查 agent 狀態
- 對指定 agent 下管理指令

例如：

```bash
/var/ossec/bin/agent_control -l
```

通常是在看：

- 有哪些 agent
- 它們有沒有連上 server

---

## 12. 為什麼是 Wazuh 卻一直看到 ossec

這是正常的。

因為 `Wazuh` 延續了很多 `OSSEC` 的歷史架構，所以你會常看到：

- `ossec.conf`
- `/var/ossec/`
- `ossec` 相關工具名稱

例如：

```text
/var/ossec/etc/ossec.conf
```

這不代表你裝錯，它本來就會這樣。

---

## 13. 最後快速記憶

- `Wazuh` 是從 `OSSEC` 發展出來的
- `Agent` 負責收資料，`Server` 負責分析與判斷
- `config` 先過濾，`decoder` 拆欄位，`rule` 做判斷
- `XML rule` 很像一條一條的 `if`
- `CDB` 是給規則快速查表用的
- 外接 API 通常要靠橋接腳本
- 自訂規則通常放在 `local_rules.xml`
- 自訂 `rule ID` 建議從 `100000` 開始
