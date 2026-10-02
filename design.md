# AO_Proc Design Notes

English | [简体中文](design.zh-cn.md)

AO_Proc supports all standard AO ranges of the S7-300 SM332 / S7-1500 AQ modules. `mode` defaults to 0 (4 ~ 20 mA), which keeps v0.1 behavior.

## Three categories

Every AO range falls into one of three categories:

| Category | Typical ranges | Raw value at zero (raw_zero) | Raw value at span (S7_SPAN) | Hardware low limit | S7_AO_MAX | underflow_SP default | overflow_SP default |
|---|---|---|---|---|---|---|---|
| 1. Unipolar, offset zero | 4 ~ 20 mA, 1 ~ 5 V | 0 | 27648 | -6912 | 32511 | -500 | 28000 |
| 2. Unipolar, no offset | 0 ~ 20 mA, 0 ~ 10 V | 0 | 27648 | 0 | 32511 | 0 | 28000 |
| 3. Bipolar | ±20 mA, ±10 V | -27648 | 27648 | -32512 | 32511 | -28000 | 28000 |

### Physical values of the defaults

- 4 ~ 20 mA: 28000 ≈ 20.20 mA, -500 ≈ 3.71 mA. 1 ~ 5 V: -500 ≈ 0.93 V.
- 0 ~ 10 V: 28000 ≈ 10.13 V; for 0 ~ 20 mA, 28000 ≈ 20.25 mA. The low limit defaults to 0 because a negative value still outputs only 0 V (or 0 mA), so a negative limit is meaningless.
- ±10 V: ±28000 ≈ ±10.13 V, symmetric, leaving about 1.3% margin on each side.

### Notes

- S7_AO_MAX is 32511 for all three categories. Above it, the S7-300 switches the output off to 0, while the S7-1500 clamps at 117.589%.
- The hardware low limit is the boundary the hardware accepts; underflow_SP is the process alarm limit. The former prevents writing invalid values, the latter decides when underflow is set. They serve different purposes.

## Scaling formula

All three categories share one general formula; `raw_zero` is selected by mode:

```
value := raw_zero + (PV - zero) * (S7_SPAN - raw_zero) / (span - zero)
```

- Categories 1 and 2: raw_zero = 0, identical to the v0.1 formula.
- Category 3: expands to `(PV - zero) / (span - zero) × 55296 - 27648`.
- `raw_zero` is declared as REAL. For bipolar ranges `S7_SPAN - raw_zero = 55296`, which exceeds the INT range, so it must be computed in REAL.

## Mode to category mapping

| mode | Output range | Category | raw_zero | Hardware low limit | Default low limit | Physical value |
|---|---|---|---|---|---|---|
| 0 | 4 ~ 20 mA | 1 | 0 | -6912 | -500 | ≈ 3.71 mA |
| 1 | 0 ~ 20 mA | 2 | 0 | 0 | 0 | 0 mA |
| 2 | 0 ~ 10 V | 2 | 0 | 0 | 0 | 0 V |
| 3 | 1 ~ 5 V | 1 | 0 | -6912 | -500 | ≈ 0.93 V |
| 4 | ±10 V | 3 | -27648 | -32512 | -28000 | ≈ -10.13 V |
| 5 | ±20 mA | 3 | -27648 | -32512 | -28000 | ≈ -20.25 mA |

- Any other mode is a mode error: AO outputs 0 and invalid is set.
- All three categories share span value 27648, S7_AO_MAX 32511 and overflow_SP default 28000.
- Modes 1 and 2, 0 and 3, 4 and 5 are handled identically in code. They are numbered separately only so the range is obvious at configuration time; the FB works by category.

## underflow_SP default: sentinel value

An FB input default is fixed and cannot depend on mode. With a fixed default of -500, category 3 breaks: if underflow_SP is left unchanged, the negative half is cut off at about -0.18 V and underflow is falsely reported, with no other warning.

Solution:

- `underflow_SP` defaults to `-32768` (`UNDERFLOW_SP_AUTO`), meaning "use the default low limit of the category for the given mode".
- The effective low limit is recomputed from mode on every scan and kept in the temporary variable `low`. **The default is never written back to `underflow_SP`**; otherwise the -32768 marker is lost: the limit would not follow a later mode change, and online monitoring could not tell an automatic value from a manual setpoint.
- An explicitly assigned value takes precedence, but is still clamped to the hardware low limit.
- Compatibility: -32768 is below the hardware low limit of every category, so in v0.1 an explicit -32768 was clamped to -6912. It now means -500; the impact is negligible.

```pascal
IF underflow_SP = UNDERFLOW_SP_AUTO THEN
    low := def_low;          // category default
ELSIF underflow_SP < raw_min THEN
    low := raw_min;          // explicit value, bounded by the hardware low limit
ELSE
    low := underflow_SP;
END_IF;
```

## Constants

All limits and defaults are defined as constants; the code contains no bare numbers:

| Constant | Value | Meaning |
|---|---|---|
| S7_ZERO | 0 | Unipolar raw value at zero |
| S7_SPAN | 27648 | Raw value at span |
| S7_BIPOLAR_ZERO | -27648 | Bipolar raw value at zero |
| S7_AO_MIN1 | -6912 | Category 1 hardware low limit |
| S7_AO_MIN2 | 0 | Category 2 hardware low limit |
| S7_AO_MIN3 | -32512 | Category 3 hardware low limit |
| S7_AO_MAX | 32511 | Hardware high limit |
| OVERFLOW_SP_DEF | 28000 | overflow_SP default |
| UNDERFLOW_SP_DEF1 | -500 | Category 1 default low limit |
| UNDERFLOW_SP_DEF2 | 0 | Category 2 default low limit |
| UNDERFLOW_SP_DEF3 | -28000 | Category 3 default low limit |
| UNDERFLOW_SP_AUTO | -32768 | Sentinel: use the category default low limit |

Parameter defaults reference the constants directly: `overflow_SP` uses `OVERFLOW_SP_DEF` and `underflow_SP` uses `UNDERFLOW_SP_AUTO`. All three platforms support this, and AI_Proc is written the same way:

- Portal: `zero_raw : Int := #S7_ZERO;`
- Step7: `zero_raw {S7_m_c := 'true'} : INT := S7_ZERO;` (constants declared in `CONST ... END_CONST`)
- PCS7: `} : INT := S7_ZERO;`

## Processing flow

1. Select `raw_zero`, `raw_min` and `def_low` from the constants by mode. Set `mode_err` when mode is outside 0 ~ 5.
2. On `mode_err` or `span = zero`, output 0 to AO, set invalid and stop.
3. High limit = MIN(overflow_SP, S7_AO_MAX); low limit follows the sentinel rule.
4. Scale with the general formula; clamp to the limits and set overflow / underflow when exceeded.
5. Round with REAL_TO_INT into AO; invalid = overflow OR underflow.
