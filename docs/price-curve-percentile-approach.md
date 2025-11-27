# ENTSO-e Price Curve Percentile Approach  
### Using Percentiles and the Five-Number Summary for Dynamic Thresholds in Home Assistant

This document explains how to derive **dynamic low/high price thresholds** from a daily electricity price curve using **percentiles** and the idea of the **five-number summary**, and how those thresholds can be used in Home Assistant to control the HomeWizard Plug-In Battery.

Although ENTSO-e data is used as an example, this method works with *any* hourly price integration that exposes a list of prices (for example Nordpool, Tibber, EasyEnergy, Dynamic Energy Contract add-ons).

---

## 📈 Five-Number Summary

A simple Home Assistant automation can work with *fixed margins* around the daily minimum and maximum electricity price, such as:

- `low_margin = lowest + 0.03`  
- `high_margin = highest - (spread / 2.5)`

This is workable, but it does **not adapt** to the actual *shape* of the price curve.  

On some days almost all prices are clustered in a narrow band; on other days there are very deep valleys and sharp peaks.

For any distribution (including a list of 24 hourly prices), the classic **five-number summary** consists of:

- Minimum  
- **Q1** — 25th percentile  
- **Median** — 50th percentile  
- **Q3** — 75th percentile  
- Maximum  

This gives a compact statistical description of the curve and splits it into meaningful regions. In data visualisation you see this in [`boxplots`](https://en.wikipedia.org/wiki/Box_plot): the box is Q1–Q3, the line is the median, and the “whiskers” are min and max.

---

## 🎯 Goal of this method

We replace fixed logic like:

- `low_margin = lowest + 0.03`  
- `high_margin = highest - (spread / 2.5)`

with **statistical thresholds** based on today’s price distribution:

- **Low threshold ≈ Q1 (25th percentile)** → “cheap hours”  
- **High threshold ≈ Q3 (75th percentile)** → “expensive hours”

Because Q1 and Q3 are derived from the actual curve of the day, the automation:

- automatically adapts when prices are almost flat;
- still behaves sensibly when there is a very deep low or a very high peak;
- does not need any hardcoded “3 cent” or “half the spread” magic numbers.

These thresholds are then used by the battery automation to decide when to:

- charge from PV surplus,
- charge from the grid,
- discharge to support expensive hours,
- or stay in standby.

---

## 🔢 Getting the right price series

The ENTSO-e integration exposes several attributes on the average price sensor, typically:

- `prices_today` — prices for “today” (UTC-based day window)  
- `prices_tomorrow` — the next day  
- `prices` — a rolling window that spans more than one day

In winter, the UTC day does **not** perfectly match the local day in the Netherlands, and
`prices_today` may temporarily miss the last local hour until the next update is fetched.

To keep the configuration simple and predictable for most users, this project:

- uses **`prices_today`** as the base series for *today*;
- always derives min, max, Q1 and Q3 from that same list;
- recomputes these values automatically when ENTSO-e refreshes the prices.

For most Home Assistant setups this is sufficient and keeps the templates readable.
If you need stricter local-day handling you can adapt the templates to filter the more general `prices` attribute by timestamp.

---

## 🧱 Sensor architecture

All required template sensors are defined in:

> [**`automations/advanced_pricing_sensors.yaml`**](../automations/advanced_pricing_sensors.yaml)

You can open that file in your editor to see the full YAML. It defines the following sensors:

1. `sensor.entso_e_min_price_today`  
   - Minimum price of **today**, based on `prices_today`.

2. `sensor.entso_e_max_price_today`  
   - Maximum price of **today**, based on `prices_today`.

3. `sensor.entso_e_low_price_threshold`  
   - Approximate **Q1** (25th percentile) of today’s price curve.  
   - Used as the **“cheap” threshold** in the automations.

4. `sensor.entso_e_high_price_threshold`  
   - Approximate **Q3** (75th percentile) of today’s price curve.  
   - Used as the **“expensive” threshold** in the automations.

5. `sensor.entso_e_round_trip_profit_today`  
   - A derived value that estimates whether price arbitrage is worth it for today, after taxes and battery round-trip losses.

## 📊 How Q1 and Q3 are calculated

In the YAML, Q1 and Q3 are computed by:

1. Taking `prices_today` from `sensor.entso_e_average_electricity_price`.
2. Extracting the `price` value from each entry.
3. Sorting the list of numeric prices.
4. Picking an index based on the desired percentile.

For example, for Q1:

```jinja
{% set n = vals | length %}
{% set sorted = vals | sort %}
{% set idx = (0.25 * (n - 1)) | int %}
{{ sorted[idx] }}
```

This “index-based” approach is a simple approximation of the 25th percentile and is more than precise enough for 24 hourly prices.

Q3 uses the same idea with 0.75 instead of 0.25 to obtain the 75th percentile.

## 💰 Round-trip profit sensor

The project also exposes:

- `sensor.entso_e_round_trip_profit_today`

This sensor estimates whether **charging at today’s minimum price and discharging at today’s maximum price** is profitable once you include:

- **energy taxes**  
- **round-trip efficiency of the HomeWizard Plug-In Battery**

The YAML uses two hardcoded values:

- `tax = 0.14` €/kWh  
  - A practical approximation for the combined Dutch energy tax + VAT on a kWh.  
  - Adjust this if your tax regime is different.

- Round-trip efficiency = **75%** (`0.75`)  
  - Based on the typical round-trip efficiency of the HomeWizard Plug-In Battery.  
  - If you have newer hardware or your own measured efficiency, you can change this.

Adjust if you have better measurements for your setup.

The formula in the template is:

```text
(highest + tax) * 0.75 - (lowest + tax)

The difference is the theoretical profit per kWh of a perfect “charge low / discharge high” cycle.

If this round-trip profit is smaller than a chosen threshold (for example 0.02 €/kWh), the advanced battery automation can decide to ignore arbitrage for that day and simply use the battery for self-consumption, avoiding unnecessary cycling.

🛠 Installing the sensors in Home Assistant

To make these sensors available in your own Home Assistant instance you need to add the template block from this repository to your configuration.yaml.

1. Locate your Home Assistant configuration
On a typical installation this is the folder that contains configuration.yaml, automations.yaml, etc. For example:

Home Assistant OS / Container: usually /config

Home Assistant Core (venv): often ~/.homeassistant

Open configuration.yaml in a text editor.

2. Add (or extend) the template: section
In this repository the complete template block lives in:

[**`automations/advanced_pricing_sensors.yaml`**](../automations/advanced_pricing_sensors.yaml)

Open that file and copy the entire template: block.

Now in your own configuration.yaml:

If you do not yet have a template: key
you can paste the block at the top level, for example:

```yaml
template:
  - sensor:
      # Paste the sensor definitions from
      # automations/advanced_pricing_sensors.yaml here
```

If you already have a template: section
merge the - sensor: list from this project into your existing one.
The important rule is: do not create a second top-level template: key, instead
append the new - sensor: entries to your existing list.

If you prefer to keep the YAML in a separate file, you can also refactor your config to use
!include, for example:

```yaml
template: !include path/to/your/templates.yaml
```

and then place the - sensor: block from automations/advanced_pricing_sensors.yaml
into that included file. The exact include structure depends on how your configuration
is organised.

3. Check the configuration and reload  

In Home Assistant:

Go to Settings → System → Developer tools → Check configuration and make sure
there are no template errors.

Restart Home Assistant, or (on recent versions) use Developer tools → YAML → Reload
Template Entities.

## ✔ Summary

This percentile-based approach provides:

- dynamic low/high thresholds using **Q1 and Q3**  
- no reliance on hardcoded offsets  
- automatic updates only when price data updates  
- compatibility with any hourly price integration  
- simpler and cleaner battery automation  
- no need for Python scripts or external components

It is an elegant way to bring proper statistical analysis to energy automations in Home Assistant.
