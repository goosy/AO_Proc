# S7 AO Channel Processing Block

English | [简体中文](README.zh-cn.md)

Linearly scales an engineering value to an AO module channel value (unipolar 0 ~ 27648, bipolar -27648 ~ 27648) and clamps out-of-range values.

## Overview

`mode` must match the output type configured in the hardware:

| mode | Output range | Category | Hardware range (raw value) | underflow_SP default |
|---|---|---|---|---|
| 0 | 4 ~ 20 mA | 1. Unipolar, offset zero | -6912 ~ 32511 | -500 (≈ 3.71 mA) |
| 1 | 0 ~ 20 mA | 2. Unipolar, no offset | 0 ~ 32511 | 0 |
| 2 | 0 ~ 10 V | 2. Unipolar, no offset | 0 ~ 32511 | 0 |
| 3 | 1 ~ 5 V | 1. Unipolar, offset zero | -6912 ~ 32511 | -500 (≈ 0.93 V) |
| 4 | ±10 V | 3. Bipolar | -32512 ~ 32511 | -28000 (≈ -10.13 V) |
| 5 | ±20 mA | 3. Bipolar | -32512 ~ 32511 | -28000 (≈ -20.25 mA) |

`overflow_SP` defaults to 28000 (≈ 20.20 mA for 4 ~ 20 mA, ≈ 10.13 V for 0 ~ 10 V). See [design.md](design.md) for the design notes.

- SCL source for TIA Portal: AO_Proc(portal).scl
- SCL source for Step7: AO_Proc(step7).scl
- SCL source for PCS7: AO_Proc(pcs7).scl

## Processing

- `AO = raw_zero + (PV - zero) / (span - zero) × (27648 - raw_zero)`, rounded to the nearest integer;
  `raw_zero` is 0 for unipolar and -27648 for bipolar ranges
- The high limit is `overflow_SP` and the low limit is `underflow_SP`, both raw values, bounded by the hardware range in the table above
- When `underflow_SP = -32768` (the default), the low limit is the default of the category for the given mode (see the table above)
- Above the high limit, AO is clamped to the high limit and `overflow` is set; below the low limit, AO is clamped to the low limit and `underflow` is set
- When `mode` is outside 0 ~ 5 (mode error) or `span = zero` (range error), AO outputs 0
- `invalid = mode error OR range error OR overflow OR underflow`

## Usage

Create an instance DB first, e.g. "FV001". All parameters can be omitted in the call and accessed directly through the instance DB.
Assigning `overflow_SP`, `underflow_SP` and `mode` is optional; when omitted, they default to 28000, -32768 (category default low limit) and 0.

Note: _name_ below stands for an actual variable.

1. Call in TIA Portal:

    ```Pascal
    "FV001"(
            PV:=_real_in_,          // output value in engineering units
            zero:=0.0,              // range low value
            span:=100.0,            // range high value
            overflow_SP:=28000,     // optional, overflow setpoint (raw value), default 28000
            underflow_SP:=-500,     // optional, underflow setpoint (raw value), default -32768 = category default
            mode:=0,                // optional, output mode 0 ~ 5, see table above, default 0 = 4 ~ 20 mA
            AO=>_word_out_,         // module channel value, preferably a symbol for the PQW channel
            invalid=>_bool_out_,    // data invalid, i.e. mode error OR range error OR overflow OR underflow
            overflow=>_bool_out_,   // overflow, output clamped
            underflow=>_bool_out_); // underflow, output clamped
    ```

1. SCL call in Step7 V5.5:

    ```Pascal
    AO_Proc.FV001(
        PV                  := _real_in_,     // output value in engineering units
        zero                := 0.0,           // range low value
        span                := 100.0,         // range high value
        overflow_SP         := 28000,         // optional, overflow setpoint (raw value), default 28000
        underflow_SP        := -500,          // optional, underflow setpoint (raw value), default -32768 = category default
        mode                := 0);            // optional, output mode 0 ~ 5, see table above, default 0 = 4 ~ 20 mA
    PQW256 := FV001.AO;                       // module channel value
    _bool_out_ := FV001.invalid;              // data invalid
    _bool_out_ := FV001.overflow;             // overflow
    _bool_out_ := FV001.underflow;            // underflow
    ```

1. STL call in Step7 V5.5:

    ```Pascal
    CALL  "AO_Proc" , "FV001"(
        PV                  := _real_in_,     // output value in engineering units
        zero                := 0.0,           // range low value
        span                := 100.0,         // range high value
        overflow_SP         := 28000,         // optional, overflow setpoint (raw value), default 28000
        underflow_SP        := -500,          // optional, underflow setpoint (raw value), default -32768 = category default
        mode                := 0,             // optional, output mode 0 ~ 5, see table above, default 0 = 4 ~ 20 mA
        AO                  := PQW256,        // module channel value
        invalid             := _bool_out_,    // data invalid
        overflow            := _bool_out_,    // overflow
        underflow           := _bool_out_);   // underflow
    ```
