# Cell 协议 v1

[TOC]
## 协议符号约定

本文档以下内容都会采用统一的结构描述一个 cell：

```
lock: ...
type: ...
data: ...
witness: ...
```

其中 `lock, type, data` 都是每个 cell 必定包含的信息，从 RPC 接口返回的数据结构中也可以看到，而 `data` 就是这笔交易中与 cell 对应的 `outputs_data` 。`witness` 比较特殊，它和 cell 之间是没有关联关系的，所以这里的 `witness` 特指 DAS witness ，而且仅仅指 DAS witness 中的 `entity` 部分，因为 DAS witness 中存放了它自己对应哪个 cell 的相关信息，所以才有了关联关系，详见[数据存储方案.md](数据存储方案.md) 。

**data 中所有的字段名意味着一段按照特定偏移量解析的数据**，因为 data 的体积会影响需要质押的 CKB 数量，所以其中除了按照文档中给出的偏移量来切分数据外本身没有任何数据结构。**witness 中所有的字段名意味着一个 molecule 编码的数据结构**，首先需要使用对应结构的结构体/类去解析数据，然后才能访问对应字段。

在描述 cell 结构时可能看到以下符号：

|        符号         |                                                                      说明                                                                       |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| lock: <...>         | 代表一个特定的 script ，其 code_hash, args, hash_type 都有简单的约定，所以就不列举明细了                                                        |
| type: <...>         | 同上                                                                                                                                            |
| hash(...)           | 指代这里存放的数据是通过什么计算得出的 hash 值                                                                                                  |
| ======              | 在描述 cell 结构的代码段类，此分隔符意味着下面的内容是对特定 molecule 结构的详细介绍，但最新的 schema 轻易 das-types 仓库中的 schemas/ 目录为准 |
| ConfigCellXXXX.yyyy | 指代数据需要去某个 ConfigCell 的 witness 中的特定字段获取，详见[ConfigCell](#ConfigCell)                                                        |

## 数据结构

### ApplyRegisterCell

申请注册账户，用户在真正执行注册前必须先创建此 Cell 进行申请，然后等待 `ConfigCellApply.apply_min_waiting_block_number` 时间后才能使用此 Cell 进行注册。这样设计的目的是为了防止用户注册账户的交易在上链的过程中被恶意拦截并被抢注。

#### 结构

```
lock: <ckb_lock_script>
type: <apply-register-cell-type>
data:
  hash(lock_args + account) // account 包含 .bit 后缀
  height // cell 创建时的区块高度(小端)
```

这个 Cell 中只有一个单纯的 hash 值。

#### 体积

`134` Bytes


### PreAccountCell

当 [ApplyRegisterCell](#ApplyRegisterCell) 在链上存在超过 `ConfigCellApply.apply_min_waiting_time` 时间之后，用户就可以将其转换为一个 PreAccountCell ，等待 Keeper 通过创建 [ProposalCell](#ProposalCell) 提案将其最终转换为 [AccountCell](#AccountCell) 。

#### 结构

```
lock: <always_success>
type: <pre-account-cell-type>
data:
  hash(witness: PreAccountCellData)
  id // account ID，生成算法为 hash(account)，然后取前 20 Bytes。这个包含 .bit 后缀

witness:
  table Data {
    old: table DataEntityOpt {
        index: Uint32,
        version: Uint32,
        entity: PreAccountCellData
    },
    new: table DataEntityOpt {
      index: Uint32,
      version: Uint32,
      entity: PreAccountCellData
    },
  }

======
table PreAccountCellData {
    // Separate chars of account.
    account: AccountChars, // 不带 .bit
    // If the PreAccountCell cannot be registered, this field specifies to whom the refund should be given.
    refund_lock: Script,
    // If the PreAccountCell is registered successfully, this field specifies to whom the account should be given.
    owner_lock_args: Bytes,
    // The lock script of inviter,
    inviter_id: Bytes,
    inviter_lock: ScriptOpt,
    // The lock script of channel,
    channel_lock: ScriptOpt,
    // Price of the account at the moment of registration.
    price: PriceConfig,
    // The exchange rate between CKB and USD.
    quote: Uint64,
    // The discount rate for invited user
    invited_discount: Uint32,
    created_at: Timestamp,
}

vector AccountChars <AccountChar>;

table AccountChar {
    // Name of the char set which the char belongs.
    char_set_name: Uint32,
    // Bytes of the char.
    bytes: Bytes,
}
```

- account，用户实际注册的账户名，不包含 `.bit` 后缀；
- refund_lock，假如 PreAccountCell 最终无法通过提案时，退款的 lock 脚本，即地址；
- owner_lock_args，假如 PreAccountCell 最终通过提案时，[AccountCell.lock.args](#AccountCell) 的值，即 das-lock 的  args；
- inviter_id: 主要目的方便服务端展示邀请者名称；
- inviter_lock，邀请者的 lock script，利润分配会被转入 IncomeCell 中并以此 lock script 记账；
- channel_lock，渠道商的 lock script，利润分配会被转入 IncomeCell 中并以此 lock script 记账；
- price，账户注册时的售价；
- quote, 账户注册时的 CKB 的美元价格；
- created_at，PreAccountCell 创建时 TimeCell 的时间；

#### 利润以及注册所获时长的计算逻辑

创建 PreAccountCell 时，用户就需要支付注册费以及创建各种 Cell 所需的基础费用，此时用户应支付 CKB 数量的计算公式为：

```
// 这一段是伪代码，存在从上往下执行的上下文环境
存储费 =  (AccountCell 基础体积 + 账户长度 + 4) * 100_000_000

利润 = PreAccountCell.capacity - 存储费

if 美元年费 < CKB 汇率 {
  CKB 年费 = 美元年费 * 100_000_000 / CKB 汇率
} else {
  CKB 年费 = 美元年费 / CKB 汇率 * 100_000_000
}

CKB 年费 = CKB 年费 - (CKB 年费 * 折扣率 / 10000) // 折扣率是以 10000 为底的百分数

注册时长 = 利润 * 365 / CKB 年费 * 86400
```

- 年份需要大于等于 1 ，实际计算时按照 365 \* 86400 秒为一年来计算；
- **账户注册年费** 保存在 [ConfigCellPrice.prices](#ConfigCell) 中，单位为 **美元**；
- **CKB 汇率**从 QuoteCell 获取，单位为 **美元/CKB**，前面在[数据存储方案.md](#数据存储方案.md)的**美元的单位**一节我们约定了 1 美元记为 `1_000_000` ，因此如果 QuoteCell 中记录的是 `1_000` 那么也就意味着 CKB 汇率就是 `0.001 美元/CKB`；
- **AccountCell 基础成本** 只在调整 Cell 数据结构时会发生变化，可以认为是固定的常量，查看对应 Cell 的 **体积** 即可获得；
- **Account 字节长度**，由于 AccountCell 会在 data 字段保存完整的账户，比如 `das.bit` 那么保存的就是 `0x6461732E626974` ，因此需要再加上这部分体积；
- 带除法的都是自动取整

####  体积

`116` Bytes


### ProposalCell

用户创建 [PreAccountCell](#PreAccountCell) 之后，就需要 Keeper 将它们收集起来发起提案，也就是创建 ProposalCell ，只有当提案等待一定时间后才能通过，所谓通过就是创建一笔交易消费提案，并将其应用的 [PreAccountCell ](#PreAccountCell) 转换为最终的 [AccountCell](#AccountCell)。这一过程确保了账户名在链上的唯一性。

#### 结构

```
lock: <always_success>
type: <proposal-cell-type>,

data:
  hash(witness: ProposalCellData)

witness:
  table Data {
    old: None,
    new: table DataEntityOpt {
      index: Uint32,
      version: Uint32,
      entity: ProposalCellData
    },
  }

======
table ProposalCellData {
    proposer_lock: Script,
    created_at_height: Uint64,
    slices: SliceList,
}

vector SliceList <SL>;

// SL is used here for "slice" because "slice" may be a keyword in some languages.
vector SL <ProposalItem>;

table ProposalItem {
  account_id: AccountId,
  item_type: Uint8,
  // When account is at the end of the linked list, its next pointer should be None.
  next: AccountId,
}

====== 举例来说看起来就像下面这样
table ProposalCellData {
  proposer_lock: Script,
  slices: [
    [
      { account_id: xxx, item_type: exist, next: xxx },
      { account_id: xxx, item_type: new, next: xxx },
    ],
    [
      { account_id: xxx, item_type: proposed, next: xxx },
      { account_id: xxx, item_type: new, next: xxx },
      { account_id: xxx, item_type: new, next: xxx },
    ],
    [
      { account_id: xxx, item_type: exist, next: xxx },
      { account_id: xxx, item_type: new, next: xxx },
      { account_id: xxx, item_type: new, next: xxx },
      { account_id: xxx, item_type: new, next: xxx },
    ],
    ...
  ]
}
```

- proposer_lock，提案发起者的 lock script；如果提案被回收，那么回收的 CKB 就应该转入此 lock script；如果提案通过，那么属于提案发起者的利润就应该转入 IncomeCell 并以此 lock script 记账；
- created_at_height ，提案发起时从时间 cell 获取到的当前高度；
- slices，当前提案通过后 AccountCell 链表被修改部分的最终状态，其解释详见 `TODO`；
- item_type 含义说明
  - exist ，值对应 0x00 ，表明此提案发起时，account_id 所指账户已经为注册状态，可以在链上找到 AccountCell；
  - proposed ，值对应 0x01，表明此提案发起时，account_id 所指账户已经为预注册状态，可以在链上找到 PreAccountCell，当此提案的前置提案通过时会将其转换为 AccountCell；
  - new ，值对应 0x02，表明此提案发起时，account_id 所指账户已经为预注册状态，可以在链上找到 PreAccountCell ，此提案通过时会将其转换为 AccountCell；

#### 体积

`116` Bytes


### AccountCell

当提案确认后，也就是 [ProposalCell](#ProposalCell) 被消费时，[PreAccountCell](#PreAccountCell) 才能被转换为 AccountCell ，它存放账户的各种信息。

#### 结构

```
lock:
  code_hash: <das-lock>
  type: type
  args: [ // 这是 das-lock 的 args 结构，同时包含了 owner 和 manager 信息
    owner_code_hash_index,
    owner_pubkey_hash,
    manager_code_hash_index,
    manager_pubkey_hash,
  ]
type: <account-cell-type>

data:
  hash(witness: AccountCellData) // 32 bytes
  id // 20 bytes，自己的 ID，生成算法为 hash(account)，然后取前 20 Bytes
  next // 20 bytes，下一个 AccountCell 的 ID
  expired_at // 8 bytes，小端编码的 u64 时间戳
  account // expired_at 之后的所有 bytes，utf-8 编码，AccountCell 为了避免数据丢失导致用户无法找回自己用户所以额外储存了 account 的明文信息, 包含 .bit 后缀

witness:
  table Data {
    old: table DataEntityOpt {
        index: Uint32,
        version: Uint32,
        entity: AccountCellData
    },
    new: table DataEntityOpt {
      index: Uint32,
      version: Uint32,
      entity: AccountCellData
    },
  }

======
table AccountCellData {
    // The first 160 bits of the hash of account.
    id: AccountId,
    // Separate chars of account. not include .bit
    account: AccountChars,
    // AccountCell register timestamp.
    registered_at: Uint64,
    // The status of the account, 0x00 means normal, 0x01 means being sold, 0x02 means being auctioned.
    status: Uint8,
    records: Records,
}

array AccountId [byte; 20];

table Record {
    record_type: Bytes,
    record_label: Bytes,
    record_key: Bytes,
    record_value: Bytes,
    record_ttl: Uint32,
}

vector Records <Record>;
```

- id ，账户 ID，对账户名(**含后缀**)计算 hash 之后，取前 20 bytes 就是账户 ID，全网唯一；
- account ，账户名字段；
- registered_at ，注册时间；
- status ，状态字段：
  - 0 ，正常；
  - 1 ，出售中；
  - 2 ，拍卖中；
- records ，解析记录字段，**此字段仅限有管理权的用户编辑**；

#### das-lock

das-lock 是为 DAS 设计的一个特殊 lock script ，它**会根据 args 中的 code_hash_index 部分去动态加载不同的验签逻辑执行**。args 中的 **code_hash_index 都是 1 byte，pubkey_hash 都是取前 20 bytes** 。

涉及验签的交易需要在 witnesses 中的 ActionData.params 标明当前是 owner 还是 manager ，**owner 使用 0，manager 使用 1**。

#### 体积

`188 + n` Bytes，`n` 为账户的字节长度。


### IncomeCell

一种用来解决批量到账时单笔账目不足 61 CKByte 无法独立存在的 Cell ，这个 Cell 以及其相关解决方案主要有以下优点：

1. 可以在批量转账的场景下，解决单笔账目不足 61 CKByte 无法创建普通 Cell 的问题；
2. 解决了总账目也不足 61 CKByte 无法创建 IncomeCell 的问题；
3. 通过复用 IncomeCell 实现上面优点 1、2 的同时降低了多笔交易抢占同一个 IncomeCell 的概率；

#### 结构

```
lock: <always_success>
type: <income-cell-type>

data: hash(witness: IncomeCellData)

======
table IncomeCellData {
    creator: Script,
    records: IncomeRecords,
}

vector IncomeRecords <IncomeRecord>;

table IncomeRecord {
    belong_to: Script,
    capacity: Uint64,
}
```

Witness 中的主要字段如下：

- creator ，记录了这个 IncomeCell 的创建者，任何人都可以自由的创建 IncomeCell ，当 records 中只有创建者一条记录时，这个 IncomeCell 就只能用于确认提案交易；
- records ，账目记录，记录了 IncomeCell 的 capacity 分别属于哪些 lock script ，每个 lock script 拥有多少 CKByte；

#### 体积

`106` Bytes


### ConfigCell

这是一个在链上保存 DAS 配置的 Cell，目前只通过 DAS 超级私钥手动更新。因为 CKB VM 在加载数据时存在性能存在数据越大开销急剧增大的问题，所以采用了将不同配置分散到多个 ConfigCell 中的保存方式。

#### 结构

所有的 ConfigCell 都遵循以下基本的数据，不同的 ConfigCell 主要是通过 `type.args` 来体现：

```
lock: <super_lock> // 一个官方的 ckb 多签 lock
type:
    code_hash: <config-cell-type>,
    type: type,
    args: [DateType], // 这里保存了一个 uint32 的 DateType 值，这个值主要是为了方便辨别和查询 ConfigCell
data:
    hash(witness) // 所有的 ConfigCell 的 data 都是一个计算自 witness 的 Hash
```

> 各个 ConfigCell 的 DataType 详见下面 [Type 常量列表](#Type 常量列表) 。

#### 体积

所有 ConfigCell 在 cell 结构上都一样，所以体积都是 `130` Bytes 。

#### ConfigCellAccount

**witness：**

```
table ConfigCellAccount {
    // The maximum length of accounts in characters.
    max_length: Uint32,
    // The basic capacity AccountCell required, it is bigger than or equal to AccountCell occupied capacity.
    basic_capacity: Uint64,
    // The grace period for account expiration in seconds
    expiration_grace_period: Uint32,
    // The minimum ttl of record in seconds
    record_min_ttl: Uint32,
}
```

- max_length ，账户的最大**字符**长度；
- basic_capacity ，账户的存储空间，可能比 cell 占用的存储更大；
- expiration_grace_period ，账户到期后的宽限期，单位 秒；
- record_min_ttl ，解析记录的最小 TTL 值，单位 秒；

#### ConfigCellApply

**witness：**

```
table ConfigCellApply {
    // The minimum waiting block number before apply_register_cell can be converted to pre_account_cell.
    apply_min_waiting_block_number: Uint32,
    // The maximum waiting block number which apply_register_cell can be converted to pre_account_cell.
    apply_max_waiting_block_number: Uint32,
}
```

- apply_min_waiting_block_number ，ApplyRegisterCell 可以用于预注册的最小等待区块数；
- apply_max_waiting_block_number ，ApplyRegisterCell 可以用于预注册的最大等待区块数；

#### ConfigCellCharSetXXXX

注册账户可用字符集 Cell ，这个 Cell 的设计是除了 Emoji, Digit 两个全局字符集外，其他各种语言都可以有一个自己独立的字符集，不同语言之间不能混用，**只有全局字符集 Emoji，Digit 能和任何语言混用**。

**witness：**

```
length|global|char|char|char ...
```

这个 cell 的 witness 在其 entity 部分**存储的是纯二进制数据**，未进行 molecule 编码。其中前 4 bytes 是 uint32 的数据总长度，**包括这 4 bytes 自身**；第 5 bytes 记录当前字符集是否为全局字符集，0x00 就不是，0x01 就是；之后就是可用字符的字节，全部为 utf-8 编码对应的字节，每个字符之间以 `0x00` 分割。

目前已经有的字符集为：

- ConfigCellCharSetEmoji
- ConfigCellCharSetDigit
- ConfigCellCharSetEn

#### ConfigCellIncome

**witness：**

```
table ConfigCellIncome {
    // The basic capacity IncomeCell required, it is bigger than or equal to IncomeCell occupied capacity.
    basic_capacity: Uint64,
    // The maximum records one IncomeCell can hold.
    max_records: Uint32,
}
```

- basic_capacity ，IncomeCell 的存储空间，可能比 cell 占用的存储更大；
- max_records ，单个 IncomeCell 的最大存储账目记录数，当记录数超出此上限时应该拆分多个 IncomeCell 存储账目记录；

#### ConfigCellMain

**witness：**

```
table ConfigCellMain {
    // Global DAS system switch, 0x01 means system on, 0x00 means system off.
    status: Uint8,
    // table of type ID of all kinds of cells
    type_id_table: TypeIdTable,
}

table TypeIdTable {
    account_cell: Hash,
    apply_register_cell: Hash,
    bidding_cell: Hash,
    income_cell: Hash,
    on_sale_cell: Hash,
    pre_account_cell: Hash,
    proposal_cell: Hash,
}
```

- status ，DAS 系统全局开关，具体值的范围见 [系统状态枚举值 Status](#系统状态枚举值 Status)；
- type_id_table ，各个 type 脚本对应的 type ID;

#### ConfigCellPrice

**witness：**

```
table ConfigCellPrice {
    // discount configurations
    discount: DiscountConfig,
    // Price list of different account length.
    prices: PriceConfigList,
}

table DiscountConfig {
    // The discount rate for invited user
    invited_discount: Uint32,
}

vector PriceConfigList <PriceConfig>;

table PriceConfig {
  // The length of the account, ".bit" suffix is not included.
  length: Uint8,
  // The price of registering an account. In USD, accurate to 6 decimal places.
  new: Uint64,
  // The price of renewing an account. In USD, accurate to 6 decimal places.
  renew: Uint64,
}
```

- discount ，DAS 中各种情况下的折扣额度列表；
- prices ，DAS 不同长度账户名的价格列表；

#### ConfigCellProposal

**witness：**

```
table ConfigCellProposal {
    // How many blocks required for every proposal to be confirmed.
    proposal_min_confirm_interval: Uint8,
    // How many blocks to wait before extending the proposal.
    proposal_min_extend_interval: Uint8,
    // How many blocks to wait before recycle the proposal.
    proposal_min_recycle_interval: Uint8,
    // How many account_cells every proposal can affect.
    proposal_max_account_affect: Uint32,
    // How many pre_account_cells be included in every proposal.
    proposal_max_pre_account_contain: Uint32,
}
```

- proposal_min_confirm_interval ，提案确认的最小等待区块数；
- proposal_min_extend_interval ，提案扩展的最小等待区块数；
- proposal_min_recycle_interval ，提案回收的最小等待区块数；
- proposal_max_account_affect ，单个提案可以涉及的最大 AccountCell 数；
- proposal_max_pre_account_contain ，单个提案可以涉及的最大 PreAccountCell 数；

> `proposal_max_account_affect `和 `proposal_max_pre_account_contain` 其中任何一个到达上限后就必须忽略 `proposal_min_extend_interval` 的限制创建新的提案，如此来控制每个提案的体积上限。

#### ConfigCellProfitRate

**witness：**

```
table ConfigCellProfitRate {
    // The profit rate of inviters who invite people to buy DAS accounts.
    inviter: Uint32,
    // The profit rate of channels who support people to create DAS accounts.
    channel: Uint32,
    // The profit rate of DAS itself.
    das: Uint32,
    // The profit rate for who created proposal
    proposal_create: Uint32,
    // The profit rate for who confirmed proposal
    proposal_confirm: Uint32,
    // The profit rate for consolidate IncomeCells
    income_consolidate: Uint32,
}
```

- inviter ，账户注册流程中邀请人的利润率；
- channel ，账户注册流程中渠道的利润率；
- das ，账户注册流程中 DAS 官方的利润率；
- proposal_create ，账户注册流程中 keeper 创建提案的利润率；
- proposal_confirm ，账户注册流程中 keeper 确认提案的利润率；
- income_consolidate ，IncomeCell 合并流程中 keeper 的利润率；

#### ConfigCellRecordKeyNamespace

解析记录 key 命名空间。

**witness：**

```
length|key|key|key ...
```

这个 cell 的 witness 在其 entity 部分**存储的是纯二进制数据**，未进行 molecule 编码。其中前 4 bytes 是 uint32 的数据总长度，**包括这 4 bytes 自身**，之后就是解析记录中可用的各个 key 值字符串的 ASCII 编码，比如原本的 key 是 `address.eth` 那么存储的就是 `0x616464726573732E657468` ，每个 key 之间以 `0x00` 分割。

#### ConfigCellPreservedAccountXX

保留账户名过滤器，目前只有一个，将来根据情况可能创建 8 个，将账户名 hash 后，根据第一个字节 的 u8 整形对 8 取模的结果进行分配。

**witness：**

```
length|hash|hash|hash ...
```

这个 cell 的 witness 在其 entity 部分**存储的是纯二进制数据**，未进行 molecule 编码。其中前 4 bytes 是 uint32 的数据总长度，**包括这 4 bytes 自身**，之后就是各个账户名 hash 后前 10 bytes 拼接而成的数据，因为每段数据固定为 10 bytes 所以**无分隔符等字节**。

### QuoteCell

由于 CKB 链上目前暂无和美元挂钩的稳定币，DAS 官方会通过一个报价 Cell 公布当前采用的 CKB 对美元的汇率。

#### 结构

```
lock: <oracle_lock>
type: null
data:
  [quote: u64]
```

单位：CKB/USD ，因为 USD 采用了扩大 `1_000_000` 倍的方式精确到小数点后 6 位，所以 CKB 也要等比扩大，既市价为 0.001 CKB/USD 时，QuoteCell 需要记录为 1000 CKB/USD 。

#### 体积

`107` Bytes



### TimeCell、HeightCell

这是 CKB 官方开发并以去中心化方式维护的一对链上 Cell ，他们的作用是为合约脚本提供当前时间的验证依据，每种 Cell 会同时有复数存在，这是为了避免热点问题而采用的措施。
对于 DAS 来说，具体使用哪一个并不会造成任何影响，所以只需注意其数据结构即可。

#### TimeCell

TimeCell 因为 TimeCell 的时间戳实际上还是基于链上的时间戳产生，所以与现实时间存在 5 分钟左右的误差。

```
lock: <always_success>
type: <time_cell>
data:
    [index] // 1 字节小端编码的 u8 整形，存放当前是 TimeCell 中的第几个
    [timestamp] // 4 字节小端编码的 u32 整形，存放当前的 UTC 时间戳
```

#### HeightCell

HeightCell 与实际的区块高度存在 1 ～ 2 个区块的误差。

```
lock: <always_success>
type: <height_cell>
data:
    [index] // 1 字节小端编码的 u8 整形，存放当前是 Height 中的第几个
    [block_height] // 8 字节小端编码的 u64 整形，存放当前的区块高度
```


## 其他
### Hash 算法

如无特殊说明，本文均使用 `blake2b` 算法，并配合 CKB 指定的默认参数：

- **output digest size**: `32`
- **personalization**: `ckb-default-hash`

具体来说就是初始化 hasher 时采用类似代码：`let hasher = blake2b(digest_size=32, person=b'ckb-default-hash')`。

> 本文中的 hash 值除非特别说明都是 32 Bytes 长。

### 系统状态枚举值 Status

```
enum SystemStatus {
    Off,
    On,
}
```

### Witness 类型枚举值 DataType

```
enum DataType {
    ActionData = 0,
    AccountCellData,
    OnSaleCellData,
    BiddingCellData,
    ProposalCellData,
    PreAccountCellData,
    IncomeCellData,
    ConfigCellAccount = 100, // args: 0x64000000
    ConfigCellApply = 101, // args: 0x65000000
    ConfigCellIncome = 103, // args: 0x67000000
    ConfigCellMain, // args: 0x68000000
    ConfigCellPrice, // args: 0x69000000
    ConfigCellProposal, // args: 0x6a000000
    ConfigCellProfitRate, // args: 0x6b000000
    ConfigCellRecordKeyNamespace, // args: 0x6c000000
    ConfigCellPreservedAccount00 = 150, // args: 0x96000000
    // ConfigCellPreservedAccount01,
    // ConfigCellPreservedAccount02,
    // ConfigCellPreservedAccount03,
    // ConfigCellPreservedAccount04,
    // ConfigCellPreservedAccount05,
    // ConfigCellPreservedAccount06,
    // ConfigCellPreservedAccount07,
    ConfigCellCharSetEmoji = 100000, // 0xa0860100
    ConfigCellCharSetDigit = 100001, // 0xa1860100
    ConfigCellCharSetEn = 100002, // 0xa2860100
}
```

### 字符集枚举值 CharSet

> ⚠️ 为了方便维护， ConfigCellCharSetXXX 总是保持减去 100000 就等于下面 CharSet 常量的形式，所以可以直接采用下面的转换方法：
>
> `ConfigCellCharSetEmoji - 100000 = EMOJI`

```
enum CharSetType {
    Emoji,
    Digit,
    En,
    ZhCn,
}
```

