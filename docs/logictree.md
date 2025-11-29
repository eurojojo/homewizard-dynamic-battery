# 📘 Logical Structure of the Automation  
*(Decision Tree Overview)*

This document describes the full decision logic used to control the HomeWizard Plug-In Battery, based on ENTSO-e electricity prices, PV export, household load, and battery state of charge.  

It corresponds 1:1 with the implementation in [**`automations/advanced_pricing_sensors.yaml`**](automations/advanced_pricing_sensors.yaml).

The automation uses **percentile-based thresholds** derived from the daily price curve:

- `low_threshold` ≈ Q1 (25th percentile) → **cheap hours**  
- `high_threshold` ≈ Q3 (75th percentile) → **expensive hours**  

Between these thresholds lies the **mid-zone**, where behaviour is more conservative and focused on self-consumption.

In addition, a daily **round-trip profit** estimate is computed from today’s minimum and maximum prices (including a tax component and round-trip efficiency). When the spread is too small, normal price arbitrage is effectively disabled.

---

## 1. Super-heavy load (e.g. EV charging, > 9 kW)

- **Condition:**  
  `import_power > super_heavy_load_threshold`  
- **Action:**  
  `standby`  
- **Reason:**  
  When the household connection is heavily loaded (typical for EV charging), the battery must stay completely out of the way and not add any extra current. Grid safety takes precedence over arbitrage or self-consumption.

---

## 2. PV export hack (wake-up from standby)

- **Condition:**  
  `export_power > min_export_power AND soc < 98`  
- **Action:**  
  `zero`  
- **Reason:**  
  This is the “charge-only” hack: whenever there is real PV surplus and the battery is not full, the mode is forced to `zero` so that the surplus can be absorbed, even if the battery was previously in `standby`.

  In other words: **PV export always wins** (as long as SoC < 98%), regardless of price zone or arbitrage profit. This compensates for the lack of a native “charge-only” mode in the HomeWizard app.

---

## 3. Arbitrage not profitable (spread too small)

- **Condition:**  
  `rt_profit <= min_profit`  

Here the expected round-trip profit of “buy low, sell high” (including taxes and 75% round-trip efficiency) is too small to be meaningful. In this situation the battery behaves very cautiously and mostly avoids cycling for price reasons.

Inside this branch:

### 3.1 Non-expensive hours

- **Condition:**  
  `current < high_threshold`  
- **Action:**  
  `standby`  
- **Reason:**  
  When prices are not in the expensive zone and arbitrage profit is marginal, the battery avoids extra wear and tear. PV surplus is still handled by Rule 2, so this branch mainly covers “normal” consumption without export.

### 3.2 Expensive hours

- **Condition:**  
  *(implicit “else” in this branch, i.e. `current >= high_threshold`)*  
- **Action:**  
  `zero`  
- **Reason:**  
  Even on low-spread days there can be a few clearly expensive hours. In those moments the battery is allowed to discharge to shave the worst peaks.

---

## 4. Expensive hours (above high_threshold)

- **Condition:**  
  `current > high_threshold`  
- **Action:**  
  `zero`  
- **Reason:**  
  When arbitrage *is* profitable (Rule 3 does **not** apply), expensive hours are a clear signal to discharge. The battery helps reduce exposure to high prices by supplying local demand.

---

## 5. Cheap hours + battery nearly full

- **Condition:**  
  `current <= low_threshold AND soc >= 98`  
- **Action:**  
  `standby`  
- **Reason:**  
  When prices are cheap but the battery is already almost full, it is better to stop cycling to preserve battery life and avoid pushing SoC beyond ~98%. The system effectively “parks” the battery until either SoC drops again or price conditions change.

---

## 6. Cheap hours + no PV export + not full

- **Condition:**  
  `current <= low_threshold AND soc < 98 AND export_power <= min_export_power`  
- **Action:**  
  `to_full`  
- **Reason:**  
  In cheap hours, with no PV surplus and room left in the battery, it is advantageous to charge from the grid. The battery is filled in anticipation of later, more expensive hours.  

  Cases with PV surplus in cheap hours are already captured by **Rule 2 (PV export hack)**, so they no longer appear here.

---

## 7. Mid-zone (between low_threshold and high_threshold)

- **Condition:**  
  `low_threshold < current <= high_threshold`  
- **Action:**  
  `standby`  
- **Reason:**  
  In the mid-zone, prices are neither clearly cheap nor clearly expensive. After the PV export hack (Rule 2) has had its chance to wake the battery for surplus solar, the remaining mid-zone behaviour is simple: the battery idles in `standby`.  

  This avoids unnecessary cycling in hours where the price signal is ambiguous.

---

## 8. Fallback behaviour

The fallback applies if none of the above branches match (a safety net).

### 8.1 Very low battery (< 10% SoC)

- **Condition:**  
  `soc < 10`  
- **Action:**  
  `zero`  
- **Reason:**  
  A nearly empty battery must never get stuck in `standby`.  
  It must be allowed to participate again (i.e. charge when conditions allow), independent of small glitches or edge cases in the other rules.

### 8.2 Otherwise (SoC ≥ 10%)

- **Action:**  
  `standby`  
- **Reason:**  
  For all other edge cases, `standby` is the safest default in the mid-zone. The battery remains available for future PV surplus or clear price signals (cheap or expensive hours).

---

This decision tree is fully aligned with the current automation code, including:

- Percentile-based low/high thresholds (Q1/Q3)  
- The PV “wake-up” hack that makes effective use of surplus solar  
- Heavy and super-heavy load handling  
- Profitability logic based on daily min/max prices and round-trip efficiency  
- A robust fallback that prevents a nearly empty battery from staying idle forever.

---

# 👤 Author & License

**Author:** Joost Smits  
**GitHub:** https://github.com/eurojojo  
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

You are free to use, modify and distribute this work, provided attribution is preserved.
