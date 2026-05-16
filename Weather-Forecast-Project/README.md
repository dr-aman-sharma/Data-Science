# Wind Forecast Bias Correction Using Machine Learning

The objective of this project was to improve raw wind forecasts using a machine learning based correction model. The idea was to learn the systematic forecast bias by comparing historical forecasts with ERA5 reanalysis data, and then use this learned bias to generate corrected wind predictions. The study was performed across multiple Indian locations belonging to different terrain categories in order to examine whether forecast error depends on terrain and season. I selected 10 locations across 5 terrain categories. These were, coastal (Mumbai, Chennai), mountaineous (Leh, Shimla), deserts (Jaisalmer, Bikaner), plains (Delhi, Lucknow), and tropical (Kochi, Guwahati). The locations were chosen intentionally to expose the model to different atmospheric and terrain conditions, for example, deserts experience strong thermal turbulence, mountain regions have terrain-induced flow effects, coastal regions are influenced by land-sea interactions, and tropical regions experience high seasonal variability.

# Data Collection

Data was collected using Open-Meteo APIs. Two separate datasets were used: historical Forecast API which was used as the raw forecast source, and historical Weather API (ERA5 Reanalysis) which was used as the reference / ground truth dataset. Hourly data was collected for wind speed at 10 m, and wind direction at 10 m. Time period was chosen to be 1 January 2025 to 31 December 2025. All timestamps were aligned in GMT+0 to avoid timezone mismatch issues.


# Data Preparation and Feature Engineering

For each location forecast data and ERA5 data were merged using timestamp matching. Forecast bias was computed as delta = actual wind speed - forecast wind speed. The correction model was trained to predict this residual error. Terrain labels were also added manually for each location.

The following features were used for training: forecast wind speed, forecast wind direction, time and terrain. Since wind direction is circular, I encoded it using sine(direction) and cosine(direction), instead of directly using the angle. Time features were encoded to capture daily and seasonal cycles as "hour_sin", "hour_cos", "month_sin", "month_cos".
Terrain features included were coastal, desert, mountain, plains, and tropical. Terrain type was one-hot encoded.


# Model Used and Train-Test Split

I used a Random Forest Regressor for the correction model. Reasons for choosing it were that it works well on nonlinear tabular data, handles mixed feature types naturally and requires minimal preprocessing which was needed for the shortness of time. 

Instead of random splitting, I used a time-based split. I trained for January to October 2025 and tested for November to December 2025.


# Results

| Metric | Raw Forecast | Corrected Forecast |
|---|---|---|
| RMSE | 3.27 | 2.33 |
| MAE | 2.58 | 1.77 |
| Bias | -1.16 | 0.15 |

The correction model significantly reduced forecast error

Forecast bias clearly depended on terrain type.

| Terrain | Raw RMSE | Corrected RMSE |
|---|---|---|
| Coastal | 3.24 | 2.53 |
| Desert | 4.12 | 3.05 |
| Mountain | 2.91 | 1.62 |
| Plains | 3.04 | 2.05 |
| Tropical | 2.88 | 2.14 |

Observations: 

Desert regions had the highest forecast error. Mountain regions showed the strongest improvement after correction, and coastal and tropical regions remained relatively harder due to more complex local atmospheric behaviour. This suggests that terrain has a strong influence on forecast bias.

The monthly RMSE analysis (in the notebook) showed a clear seasonal dependence: raw forecast errors increased during mid-year months, the highest errors appeared around June–August, and the correction model consistently reduced error across all seasons. This behaviour is physically reasonable because summer and monsoon periods are associated with stronger atmospheric instability and higher wind variability. The corrected forecast was much more stable across seasons compared to the raw forecast.

The most important features learned by the model were:

| Feature | Importance |
|---|---|
| forecast_wind_speed | 0.290 |
| terrain_desert | 0.186 |
| dir_cos | 0.115 |
| dir_sin | 0.107 |

This indicates, that forecast magnitude itself strongly influences forecast bias. We also observe that desert terrain contributes strongly to systematic forecast error, and wind direction contains important residual information, 


# Discussion

If deploying this model on an autonomous drone flying across mixed terrain, I would want to add additional information such as: land-use information, higher-resolution weather forecasts, pressure and temperature information. I would also likely retrain the model continuously using incoming flight data. An estimate of uncertainty is missing which i would like to add for safer operational deployment.

Assumptions: ERA5 reanalysis was treated as ground truth. I assumed Forecast and ERA5 timestamps to be correctly aligned after timezone normalization. Terrain was oversimplified into discrete categories.

Limitations: I used only one year of data. Only 10 locations were included. Extreme wind events were difficult to predict. I did not have time to implement wind direction correction.


# What I Would Do With More Time

Given more time, I would implement full wind direction correction. I would use additional meteorological variables, test gradient boosting / XGBoost models, study forecast lead-time dependence, include uncertainty estimation, evaluate performance during extreme weather events.


# Conclusion

This project showed that a simple machine learning based residual correction can significantly improve wind forecast quality across multiple terrain types and seasons. The model reduced RMSE and MAE consistently. The analysis also showed that both terrain and season strongly affect forecast performance. Overall, the results suggest that terrain-aware ML post-processing can be useful for improving operational wind forecasting systems.
