# PayPaz 渠道 · 合作伙伴 · 佣金分配系统 设计文档

版本：v0.1
状态：设计讨论已定稿，正在配套原型

## 0. 背景

管理后台商户详情里已经有一个"合作伙伴"标签页，但目前只是一个只读统计报表，背后是一个写死的三值枚举（`P01`=直连、`P02`=Meta Trader、`P03`=On Ramp），没有真正的渠道/合作伙伴数据结构，也没有任何返佣/佣金相关的代码——这是一个从零设计的新系统，只有"充值地址按来源分流"这一条底层机制可以直接复用（`SubWalletAddress` 按 `partnerCode` 区分，同一客户可以有多个地址）。

## 1. 核心概念

- **渠道（Channel）**：比如"Meta Trader"，是一类合作来源的统称。每个渠道配置一套自己的充提币收费规则（替换而非叠加基础费率）。
- **合作伙伴（Partner）**：挂在某个渠道下的具体个人/机构，有名称、联系方式，预留钱包地址字段（供未来自动打款用，本期不接真实打款）。
- **商户的渠道配置**：为某个商户（broker）选择一个渠道 + 该渠道下的若干合作伙伴，为每个合作伙伴单独设置返佣比例。同一商户下所有合作伙伴的返佣比例之和，不能超过渠道收费规则产生的手续费金额。

## 2. 三类收费规则的边界

系统里的充提币收费规则分散在三处配置，互不覆盖：

| 配置位置 | 适用对象 | 与本系统的关系 |
|---|---|---|
| 「代币」菜单 | 没有配置任何渠道的自然流量商户 | 默认规则，本系统不改动 |
| 「On Ramp」菜单 | On Ramp 业务的商户+币对 | 独立计费，本系统不改动 |
| 「合作伙伴」菜单（本系统新增） | 配置了渠道的商户，其渠道来源的客户 | 本系统新增的收费规则，返佣从这里面出 |

三个菜单页面顶部都应带一条一致的提示条，注明"系统收费规则分三处配置"，当前页高亮，另外两处可点击跳转，避免运营人员找错地方改错费率。

## 3. 商户识别客户来源的机制（复用真实机制，不新增）

- 商户不配置渠道 = 自然流量：客户使用系统默认充值地址，走「代币」菜单的基础费率。
- 商户配置了渠道 + 合作伙伴：系统为该商户名下产生渠道来源客户的**新增一个专属充值地址**（复用真实的 `SubWalletAddress.partnerCode` 分流机制）。同一个客户即使既走过自然流量、也走过渠道，也会对应两个不同地址，系统靠到账地址不同区分资金来源，从而决定这笔充值该按哪套规则收费、要不要产生佣金。
- 商户完成渠道配置后，OpenAPI 同步为该商户开通这个渠道维度的调用权限。

## 4. 佣金计算与调整

### 4.1 何时产生佣金记录

订单完成"充值手续费"或"提币手续费"的扣除动作后，系统**实时**计算并生成佣金记录——不设系统内的等待期。佣金的实际下发由运营人工处理，处理节奏由线下把控，不在系统管控范围，这天然形成一个"运营还没处理之前，出问题还能追回"的缓冲窗口，但这个窗口不是系统逻辑的一部分。

### 4.2 佣金记录的字段与快照原则

每一条佣金记录都要快照当时生效的信息，不随后续配置变化而改变：

```
CommissionEntry:
  id
  orderId, orderType         -- 关联回具体订单
  merchantId
  channelId, partnerId
  feeAmount                  -- 该订单当时产生的渠道手续费金额（快照）
  rebateRate                 -- 该合作伙伴当时生效的返佣比例（快照）
  amount                     -- 本条金额，正常为正数，调整记录可以是负数
  entryType                  -- ORIGINAL | ADJUSTMENT
  adjustReasonOrderId        -- 仅调整记录：触发这次调整的原始事件（如退款）
  status                     -- PENDING | PROCESSED
  processedBy, processedAt
  createTime
```

商户后续更换渠道、更换合作伙伴、调整返佣比例，都只影响新订单的计算依据，不会修改任何历史记录——每笔订单的佣金永远只看"下单那一刻"的配置快照。

### 4.3 退款导致手续费变化时，如何调整佣金

佣金调整是自动触发的，不需要人工重新计算：

