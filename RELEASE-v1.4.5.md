# forward-panel v1.4.5 — 规则「已消耗流量」显示(nyanpass 口径)

## 背景

规则表此前只显示**净荷**(入口上报的 in/out,实际转发量)。但用户配额扣减的是
**计费量 = 净荷 × 倍率**(入口腿 × 入口倍率 + 出口腿 × 出口倍率)。两者口径不同,
代理商/终端用户无法从规则表直接看出"这条规则到底扣了多少配额"。

本版按 **nyanpass 官方口径**(`reference/forward_rule` 倍率说明)补齐显示:
**净荷(实际使用) + 已消耗流量(计费量,与配额扣减一字不差)** 双列并存。

> 计量层、倍率层、扣费逻辑**零改动**。本版**纯加一个显示维度**。倍率怎么设、亏不亏,
> 由商家按 IDC 成本自行配置(市场惯例),不在产品内。

## 改动

### 计量(master 侧,与 `accrue` 同步累计真实计费量)
- 新增内存累计 `ruleBilled map[string]int64`(ruleID → 累计已消耗)+ `rbDirty` 脏标,
  全程在 `accrue` 已持的 `billMu` 下,无嵌套锁。
- `accrue` 在三处 `billPend +=` 点(owner 兜底 1× / 入口腿 / 出口腿)同步
  `ruleBilled[rid] += <同一计费增量>`。**不改任何现有计费数值**,只是把"已经在算的钱"
  额外按规则归集一份用于显示。

### 持久化(镜像 `rule_traffic`,独立小表)
- 新表 `rule_billed(rule_id, billed_bytes)`,`migrateRuleTraffic` 建表。
- `SaveRuleBilled` / `AllRuleBilled` / `DeleteRuleBilled`。
- `loadRuleBilled`(启动回灌,重启不丢)/ `flushRuleBilled`(随采样循环定时落库)/
  `forgetRuleBilled`(删规则清掉)——与 `loadRuleTraffic` / `flushRuleTraffic` /
  `forgetRuleTraffic` 一一对应、同位置调用。

### API
- 规则列表新增 `billed_bytes` 字段(在独立 `billMu` 临界区读 `ruleBilled`,不与 `rtMu` 嵌套)。

### 前端
- 规则表「流量」列表头改为 **流量 净荷·已消耗**;
- 单元格在净荷 `in / out` 下补一行 **已消耗 <计费量>**(hover 提示"= 净荷×倍率,与配额扣减一致")。

## 兼容性

- **proto 不变**(协议版本不动),**agent 无需更新**。本版纯 master + 前端。
- 老数据库自动建 `rule_billed` 表;历史规则首次显示已消耗从 0 起累计(不回算历史,
  与 `rule_traffic` 当初上线一致)。
- 老规则在 `billed_bytes` 缺省时前端按 0 显示(`fmtBytesT` 已做 `Number||0`)。

## 验证要点(test2)

1. 跑一条**双向**规则一段流量,核对:
   `已消耗 ≈ 净荷(in+out) × (入口倍率 + 出口倍率) / 100`,
   且与该用户 `used_bytes` 的增量对得上。
2. 改该线路/组**倍率**后继续跑,已消耗的**增量**应按新倍率变化(历史累计不动)。
3. 自托管/单向(免计)规则:被免掉的那条腿不计入已消耗(与扣费一致)。
4. master 重启后,已消耗累计从库回灌、不归零。
5. 删规则后 `rule_billed` 对应行清除。
