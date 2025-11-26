# 📘 Logical Structure of the Automation  
*(Decision Tree Overview)*

This document describes the full decision logic used to control the HomeWizard Plug-In Battery, based on ENTSO-e electricity prices, PV export, household load, and battery state of charge.  
It corresponds 1:1 with the implementation in `advanced-homewizard-battery-price.yaml`.

---

## 1. Super-heavy load (e.g., EV charging, > 9 kW)
- **Condition:** `import_power > super_heavy_load_threshold`  
- **Action:** `standby`  
- **Reason:** The battery must stay fully out of the way when the household connection is heavily stressed (e.g., EV charging).

---

## 2. Arbitrage not profitable
- **Condition:** `rt_profit <= min_profit`  
- **If heavy load AND not in expensive hours:** `standby`  
- **Else:** `zero`  
- **Reason:** When the price spread is too small, complex behaviour is unnecessary — the battery behaves like a normal self-consumption buffer.

---

## 3. Expensive hours (above high_threshold)
- **Action:** `zero`  
- **Reason:** The battery may discharge to compensate expensive peak hours.

---

## 4. Cheap hours + battery nearly full
- **Condition:** `current ≤ low_threshold AND soc ≥ 98%`  
- **Action:** `standby`  
- **Reason:** Avoid pushing the battery above ~98%, preserving lifetime and avoiding unnecessary cycling.

---

## 5. Cheap hours + PV export + not full
- **Condition:**  
  `current ≤ low_threshold AND soc < 98 AND export_power > min_export_power`  
- **Action:** `zero`  
- **Reason:** PV surplus should be absorbed instead of exported.

---

## 6. Cheap hours + no PV export + not full
- **Condition:**  
  `current ≤ low_threshold AND soc < 98 AND export_power ≤ min_export_power`  
- **Action:** `to_full`  
- **Reason:** Charging from the grid is desirable when prices are low.

---

## 7. Mid-zone (between low_threshold and high_threshold)
- **Condition:**  
  `low_threshold < current ≤ high_threshold`  
- **If PV export AND not full:** `zero`  
- **Else:** `standby`  
- **Reason:** In moderate-price hours, charge only from solar; otherwise preserve SoC.

---

## 8. Fallback behaviour

### 8a. Very low battery (< 10% SoC)
- **Action:** `zero`  
- **Reason:** A nearly empty battery must never remain stuck in `standby`.  
  It must always be allowed to charge in the next suitable window.

### 8b. Otherwise (SoC ≥ 10%)
- **Action:** `standby`  
- **Reason:** Safe default for the mid-zone when no explicit branch matches (if ever).

---

This decision tree is fully aligned with the automation code, including:
- Percentile-based low/high thresholds  
- Export-aware behaviour  
- Heavy and super-heavy load handling  
- Profitability logic  
- Smart fallback for <10% SoC  
