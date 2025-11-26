# ENTSO-e Price Curve Percentile Approach  
### Using Percentiles and the Five-Number Summary for Dynamic Thresholds in Home Assistant

This document explains how to derive **dynamic low/high price thresholds** from a daily electricity price curve using **percentiles** based on the basics of the so-called **five-number summary**, and how these thresholds can be used in Home Assistant to control the HomeWizard Plug-In Battery.

Although ENTSO-e data is used as an example, this method works with *any* hourly price integration that exposes a list of prices (e.g. Nordpool, Tibber, EasyEnergy, Dynamic Energy Contract add-ons).

---

## 📈 Five-Number Summary
A simplified Home Assistant automation could use *fixed margins* around the lowest and highest electricity price of the day, such as:

- `low_margin = lowest + 0.03`
- `high_margin = highest - (spread / 2.5)`

While workable, fixed margins **don’t adapt** to the actual shape of the price curve.

For a distribution such as 24 hourly electricity prices, the **five-number summary** consists of:

- Minimum  
- **Q1** — 25th percentile  
- **Median** — 50th percentile  
- **Q3** — 75th percentile  
- Maximum  

This provides a compact statistical description of the price curve and divides it into meaningful regions. You can find it in visualisations like boxplots.

---

## 🎯 Goal of this method

We replace fixed logic like:

- `low_margin = lowest + 0.03`  
- `high_margin = highest - (spread / 2.5)`

with **statistical thresholds**:

- **Low threshold ≈ Q1 (25th percentile)** → “cheap hours”  
- **High threshold ≈ Q3 (75th percentile)** → “expensive hours”

The automation becomes dynamic and adjusts to the actual price distribution of the day.

---

## 🔍 Extracting the price curve

ENTSO-e sensors typically expose data like:

- `today` or `prices_today`
- a list of attributes
  each with `time` and `price`

Example (simplified):

```yaml
attributes:
  prices_today:
    - time: "2025-11-26 00:00:00"
      price: 0.10555
    - time: "2025-11-26 01:00:00"
      price: 0.10274
    ...
```

Any provider exposing a list of prices (Nordpool, Tibber, EasyEnergy, Dynamic-Contract add-ons) works identically.
We only need the array of hourly **price** values.

---

## 🧱 Architecture

We create two **template sensors**:

- `sensor.entso_e_low_price_threshold` → Q1 (approx. 25th percentile)  
- `sensor.entso_e_high_price_threshold` → Q3 (approx. 75th percentile)

These sensors:

- extract the price list  
- sort it  
- compute approximate percentile indices  
- update automatically when the underlying ENTSO sensor updates  
- impose **zero overhead** on your main battery automation

### Why not compute percentiles inside the automation?

Because your battery automation triggers every few minutes.  
Percentile computation should happen **only when price data updates**.

### Why not use Python scripts?

A Python script would work, but is unnecessary:  
template sensors in YAML are simpler, clearer, and version-controlled in your Git repo.

---

## 🛠 Template sensors for Q1 & Q3

Add the following to `configuration.yaml` or any `templates:` block:

```yaml
template:
  - sensor:

      - name: "ENTSO-e low price threshold"
        unique_id: entsoe_low_price_threshold
        unit_of_measurement: "€/kWh"
        state: >
          {% set prices = state_attr('sensor.entso_e_average_electricity_price', 'prices_today') %}
          {% if not prices %}
            unknown
          {% else %}
            {% set vals = prices
              | map(attribute='price')
              | select('number')
              | list %}
            {% set n = vals | length %}
            {% if n < 2 %}
              unknown
            {% else %}
              {% set sorted = vals | sort %}
              {# Approximate 25th percentile (Q1) #}
              {% set idx = (0.25 * (n - 1)) | int %}
              {{ sorted[idx] }}
            {% endif %}
          {% endif %}

      - name: "ENTSO-e high price threshold"
        unique_id: entsoe_high_price_threshold
        unit_of_measurement: "€/kWh"
        state: >
          {% set prices = state_attr('sensor.entso_e_average_electricity_price', 'prices_today') %}
          {% if not prices %}
            unknown
          {% else %}
            {% set vals = prices
              | map(attribute='price')
              | select('number')
              | list %}
            {% set n = vals | length %}
            {% if n < 2 %}
              unknown
            {% else %}
              {% set sorted = vals | sort %}
              {# Approximate 75th percentile (Q3) #}
              {% set idx = (0.75 * (n - 1)) | int %}
              {{ sorted[idx] }}
            {% endif %}
          {% endif %}
```

You can of course modify the 0.25 and 0.75 numbers to your liking.

### What these sensors do

- extract `prices_today`  
- filter valid numeric values  
- sort the list  
- calculate:
  - **Q1** at index `0.25 * (n-1)`  
  - **Q3** at index `0.75 * (n-1)`  
- provide the result as a normal Home Assistant sensor

This effectively splits the price curve into:

- **cheap hours** (below Q1)  
- **middle band** (between Q1 and Q3)  
- **expensive hours** (above Q3)

---

## 🔄 Integrating with the battery automation

In your Home Assistant automation, define:

```yaml
low_threshold: "{{ states('sensor.entso_e_low_price_threshold') | float(0) }}"
high_threshold: "{{ states('sensor.entso_e_high_price_threshold') | float(0) }}"
```

Replace fixed logic such as:

```yaml
current < (lowest + low_margin)
```

with:

```yaml
current <= low_threshold
```

Replace:

```yaml
current > (highest - high_margin)
```

with:

```yaml
current > high_threshold
```

Middle band:

```yaml
current > low_threshold and current <= high_threshold
```

Your automation no longer relies on fixed margins or spreads.  
It becomes fully data-driven.

---

## ✔ Summary

This percentile-based approach provides:

- dynamic low/high thresholds using **Q1 and Q3**  
- no reliance on hardcoded offsets  
- automatic updates only when price data updates  
- compatibility with any hourly price integration  
- simpler and cleaner battery automation  
- no need for Python scripts or external components

It is an elegant way to bring proper statistical analysis to energy automations in Home Assistant.
