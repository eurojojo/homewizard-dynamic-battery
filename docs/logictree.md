# 📘 Logical Structure of the Automation  
*(Decision Tree Overview)*

This document describes the full decision logic used to control the HomeWizard Plug-In Battery, based on ENTSO-e electricity prices, PV export, household load, and battery state of charge.  
It corresponds 1:1 with the implementation in [**`automations/advanced_pricing_sensors.yaml`**](automations/advanced_pricing_sensors.yaml).

The automation uses **percentile-based thresholds** derived from the daily price curve:

- `low_threshold` ≈ Q1 (25th percentile) → **cheap hours**  
- `high_threshold` ≈ Q3 (75th percentile) → **expensive hours**  

Between these thresholds lies the **mid-zone**, where behaviour is more conservative and focused on self-consumption.

---

## 1. Super-heavy load (e.g. EV charging, > 9 kW)

- **Condition:**  
  `import_power > super_heavy_load_threshold`  
- **Action:**  
  `standby`  
- **Reason:**  
  When the household connection is heavily loaded (typical for EV charging), the battery must stay completely out of the way and not add any extra current.

---

## 2. PV export hack (wake-up from standby)

- **Condition:**  
  `export_power > min_export_power AND soc < 98`  
- **Action:**  
  `zero`  
- **Reason:**  
  This is the “charge-only” hack: whenever there is real PV surplus and the battery is not full, the mode is forced to `zero` so that the surplus can be absorbed, even if the battery was previously in `standby`. Price level does **not** matter here.

---

## 3. Arbitrage not profitable (spread too small)

- **Condition:**  
  `rt_profit <= min_profit`  

Inside this branch:

### 3.1 Heavy load, not in expensive zone

- **Condition:**  
  `import_power > heavy_load_threshold AND current < high_threshold`  
- **Action:**  
  `standby`  
- **Reason:**  
  When prices are not truly expensive and arbitrage is barely profitable, the battery should avoid extra stress during heavy household loads.

### 3.2 No heavy load, PV export available

- **Condition:**  
  `export_power > min_export_power`  
- **Action:**  
  `zero`  
- **Reason:**  
  Use the battery for self-consumption only: soak up PV surplus instead of exporting it.

### 3.3 No heavy load, no PV export, price not expensive

- **Condition:**  
  `current < high_threshold`  
- **Action:**  
  `standby`  
- **Reason:**  
  With no export and no significant price signal, the safest behaviour is to idle.

### 3.4 Otherwise (expensive zone)

- **Condition:**  
  *(implicit `else` in this branch)*  
- **Action:**  
  `zero`  
- **Reason:**  
  If we end up in the expensive zone despite low arbitrage profit, the battery may still discharge to shave the most expensive hours.

---

## 4. Expensive hours (above high_threshold)

- **Condition:**  
  `current > high_threshold`  
- **Action:**  
  `zero`  
- **Reason:**  
  Outside of the special “arbitrage not profitable” case above, expensive hours are always a signal to discharge and reduce peak price exposure.

---

## 5. Cheap hours + battery nearly full

- **Condition:**  
  `current <= low_threshold AND soc >= 98`  
- **Action:**  
  `standby`  
- **Reason:**  
  When prices are cheap but the battery is already almost full, it is better to stop cycling to preserve battery life and avoid pushing SoC beyond ~98%.

---

## 6. Cheap hours + PV export + not full

- **Condition:**  
  `current <= low_threshold AND soc < 98 AND export_power > min_export_power`  
- **Action:**  
  `zero`  
- **Reason:**  
  In cheap hours with PV surplus and remaining capacity, the battery should absorb the surplus instead of exporting.

---

## 7. Cheap hours + no PV export + not full

- **Condition:**  
  `current <= low_threshold AND soc < 98 AND export_power <= min_export_power`  
- **Action:**  
  `to_full`  
- **Reason:**  
  When there is no PV surplus but prices are cheap, the battery may be charged from the grid up to full.

---

## 8. Mid-zone (between low_threshold and high_threshold)

- **Condition:**  
  `low_threshold < current <= high_threshold`  

Inside this branch:

### 8.1 Mid-zone with PV export and room in the battery

- **Condition:**  
  `export_power > min_export_power AND soc < 98`  
- **Action:**  
  `zero`  
- **Reason:**  
  In mid-zone prices, the battery will still absorb PV surplus — but only when there is room left.

### 8.2 Mid-zone without PV export (or battery nearly full)

- **Action:**  
  `standby`  
- **Reason:**  
  When there is no surplus or little room left, the battery idles in mid-zone hours.

---

## 9. Fallback behaviour

The fallback applies if none of the above branches match (a safety net).

### 9.1 Very low battery (< 10% SoC)

- **Condition:**  
  `soc < 10`  
- **Action:**  
  `zero`  
- **Reason:**  
  A nearly empty battery must never get stuck in `standby`.  
  It must be allowed to charge as soon as conditions permit, independent of price quirks.

### 9.2 Otherwise (SoC ≥ 10%)

- **Action:**  
  `standby`  
- **Reason:**  
  For all other edge cases, `standby` is the safest default in the mid-zone.

---

This decision tree is fully aligned with the automation code, including:

- Percentile-based low/high thresholds (Q1/Q3)  
- Export-aware behaviour and the PV “wake-up” hack  
- Heavy and super-heavy load handling  
- Profitability logic for small or large price spreads  
- A robust fallback that prevents a nearly-empty battery from staying idle.