1. 某订单已扣除的"充值/提币手续费"因退款等原因发生变化（无论具体退款流程如何触发）。
2. 系统找出这笔订单已经生成过的全部佣金记录（可能对应多个合作伙伴）。
3. 用**该订单当时快照的返佣比例**（不用现在最新比例），按新的手续费金额重新算出"应得佣金"。
4. 应得佣金减去该订单迄今为止已产生的佣金记录总和（原始 + 历次调整），得到差额。
5. 差额不为零时，为每个受影响的合作伙伴各追加一条 `ADJUSTMENT` 记录，金额可正可负，明确关联回触发事件，原始记录永不修改。
6. 运营在佣金处理界面看到的应该是"某订单/某合作伙伴名下原始+全部调整记录加总后的净额"，而不是要求运营自己心算历次调整。

### 4.4 佣金下发（本期）

本期佣金分配是**纯记账**，不接真实打款：

- 运营可以把某条（或某商户/某合作伙伴名下的净额）标记为「已处理」，仅作为流程追溯用，不触发任何资金动作。
- 已标记「已处理」的历史记录，即使后续对应订单被调整产生了新的 `ADJUSTMENT` 记录，原「已处理」记录本身也不会被改动或撤销状态——运营看到新的调整记录后自行判断是否需要线下处理。
- `Partner.walletAddress` 字段预留，供未来做佣金审批+自动打款时使用，本期只做展示/编辑，不接执行动作。

## 5. 数据库设计草案（增量，供原型参考）

```sql
CREATE TABLE channel (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    name            VARCHAR(64)  NOT NULL COMMENT '如 Meta Trader',
    deposit_fee_rate  DECIMAL(8,4) NOT NULL COMMENT '该渠道客户的充值手续费率，替换基础费率',
    withdraw_fee_rate DECIMAL(8,4) NOT NULL COMMENT '该渠道客户的提现手续费率，替换基础费率',
    status          TINYINT NOT NULL DEFAULT 1,
    create_time     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
) COMMENT '渠道';

CREATE TABLE partner (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    channel_id      BIGINT NOT NULL,
    name            VARCHAR(64) NOT NULL,
    contact         VARCHAR(128) NULL COMMENT '联系方式',
    wallet_address  VARCHAR(128) NULL COMMENT '预留字段，供未来自动打款使用，本期仅展示',
    status          TINYINT NOT NULL DEFAULT 1,
    create_time     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    KEY idx_channel (channel_id)
) COMMENT '渠道下的合作伙伴';

CREATE TABLE merchant_channel_partner (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    merchant_id     BIGINT NOT NULL,
    channel_id      BIGINT NOT NULL,
    partner_id      BIGINT NOT NULL,
    rebate_rate     DECIMAL(8,4) NOT NULL COMMENT '该商户下，该合作伙伴的返佣比例；同商户同渠道下所有合作伙伴之和 <= 渠道手续费产生的金额',
    status          TINYINT NOT NULL DEFAULT 1,
    create_time     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    KEY idx_merchant (merchant_id)
) COMMENT '商户-渠道-合作伙伴返佣配置（当前生效值，订单产生时被快照进commission_entry）';

CREATE TABLE commission_entry (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id        VARCHAR(64) NOT NULL,
    order_type      VARCHAR(16) NOT NULL COMMENT 'DEPOSIT / WITHDRAW',
    merchant_id     BIGINT NOT NULL,
    channel_id      BIGINT NOT NULL,
    partner_id      BIGINT NOT NULL,
    fee_amount      DECIMAL(24,8) NOT NULL COMMENT '产生本条记录时该订单的渠道手续费金额快照',
    rebate_rate     DECIMAL(8,4) NOT NULL COMMENT '产生本条记录时的返佣比例快照',
    amount          DECIMAL(24,8) NOT NULL COMMENT '本条佣金金额，调整记录可为负',
    entry_type      TINYINT NOT NULL COMMENT '1原始 2调整',
    adjust_ref_order_id VARCHAR(64) NULL COMMENT '仅调整记录：触发调整的事件/订单',
    status          TINYINT NOT NULL DEFAULT 0 COMMENT '0待处理 1已处理',
    processed_by    VARCHAR(64) NULL,
    processed_at    DATETIME NULL,
    create_time     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    KEY idx_order (order_id),
    KEY idx_partner (partner_id),
    KEY idx_merchant (merchant_id)
) COMMENT '佣金流水，只追加不修改';
```

## 6. 待细化 / Open Questions

- 佣金处理页面展示"净额"时，是按合作伙伴汇总，还是也支持按商户、按渠道、按时间段切片查看——原型先做按合作伙伴的净额视图，其余切片作为后续增强。
- 商户可以同时挂几个合作伙伴，原型先不设上限，只做"总和不超过渠道手续费金额"的校验。
- 渠道下线（合作伙伴不再合作）后，历史佣金记录、历史地址如何处理——本期不涉及，只做"渠道/合作伙伴可停用，停用后不再产生新配置"。
