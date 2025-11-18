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

This is a great approach for sharing your work! Placing the script on GitHub and providing a comprehensive guide ensures maximum usability.

Here is the user guide formatted in Markdown, which is ideal for a GitHub repository's README.md file and is easily downloadable.

📚 iNaturalist Species Rarity Data Tool
This Python script fetches, processes, and visualizes species observation data from iNaturalist for any specified region (Place ID), focusing only on wild observations to determine species rarity.

Prerequisites
To run this script, you must have Python installed, along with four key libraries.

Install Python: Ensure you have Python 3.6+ installed.

Install Libraries: Open your terminal or PowerShell and run the following command to install all necessary packages:

Bash

python -m pip install requests pandas matplotlib numpy
1. The Script (inaturalist_data_tool.py)
Save the following code as inaturalist_data_tool.py:

Python

import requests
import csv
import time
import os
import sys
import pandas as pd
import matplotlib.pyplot as plt
import numpy as np
import glob

# --- Configuration ---
BASE_URL = "https://api.inaturalist.org/v1/observations/species_counts"
PER_PAGE = 500  # Max results per page for species_counts
# ---------------------

def get_species_counts(place_id):
    """Fetches all wild species and observation counts for the specified place ID."""
    
    all_species_data = []
    page = 1
    total_results = None
    
    print(f"--- Step 1: Fetching Data for Place ID {place_id} ---")

    while True:
        params = {
            'place_id': place_id,
            'verifiable': 'any',
            'captive': 'false', # Exclude captive/cultivated observations
            'per_page': PER_PAGE,
            'page': page
        }
        
        try:
            response = requests.get(BASE_URL, params=params)
            response.raise_for_status()
            data = response.json()
        except requests.exceptions.RequestException as e:
            print(f"\n--- ERROR ---")
            print(f"Error fetching data from API page {page}: {e}")
            return None

        if total_results is None:
            total_results = data.get('total_results', 0)
            if total_results == 0:
                print(f"No wild species results found for Place ID {place_id}.")
                return None
            print(f"Total unique wild species found: {total_results}")

        for result in data.get('results', []):
            taxon = result.get('taxon', {})
            common_name = taxon.get('preferred_common_name')
            scientific_name = taxon.get('name')
            observation_count = result.get('count')
            
            display_name = common_name if common_name else scientific_name
            
            if display_name and observation_count is not None:
                all_species_data.append({
                    'Species': display_name,
                    'Scientific_Name': scientific_name,
                    'Observation_Count': observation_count
                })

        records_retrieved = len(all_species_data)
        percent_complete = round((records_retrieved / total_results) * 100)
        print(f"Retrieved {records_retrieved} / {total_results} species ({percent_complete}%)...")

        if records_retrieved >= total_results:
            break
        
        page += 1
        time.sleep(1) 

    return all_species_data

def save_and_display_data(data, filename, place_id):
    """Sorts data, displays top 10 rarest, and saves to CSV."""
    if not data:
        print("No data to save or display.")
        return
        
    data.sort(key=lambda x: x['Observation_Count'], reverse=False)
    
    print("-" * 50)
    print(f"Top 10 Rarest WILD Species in Place ID {place_id}:")
    for i, item in enumerate(data[:10]):
        print(f"{i+1}. {item['Species']} (Count: {item['Observation_Count']})")
    print("-" * 50)
    
    keys = ['Species', 'Scientific_Name', 'Observation_Count']
    try:
        with open(filename, 'w', newline='', encoding='utf-8') as output_file:
            dict_writer = csv.DictWriter(output_file, fieldnames=keys)
            dict_writer.writeheader()
            dict_writer.writerows(data)
        
        print(f"\n--- Step 2: Data Saved ---")
        print(f"Exported all {len(data)} wild species to: {os.path.abspath(filename)}")
        return True
    except IOError as e:
        print(f"I/O Error while writing CSV: {e}")
        return False


def plot_count_distribution(filename, place_id):
    """Generates and saves a histogram of the observation counts using a logarithmic scale."""
    
    print(f"\n--- Step 3: Generating Plot ---")

    try:
        df = pd.read_csv(filename)
    except Exception as e:
        print(f"Error loading CSV file for plotting: {e}")
        return

    # --- Create Logarithmic Bins ---
    min_count = df['Observation_Count'].min()
    max_count = df['Observation_Count'].max()
    
    # Generate 15 logarithmically spaced bins, starting at min_count
    start_log = np.log10(min_count if min_count > 0 else 1)
    end_log = np.log10(max_count)
    log_bins = np.logspace(start_log, end_log, 15)
    log_bins = np.unique(log_bins.astype(int))

    plt.figure(figsize=(10, 6))
    
    # Plot the histogram with both X and Y axes on a log scale
    plt.hist(df['Observation_Count'], bins=log_bins, edgecolor='black', log=True, color='#1f77b4')
    
    # --- Formatting and Save ---
    
    plt.xscale('log')
    plt.title(f'Distribution of Wild Species Observation Counts (Place ID: {place_id})')
    plt.xlabel('Observation Count (Log Scale)')
    plt.ylabel('Number of Species (Log Scale)')
    plt.grid(axis='y', alpha=0.5)
    
    # Set custom X-axis ticks to be readable powers of 10
    ticks = [1, 10, 100, 1000, 10000, 100000]
    plt.xticks([t for t in ticks if t <= max_count and t >= min_count], [f'{t:,}' for t in ticks if t <= max_count and t >= min_count])
    
    output_image = f"iNat_Distribution_Place_{place_id}.png"
    plt.savefig(output_image)
    plt.close()
    
    print(f"Successfully created histogram: {output_image}")

if __name__ == '__main__':
    # --- Dynamic Place ID Handling ---
    DEFAULT_PLACE_ID = 2685 # Bexar County
    place_id = DEFAULT_PLACE_ID

    if len(sys.argv) > 1:
        # 1. Try to read from command-line argument
        try:
            place_id = int(sys.argv[1])
            print(f"Using Place ID from command-line argument: {place_id}")
        except ValueError:
            print(f"Invalid Place ID provided as argument. Using default: {DEFAULT_PLACE_ID}")
    else:
        # 2. Prompt user if no argument is given
        user_input = input(f"Enter the iNaturalist Place ID (default is {DEFAULT_PLACE_ID}): ")
        if user_input.strip():
            try:
                place_id = int(user_input.strip())
            except ValueError:
                print(f"Invalid input. Using default: {DEFAULT_PLACE_ID}")
        else:
            print(f"No input provided. Using default: {DEFAULT_PLACE_ID}")

    # Set dynamic file names
    OUTPUT_CSV = f"iNat_Data_Place_{place_id}.csv"

    # --- Execute workflow ---
    species_data = get_species_counts(place_id)
    
    if species_data:
        if save_and_display_data(species_data, OUTPUT_CSV, place_id):
            plot_count_distribution(OUTPUT_CSV, place_id)

    print("\nScript Finished.")
2. GitHub README Guide
You can use the following text as the content for your README.md file on GitHub.

Markdown

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

### Step 1: Find Your Place ID
The Place ID is the unique numerical identifier for a region on iNaturalist.

Go to the iNaturalist Observations page.

Use the search bar to find your desired location (e.g., "Big Bend National Park").

Once the results load, look at the URL in your browser. The Place ID is the number after place_id=.

Example URL: ...observations?place_id=**2685**&...

Place ID: 2685 (Bexar County, TX)

### Step 2: Run the Script
Save the code above as a file named inaturalist_data_tool.py

Open your terminal, navigate to the directory where you saved the file, and run the script, providing your Place ID as an argument:
