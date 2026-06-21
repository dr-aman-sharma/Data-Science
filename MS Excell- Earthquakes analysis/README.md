# South Asia Earthquake Analysis (2020–2025)

## Overview

This project explores earthquake activity in South Asia between 2020 and 2025 using data obtained from the USGS Earthquake Catalog. The objective was to perform an exploratory data analysis using Microsoft Excel and identify patterns in earthquake magnitude, depth, location, and occurrence over time.

The dataset contains 5,410 earthquake records with magnitudes ranging from 2.6 to 7.7.

---

## Dataset

Source: USGS Earthquake Catalog

Region: South Asia

Period Covered: January 2020 – December 2025

Number of Records: 5,410

Variables used in the analysis:

* Date and Time
* Latitude
* Longitude
* Depth (km)
* Magnitude
* Magnitude Type
* Number of Seismic Stations (nst)

---

## Data Preparation

Before performing the analysis, the dataset was inspected for missing values and formatting issues.

The following steps were carried out:

* Converted timestamps into separate Date and Time columns.
* Created Year and Month-Year fields for time-series analysis.
* Checked for missing values in key variables.
* Verified numerical ranges for depth, magnitude, and location coordinates.
* Examined the distribution of magnitude types.

No missing values were found in the primary variables used for the analysis.

---

## Summary Statistics

| Variable           | Minimum | Maximum | Mean  |
| ------------------ | ------- | ------- | ----- |
| Magnitude          | 2.6     | 7.7     | 4.44  |
| Depth (km)         | 3.16    | 300.15  | 53.58 |
| Number of Stations | 5       | 423     | 51.43 |

A weak negative correlation was observed between earthquake depth and magnitude:

Correlation (Depth, Magnitude) = -0.225

This indicates that deeper earthquakes tend to be slightly weaker on average, although the relationship is not strong.

---

## Magnitude Distribution

The majority of recorded earthquakes had magnitudes between 4 and 5.

Higher magnitude earthquakes were considerably less frequent, with only a small number of events exceeding magnitude 6.5.

This behaviour is consistent with the general observation that small and moderate earthquakes occur much more frequently than large earthquakes.

![Magnitude Histogram](images/magnitude_histogram.png)

---

## Depth Distribution

Most earthquakes occurred at relatively shallow depths.

Approximately 60% of all recorded events were located within the shallowest depth interval of the dataset.

A smaller number of earthquakes were observed at intermediate and deep depths.

![Depth Histogram](images/depth_histogram.png)

---

## Magnitude and Depth Relationship

A scatter plot was created to investigate whether deeper earthquakes tend to have larger or smaller magnitudes.

The results show a weak negative relationship between depth and magnitude. Large earthquakes were observed primarily at shallow depths, while deeper earthquakes generally occurred within a narrower magnitude range.

![Magnitude vs Depth](images/mag_vs_depth.png)

---

## Earthquake Activity Through Time

Monthly earthquake counts were calculated from 2020 to 2025.

The overall level of seismic activity remained relatively stable throughout the study period. Although several months experienced elevated activity, no clear long-term increase or decrease in earthquake occurrence was observed.

![Monthly Earthquake Activity](images/monthly_activity.png)

---

## Spatial Distribution

Earthquake locations were plotted using latitude and longitude coordinates.

Several distinct clusters can be observed across South Asia. These clusters are associated with major tectonic regions including:

* The Himalayan seismic belt
* Afghanistan–Pakistan region
* Myanmar region
* Andaman–Nicobar region

These areas correspond to active plate boundaries and zones of ongoing tectonic deformation.

![Earthquake Map](images/earthquake_map.png)

---

## Yearly Summary

| Year | Average Magnitude | Average Depth (km) | Number of Events |
| ---- | ----------------- | ------------------ | ---------------- |
| 2020 | 4.4               | 57.1               | 828              |
| 2021 | 4.5               | 54.4               | 876              |
| 2022 | 4.5               | 51.3               | 937              |
| 2023 | 4.4               | 51.8               | 959              |
| 2024 | 4.4               | 66.1               | 782              |
| 2025 | 4.5               | 44.3               | 1028             |

The average earthquake magnitude remained remarkably stable over the six-year period, while average depth showed larger year-to-year variations.

---

## Key Findings

* The dataset contains 5,410 earthquakes recorded between 2020 and 2025.
* Most earthquakes had magnitudes between 4 and 5.
* Earthquake depths ranged from approximately 3 km to 300 km.
* Around 60% of events occurred at shallow depths.
* A weak negative correlation (-0.225) exists between earthquake depth and magnitude.
* Monthly earthquake activity remained broadly stable during the study period.
* Earthquakes clustered around major tectonic boundaries within South Asia.

---

## Tools Used

* Microsoft Excel
* Pivot Tables
* Scatter Plots
* Histograms
* Descriptive Statistics
* Correlation Analysis

---

## Author

Aman Sharma

This project was completed as a personal data analysis exercise to practice data cleaning, exploratory data analysis, visualization, and interpretation of geophysical datasets using Microsoft Excel.
