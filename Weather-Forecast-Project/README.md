# Wind Forecast Bias Correction Using Machine Learning

The objective of this project was to improve raw wind forecasts using a machine learning based correction model. The idea was to learn the systematic forecast bias by comparing historical forecasts with ERA5 reanalysis data, and then use this learned bias to generate corrected wind predictions. The study was performed across multiple Indian locations belonging to different terrain categories in order to examine whether forecast error depends on terrain and season. I selected 10 locations across 5 terrain categories. These were, coastal (Mumbai, Chennai), mountaineous (Leh, Shimla), deserts (Jaisalmer, Bikaner), plains (Delhi, Lucknow), and tropical (Kochi, Guwahati).

The locations were chosen intentionally to expose the model to different atmospheric and terrain conditions.

For example:
- deserts experience strong thermal turbulence,
- mountain regions have terrain-induced flow effects,
- coastal regions are influenced by land-sea interactions,
- tropical regions experience high seasonal variability.

---

# Data Collection

Data was collected using Open-Meteo APIs.

Two separate datasets were used:

1. Historical Forecast API
   Used as the raw forecast source.

2. Historical Weather API (ERA5 Reanalysis)
   Used as the reference / ground truth dataset.

Hourly data was collected for:
- wind speed at 10 m
- wind direction at 10 m

Time period:
- 1 January 2025 to 31 December 2025

All timestamps were aligned in GMT+0 to avoid timezone mismatch issues.

---

# Data Preparation

For each location:
- forecast data and ERA5 data were merged using timestamp matching,
- forecast bias was computed as:

delta = actual wind speed - forecast wind speed

The correction model was trained to predict this residual error.

Terrain labels were also added manually for each location.

---

# Feature Engineering

The following features were used for training:

### Forecast Variables
- forecast wind speed
- forecast wind direction

### Wind Direction Encoding
Since wind direction is circular, I encoded it using:
- sine(direction)
- cosine(direction)

instead of directly using the angle.

### Time Features
To capture daily and seasonal cycles:
- hour_sin
- hour_cos
- month_sin
- month_cos

were included.

### Terrain Features
Terrain type was one-hot encoded into:
- coastal
- desert
- mountain
- plains
- tropical

---

# Model Used

I used a Random Forest Regressor for the correction model.

Reasons for choosing it:
- works well on nonlinear tabular data,
- handles mixed feature types naturally,
- requires minimal preprocessing,
- relatively robust and interpretable.

---

# Train-Test Split

Instead of random splitting, I used a time-based split.

Training:
- January to October 2025

Testing:
- November to December 2025

This setup better simulates a real forecasting scenario and avoids temporal leakage.

---

# Results

## Overall Performance

| Metric | Raw Forecast | Corrected Forecast |
|---|---|---|
| RMSE | 3.27 | 2.33 |
| MAE | 2.58 | 1.77 |
| Bias | -1.16 | 0.15 |

The correction model significantly reduced forecast error and almost removed the systematic negative bias present in the raw forecasts.

---

# Terrain Dependence Analysis

Forecast bias clearly depended on terrain type.

| Terrain | Raw RMSE | Corrected RMSE |
|---|---|---|
| Coastal | 3.24 | 2.53 |
| Desert | 4.12 | 3.05 |
| Mountain | 2.91 | 1.62 |
| Plains | 3.04 | 2.05 |
| Tropical | 2.88 | 2.14 |

### Observations

- Desert regions had the highest forecast error.
- Mountain regions showed the strongest improvement after correction.
- Coastal and tropical regions remained relatively harder due to more complex local atmospheric behaviour.

This suggests that terrain has a strong influence on forecast bias.

---

# Temporal Dependence Analysis

The monthly RMSE analysis showed a clear seasonal dependence.

Main observations:
- Raw forecast errors increased during mid-year months.
- The highest errors appeared around June–August.
- The correction model consistently reduced error across all seasons.

This behaviour is physically reasonable because summer and monsoon periods are associated with stronger atmospheric instability and higher wind variability.

The corrected forecast remained much more stable across seasons compared to the raw forecast.

---

# Feature Importance

The most important features learned by the model were:

| Feature | Importance |
|---|---|
| forecast_wind_speed | 0.290 |
| terrain_desert | 0.186 |
| dir_cos | 0.115 |
| dir_sin | 0.107 |

This indicates that:
- forecast magnitude itself strongly influences forecast bias,
- wind direction contains important residual information,
- desert terrain contributes strongly to systematic forecast error.

---

# Operational Deployment Discussion

If deploying this model on an autonomous drone flying across mixed terrain, I would want additional information such as:

- elevation and terrain slope,
- land-use information,
- higher-resolution weather forecasts,
- pressure and temperature fields,
- real-time onboard sensor measurements,
- forecast lead time,
- wind gust forecasts.

I would also likely:
- retrain models continuously using incoming flight data,
- use separate models for different terrain regions,
- incorporate spatial information from nearby locations,
- estimate uncertainty for safer operational deployment.

---

# Assumptions

Some assumptions made during this work:

- ERA5 reanalysis was treated as ground truth.
- Forecast and ERA5 timestamps were assumed correctly aligned after timezone normalization.
- Terrain was simplified into discrete categories.
- Wind speed correction was modeled independently at each hourly timestep.

---

# Limitations

Some limitations of the current approach are:

- only one year of data was used,
- only 10 locations were included,
- extreme wind events remain difficult to predict,
- wind direction correction was not fully implemented,
- spatial relationships between locations were not explicitly modeled.

---

# What I Would Do With More Time

Given more time, I would:

- implement full wind direction correction,
- use additional meteorological variables,
- test gradient boosting / XGBoost models,
- study forecast lead-time dependence,
- include uncertainty estimation,
- incorporate spatial and topographic information,
- evaluate performance during extreme weather events.

---

# Conclusion

This project showed that machine learning based residual correction can significantly improve wind forecast quality across multiple terrain types and seasons.

The model reduced RMSE and MAE consistently while also reducing systematic forecast bias. The analysis also showed that both terrain and season strongly affect forecast performance.

Overall, the results suggest that terrain-aware ML post-processing can be useful for improving operational wind forecasting systems.
