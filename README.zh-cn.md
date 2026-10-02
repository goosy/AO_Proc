# S7 AO 通道常规处理子过程

[English](README.md) | 简体中文

将工程单位数值线性转换为 AO 模块通道值（单极性 0 ~ 27648，双极性 -27648 ~ 27648），并对超出范围的值限幅。

## 说明

`mode` 必须与硬件组态的输出类型一致：

| mode | 输出量程 | 类别 | 硬件范围（通道原始值） | underflow_SP 默认 |
|---|---|---|---|---|
| 0 | 4 ~ 20 mA | 1. 单极性、零点带偏置 | -6912 ~ 32511 | -500（≈ 3.71 mA） |
| 1 | 0 ~ 20 mA | 2. 单极性、零点不带偏置 | 0 ~ 32511 | 0 |
| 2 | 0 ~ 10 V | 2. 单极性、零点不带偏置 | 0 ~ 32511 | 0 |
| 3 | 1 ~ 5 V | 1. 单极性、零点带偏置 | -6912 ~ 32511 | -500（≈ 0.93 V） |
| 4 | ±10 V | 3. 双极性 | -32512 ~ 32511 | -28000（≈ -10.13 V） |
| 5 | ±20 mA | 3. 双极性 | -32512 ~ 32511 | -28000（≈ -20.25 mA） |

`overflow_SP` 默认 28000（4 ~ 20 mA 时 ≈ 20.20 mA，0 ~ 10 V 时 ≈ 10.13 V）。设计说明见 [design.zh-cn.md](design.zh-cn.md)。

- 用于博途的SCL源码：AO_Proc(portal).scl
- 用于Step7的SCL源码：AO_Proc(step7).scl
- 用于PCS7的SCL源码：AO_Proc(pcs7).scl

## 处理逻辑

- `AO = raw_zero + (PV - zero) / (span - zero) × (27648 - raw_zero)`，四舍五入取整；
  `raw_zero` 单极性为 0，双极性为 -27648
- 上限为 `overflow_SP`，下限为 `underflow_SP`，均为通道原始值，且不超出上表的硬件范围
- `underflow_SP = -32768`（默认）时，下限取 mode 对应类别的默认值（见上表）
- 超过上限时 AO 限幅为上限并置 `overflow`；低于下限时限幅为下限并置 `underflow`
- `mode` 不在 0 ~ 5 范围内（模式错误）或 `span = zero`（量程设置错误）时，AO 输出 0
- `invalid = 模式错误 OR 量程错误 OR overflow OR underflow`

## 调用示例

调用时，先建立对应的背景数据块，比如"FV001"，调用时所有参数可省略，直接操纵背景块。
`overflow_SP`、`underflow_SP`、`mode` 的显式赋值是可选的，省略时使用默认值 28000、-32768（按类别取默认下限）和 0。

注：下方_name_表示具体变量

1. 在 TIA Portal 中调用示例：

    ```Pascal
    "FV001"(
            PV:=_real_in_,          // 输出量工程单位数值
            zero:=0.0,              // 量程低值
            span:=100.0,            // 量程高值
            overflow_SP:=28000,     // 可选，上溢出值（通道原始值），默认 28000
            underflow_SP:=-500,     // 可选，下溢出值（通道原始值），默认 -32768 = 按类别取默认值
            mode:=0,                // 可选，输出模式 0 ~ 5，见上表，默认 0 = 4 ~ 20 mA
            AO=>_word_out_,         // 模块输出通道值，最好定义一个对应PQW通道的符号
            invalid=>_bool_out_,    // 数据无效，即 模式错误 OR 量程错误 OR overflow OR underflow
            overflow=>_bool_out_,   // 高溢出，已限幅
            underflow=>_bool_out_); // 低溢出，已限幅
    ```

1. 在 Step7 V5.5 中SCL调用示例：

    ```Pascal
    AO_Proc.FV001(
        PV                  := _real_in_,     // 输出量工程单位数值
        zero                := 0.0,           // 量程低值
        span                := 100.0,         // 量程高值
        overflow_SP         := 28000,         // 可选，上溢出值（通道原始值），默认 28000
        underflow_SP        := -500,          // 可选，下溢出值（通道原始值），默认 -32768 = 按类别取默认值
        mode                := 0);            // 可选，输出模式 0 ~ 5，见上表，默认 0 = 4 ~ 20 mA
    PQW256 := FV001.AO;                       // 模块输出通道值
    _bool_out_ := FV001.invalid;              // 数据无效
    _bool_out_ := FV001.overflow;             // 高溢出
    _bool_out_ := FV001.underflow;            // 低溢出
    ```

1. 在 Step7 V5.5 中STL调用示例：

    ```Pascal
    CALL  "AO_Proc" , "FV001"(
        PV                  := _real_in_,     // 输出量工程单位数值
        zero                := 0.0,           // 量程低值
        span                := 100.0,         // 量程高值
        overflow_SP         := 28000,         // 可选，上溢出值（通道原始值），默认 28000
        underflow_SP        := -500,          // 可选，下溢出值（通道原始值），默认 -32768 = 按类别取默认值
        mode                := 0,             // 可选，输出模式 0 ~ 5，见上表，默认 0 = 4 ~ 20 mA
        AO                  := PQW256,        // 模块输出通道值
        invalid             := _bool_out_,    // 数据无效
        overflow            := _bool_out_,    // 高溢出
        underflow           := _bool_out_);   // 低溢出
    ```
