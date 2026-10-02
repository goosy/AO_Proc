# AO_Proc 设计说明

[English](design.md) | 简体中文

AO_Proc 支持 S7-300 SM332 / S7-1500 AQ 的全部常规 AO 量程。`mode` 的默认值为 0（4 ~ 20 mA），与 v0.1 的行为兼容。

## 三种情况

所有 AO 量程都可以归入以下三类：

| 类别 | 典型量程 | 零点对应值 (raw_zero) | 满量程对应值 (S7_SPAN) | 硬件下限 | S7_AO_MAX | underflow_SP 默认 | overflow_SP 默认 |
|---|---|---|---|---|---|---|---|
| 1. 单极性、零点带偏置 | 4 ~ 20 mA、1 ~ 5 V | 0 | 27648 | -6912 | 32511 | -500 | 28000 |
| 2. 单极性、零点不带偏置 | 0 ~ 20 mA、0 ~ 10 V | 0 | 27648 | 0 | 32511 | 0 | 28000 |
| 3. 双极性 | ±20 mA、±10 V | -27648 | 27648 | -32512 | 32511 | -28000 | 28000 |

### 默认值对应的物理量

- 4 ~ 20 mA：28000 ≈ 20.20 mA，-500 ≈ 3.71 mA。1 ~ 5 V：-500 ≈ 0.93 V。
- 0 ~ 10 V：28000 ≈ 10.13 V；0 ~ 20 mA 时 28000 ≈ 20.25 mA。下限默认取 0，因为写负值也只会输出 0 V（或 0 mA），设成负数没有意义。
- ±10 V：±28000 ≈ ±10.13 V，上下对称，各留出约 1.3% 的余量。

### 注意

- S7_AO_MAX 三类都是 32511。超过这个值后，S7-300 会断电输出 0，S7-1500 会钳在 117.589%。
- 硬件下限是硬件允许的边界，underflow_SP 是工艺上的报警限。前者用来防止写出无效值，后者决定什么时候置 underflow，两者的作用不同。

## 换算公式

三类共用一条通用公式，按 mode 选择 `raw_zero`：

```
value := raw_zero + (PV - zero) * (S7_SPAN - raw_zero) / (span - zero)
```

- 类别 1、2：raw_zero = 0，结果与 v0.1 的公式完全一致。
- 类别 3：展开后就是 `(PV - zero) / (span - zero) × 55296 - 27648`。
- `raw_zero` 声明为 REAL。双极性时 `S7_SPAN - raw_zero = 55296`，超出 INT 范围，必须按 REAL 计算。

## mode 与类别的对应

| mode | 输出量程 | 类别 | raw_zero | 硬件下限 | 默认下限 | 默认下限对应物理量 |
|---|---|---|---|---|---|---|
| 0 | 4 ~ 20 mA | 1 | 0 | -6912 | -500 | ≈ 3.71 mA |
| 1 | 0 ~ 20 mA | 2 | 0 | 0 | 0 | 0 mA |
| 2 | 0 ~ 10 V | 2 | 0 | 0 | 0 | 0 V |
| 3 | 1 ~ 5 V | 1 | 0 | -6912 | -500 | ≈ 0.93 V |
| 4 | ±10 V | 3 | -27648 | -32512 | -28000 | ≈ -10.13 V |
| 5 | ±20 mA | 3 | -27648 | -32512 | -28000 | ≈ -20.25 mA |

- 其他 mode 值按模式错误处理：AO 输出 0，置 invalid。
- 三类的满量程值都是 27648，S7_AO_MAX 都是 32511，overflow_SP 默认都是 28000。
- mode 1 和 2、mode 0 和 3、mode 4 和 5 在代码里处理完全相同。分开编号只是为了组态时能直接看出量程，FB 只按类别处理。

## underflow_SP 的默认值：哨兵值

FB 输入参数的默认值是固定的，没法随 mode 变化。如果固定用 -500 作为默认值，类别 3 就会出错：忘记改 underflow_SP 时，负半段在约 -0.18 V 处被截断，还会误报 underflow，而且没有任何提示。

处理方法：

- `underflow_SP` 的默认值为 `-32768`（`UNDERFLOW_SP_AUTO`），含义是"按 mode 对应类别取默认下限"。
- 每个扫描周期都根据 mode 重新算出实际下限，存在临时变量 `low` 里。**不把默认值写回 `underflow_SP`**，否则 -32768 这个标记会丢失：改 mode 后下限不会跟着变，在线监视时也分不清是自动值还是人工设定值。
- 显式赋值时以赋的值为准，但仍受硬件下限钳位。
- 兼容性：-32768 比三类的硬件下限都小，v0.1 中显式传入时会被钳到 -6912。现在它的含义变成 -500，影响可以忽略。

```pascal
IF underflow_SP = UNDERFLOW_SP_AUTO THEN
    low := def_low;          // 按类别取默认值
ELSIF underflow_SP < raw_min THEN
    low := raw_min;          // 显式赋值，但不能超出硬件下限
ELSE
    low := underflow_SP;
END_IF;
```

## 常量

所有边界值和默认值都定义成常量，代码里不直接写数字：

| 常量 | 值 | 含义 |
|---|---|---|
| S7_ZERO | 0 | 单极性零点原始值 |
| S7_SPAN | 27648 | 满量程原始值 |
| S7_BIPOLAR_ZERO | -27648 | 双极性零点原始值 |
| S7_AO_MIN1 | -6912 | 类别 1 硬件下限 |
| S7_AO_MIN2 | 0 | 类别 2 硬件下限 |
| S7_AO_MIN3 | -32512 | 类别 3 硬件下限 |
| S7_AO_MAX | 32511 | 硬件上限 |
| OVERFLOW_SP_DEF | 28000 | overflow_SP 默认值 |
| UNDERFLOW_SP_DEF1 | -500 | 类别 1 默认下限 |
| UNDERFLOW_SP_DEF2 | 0 | 类别 2 默认下限 |
| UNDERFLOW_SP_DEF3 | -28000 | 类别 3 默认下限 |
| UNDERFLOW_SP_AUTO | -32768 | 哨兵值，表示按类别取默认下限 |

参数初值直接引用常量：`overflow_SP` 取 `OVERFLOW_SP_DEF`，`underflow_SP` 取 `UNDERFLOW_SP_AUTO`。三个平台都支持这种写法，AI_Proc 也是这样写的：

- Portal：`zero_raw : Int := #S7_ZERO;`
- Step7：`zero_raw {S7_m_c := 'true'} : INT := S7_ZERO;`（常量在 `CONST ... END_CONST` 中声明）
- PCS7：`} : INT := S7_ZERO;`

## 处理流程

1. 根据 mode 从常量中选出 `raw_zero`、`raw_min`、`def_low`。mode 不在 0 ~ 5 时置 `mode_err`。
2. 出现 `mode_err` 或 `span = zero` 时，AO 输出 0，置 invalid，不再往下处理。
3. 上限 = MIN(overflow_SP, S7_AO_MAX)；下限按哨兵值规则计算。
4. 用通用公式换算，超出上下限时钳位，并置 overflow / underflow。
5. 用 REAL_TO_INT 四舍五入写入 AO，invalid = overflow OR underflow。
