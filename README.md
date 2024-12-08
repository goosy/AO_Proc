# S7 AO 通道常规处理子过程

将工程单位数值线性转换为 AO 模块通道值（0 ~ 27648），并对超出范围的值限幅。

## 说明

**仅支持硬件组态为 4 ~ 20 mA 的输出通道。** `mode` 为预留参数，目前只能为 0（4 ~ 20 mA）。

- 用于博途的SCL源码：AO_Proc(portal).scl
- 用于Step7的SCL源码：AO_Proc(step7).scl
- 用于PCS7的SCL源码：AO_Proc(pcs7).scl

## 处理逻辑

- `AO = (PV - zero) / (span - zero) × 27648`，四舍五入取整
- 上限为 `overflow_SP`，下限为 `underflow_SP`，均为通道原始值，
  且不超出 S7 模拟量范围 -6912 ~ 32511（4 ~ 20 mA 时对应 0 ~ 22.81 mA）
- 超过上限时 AO 限幅为上限并置 `overflow`；低于下限时限幅为下限并置 `underflow`
- `mode <> 0`（不支持的模式）或 `span = zero`（量程设置错误）时，AO 输出 0
- `invalid = 模式错误 OR 量程错误 OR overflow OR underflow`

## 调用示例

调用时，先建立对应的背景数据块，比如"FV001"，调用时所有参数可省略，直接操纵背景块。
`overflow_SP`、`underflow_SP`、`mode` 的显式赋值是可选的，省略时使用默认值 28000、-500 和 0。

注：下方_name_表示具体变量

1. 在 TIA Portal 中调用示例：

    ```Pascal
    "FV001"(
            PV:=_real_in_,          // 输出量工程单位数值
            zero:=0.0,              // 量程低值
            span:=100.0,            // 量程高值
            overflow_SP:=28000,     // 可选，上溢出值（通道原始值），默认 28000
            underflow_SP:=-500,     // 可选，下溢出值（通道原始值），默认 -500
            mode:=0,                // 可选，输出模式，0 = 4 ~ 20 mA（目前唯一支持），默认 0
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
        underflow_SP        := -500,          // 可选，下溢出值（通道原始值），默认 -500
        mode                := 0);            // 可选，输出模式，0 = 4 ~ 20 mA（目前唯一支持），默认 0
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
        underflow_SP        := -500,          // 可选，下溢出值（通道原始值），默认 -500
        mode                := 0,             // 可选，输出模式，0 = 4 ~ 20 mA（目前唯一支持），默认 0
        AO                  := PQW256,        // 模块输出通道值
        invalid             := _bool_out_,    // 数据无效
        overflow            := _bool_out_,    // 高溢出
        underflow           := _bool_out_);   // 低溢出
    ```
