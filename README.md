# HomeWizard Dynamic Battery Control for Home Assistant  
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A complete, modular automation system that turns the **HomeWizard Plug-In Battery** into a *smart, price-aware, PV-aware* device — even though HomeWizard does **not** yet support “charge-only” or “discharge-only” modes.

This project contains two automation strategies (simple & advanced), full documentation, and examples showing how these automations keep your power usage flat while exploiting dynamic electricity prices.

---

# 🌍 Why this project exists

HomeWizard currently exposes only **three modes** for the Plug-In Battery:

- **zero** — charge or discharge to keep net usage at 0 W  
- **to_full** — charge to 100%  
- **standby** — do nothing (not charge, nor discharge)  

There is *no mode* for:

- **charge only (block discharge)**  
- **discharge only (block charge)**  

This makes automation **necessary** if you want to:

- charge the battery when power is cheap  
- discharge when prices are high  
- prefer grid charging when there is no sun  
- avoid draining the battery during large appliance loads, especially during cheap hours  
- avoid overloading during EV charging  
- keep behaviour stable when prices fluctuate  

This repository provides a solution to that missing functionality, fully configurable.

---

# 📸 Examples

### 1. Flattened consumption curve (HomeWizard app example)
*Shows how the Plug-In Battery kept household usage flat while the EV charged during the night.*

![Flattened usage curve](docs/img/flattened-curve.png)

### 2. Daily electricity prices curve
*Shows a daytime mini-peak, a midday dip, and a strong evening price spike.*

![Daily prices (NextEnergy app)](docs/img/flattened-usage-curve.png)

---

# 📦 Repository Structure

```
homewizard-dynamic-battery/
│
├── automations/
│   ├── simple-homewizard-battery-price.yaml
│   └── advanced-homewizard-battery-price.yaml
│
├── docs/
│   ├── logictree.md
│   └── price-curve-percentile-approach.md
│
├── README.md   ← this file
└── LICENSE
```

---

# ⚡ Automations

The repository contains **two** fully documented automations.

---

## ✅ Simple Automation  
**File:** `automations/simple-homewizard-battery-price.yaml`

Uses:  
- fixed *cheap* threshold = `lowest_price + 0.03 EUR`  
- fixed *expensive* threshold = `highest_price − (spread / 2.5)`  

Advantages:  
- no template sensors  
- easy to understand and adjust  
- perfect for users who want quick setup  

This automation still includes:

- heavy load protection  
- super-heavy load override  
- SoC-aware behaviour  
- PV-aware charging  
- arbitrage profitability logic  

---

## 🔥 Advanced Automation  
**File:** `automations/advanced-homewizard-battery-price.yaml`

This version uses **statistical analysis** of the daily ENTSO-e price curve to compute dynamic thresholds.

Instead of fixed margins, the automation uses:

- **Q1 (25th percentile)** → “cheap hours”  
- **Q3 (75th percentile)** → “expensive hours”  

### Why percentiles?

Fixed margins assume the price curve behaves the same every day — but dynamic contracts often show:

- long cheap valleys  
- short sharp morning peaks  
- double evening peaks  
- flat days  
- extremely volatile weekends  

Percentiles make thresholds **curve-shaped**, not hardcoded.

### How it works  

The advanced automation uses two template sensors:

- `sensor.entso_e_low_price_threshold` ≈ Q1  
- `sensor.entso_e_high_price_threshold` ≈ Q3  

Described in:

📄 `docs/price-curve-percentile-approach.md`

These sensors only recalculate when ENTSO-e updates its prices (typically once per day).  
The main automation reuses the thresholds, making it efficient and stable.

### Full decision tree

The decision tree provides a **clear, human-readable overview** of how the automation behaves in every possible situation.  
It is the conceptual model behind both the simple and advanced automations.  
It reflects the intended behaviour in real-world scenarios, independent of implementation details.

While the YAML files contain the precise implementation, the structure of the logic can be difficult to follow directly from code.  

Using a decision tree helps for:

#### 1. **Clarity**
It breaks complex logic into intuitive branches. For example "What happens during cheap hours?"  

#### 2. **Transparency**
Every condition that affects the battery state is shown explicitly, like heavy and super-heavy load overrides and thresholds for cheap / midzone / expensive hours  

#### 3. **Safety and Predictability**
The tree makes it easy to verify that:  
- the battery stays out of the way during EV charging  
- it always uses solar when available  
- it avoids unnecessary cycling near 100% SoC  
- it only performs arbitrage when it is profitable

See the decision tree in `docs/logictree.md`  

---

# 🧠 Under the Hood: The Five-Number Summary

The advanced version uses the statistical concept of the **five-number summary**:

- Minimum  
- **Q1** (25th percentile)  
- Median  
- **Q3** (75th percentile)  
- Maximum  

From this we derive robust thresholds for:

- cheap (≤ Q1)  
- expensive (≥ Q3)  

A full explanation is included in:  
📄 `docs/price-curve-percentile-approach.md`

---

# 📝 Requirements

- Home Assistant (Docker or OS)  
- ENTSO-e integration  
- HomeWizard P1 integration  
- Plug-In Battery group mode entity **enabled** in HA  
- For advanced version: the two template sensors in your `configuration.yaml`  

---

# 🚀 Installation

### 1. Copy one automation  
Into your Home Assistant configuration:  

```
config/automations/
```

### 2. Reload automations  
`Settings → Developer Tools → YAML → Reload Automations`

### 3. (Advanced only) Add template sensors  
The documentation contains ready-to-paste YAML.

### 4. Optional tuning  
You *may* adjust:  
- heavy-load thresholds  
- export threshold  
- minimum profit requirement  
- SoC thresholds  

Defaults work for most households.

---

# 👤 Author & License

**Author:** Joost Smits  
**GitHub:** https://github.com/eurojojo  
**License:** MIT  
![MIT Badge](https://img.shields.io/badge/License-MIT-yellow.svg)

You are free to use, modify and distribute this work, provided attribution is preserved.

---

# 🤝 Contributing

Suggestions, improvements and pull requests are welcome!  
This automation continues to improve as HomeWizard updates its API and as more users adopt dynamic charging strategies.

