# 🌿 iNaturalist Species Rarity Data Tool

This Python script uses the iNaturalist API to fetch a complete, filtered list of **wild species** observed in any geographic region (county, state, park, etc.) and visualizes the rarity distribution.

It ensures data quality by excluding observations flagged as captive or cultivated (e.g., zoo animals, garden plants).

## Features

* Fetches species name, scientific name, and observation count for a given Place ID.
* **Filters out** captive or cultivated observations (`captive=false`).
* Displays the **Top 10 Rarest Species** (lowest counts) in the console.
* Saves the complete, sorted dataset to a **CSV file**.
* Generates a **logarithmic histogram** to visualize the rarity distribution.

## 🚀 Getting Started

### Prerequisites

You must have [Python 3](https://www.python.org/downloads/) installed. Then, install the required libraries:

```bash
python -m pip install requests pandas matplotlib numpy
