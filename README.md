# Nashville Airbnb – Price Modeling

This is a follow-up to my Nashville AirBnb Dashboard. The dashboard was aimed at describing what the Nashville short-term rental market looked like. This project moves from description to prediction, building models that estimate a listing's nightly price and identify which property, host, and location factors drive it. Furthermore, my goal is to build a model that predicts which listings are 'high demand'.

## Data
Inside Airbnb, Nashville detailed listings (`listings.csv.gz`), snapshot dated 19 September 2026. 
Not included in this repo. 

## Models
### Linear regression (nightly price): 

<ins>Cleaning Notebook</ins>

In this notebook, the raw dataset was imported and initial cleaning and variable reduction was performed. To start, variables were removed based on business logic, as some columns in the raw dataset cannot be used to predict nightly pricing or would not be known at the time of making a prediction. After checking for columns with over 50% missingness, only the license variable applied. However, it was kept as a predictor because the presence of a license may indicate a more professionally managed listing, which could be associated with higher prices. It was converted to a binary `has_license` flag after confirming that none of the text entries were placeholders such as "Exempt" or "Pending". Following this, variables with low variability were checked. Longitude and latitude showed low variability, but measuring in degrees is misleading: one degree of latitude is approximately 69 miles, and one degree of longitude at Nashville's latitude is around 55.8 miles. It made more sense to keep them and engineer a new feature, distance to downtown, which will be done in the feature engineering notebook. Lastly, all listings with a minimum stay of 28 or more nights were removed (1,369 listings), since Airbnb treats stays of 28+ nights as monthly rentals, which are outside the project scope.

<ins>Features Notebook</ins>

In this notebook, additional features were engineered to better predict listing prices. Only fixed, row-by-row transformations were done here. Imputation, dummy variables, scaling, and frequency-based grouping (such as merging rare neighborhoods) are not performed until after the train/test split to avoid leakage. 

Raw latitude and longitude do not help a linear model, since price does not rise or fall in a straight line as you move north or east. What matters is how close a listing is to the main tourist area, so both were converted into a distance from Lower Broadway and then split into four bands: 0-3 km (downtown core), 3-5 km (inner neighborhoods), 5-10 km (wider urban area), and 10+ km (outer suburbs). The raw coordinates were then dropped.

The amenities column was originally exported as a JSON list. To make it usable and easier to interpret, it was turned into a total amenity count plus nine flags for amenities that plausibly affect price. ex: pool, hot tub, gym, game room, fire pit, EV charger, pets allowed, city skyline view, and free parking on premises.

Room type was condensed into two groups, "Entire place" and "Room", since categories like "Shared room" had only 5 listings. Property type was grouped by building type into House, Apartment/Condo, Townhouse, Hotel/Resort, Guest Unit, and Other.

Lastly, since nightly price (the target) is heavily right-skewed, a log-transformed version (`log_price`) was created alongside the raw price. The result was saved as `listings_features.csv` for the modeling notebook.

<ins>Model Notebook</ins>

In this notebook, the engineered dataset was loaded and the linear regression model was built on `log_price`. Listings with a missing price were dropped, and the bathroom text (ex: "1.5 baths", "shared bath") was converted into a numeric `bathrooms` count plus a `shared_bath` flag, which avoids creating dozens of unnecessary dummy variables. Listings with more than 16 bedrooms or more than 16 bathrooms were then removed (26 listings), since no listing in the data sleeps more than 16 guests and entries such as 40 or 50 bathrooms are data errors. The data was then split 75/25 into a training set (6,128 listings) and a test set (2,043 listings). Everything that follows was learned from the training set only.

Missing values in the continuous variables (bedrooms, minimum nights, review score, and bathrooms) were filled with the training median, and a "was missing" flag was added for each so the model can tell imputed values from real ones. The medians were saved so the exact same values can be applied to the test set later. Categorical variables were then turned into dummy variables, leaving 66 candidate predictors.

Variable reduction happened in two steps. First, each predictor was tested against log price on its own (univariate F-test), and only those with a p-value below 0.001 were kept, a stricter cutoff than the usual 0.05 because of the sample size (Raftery). This left 40 predictors and an R-squared of 0.605. Second, backward, forward, and stepwise selection were run using 10-fold cross-validation. All three methods agreed on the same 34 features, so there was no need to choose between them. This model (R-squared of 0.603, adjusted R-squared of 0.600) was carried forward into diagnostics.

<ins>Diagnostics Notebook</ins>

In this notebook, the same data preparation was repeated so the results match, and the 34-feature model was checked against the standard linear regression assumptions.

- **Equal variance:** Failed. The Breusch-Pagan test returned a p-value of essentially zero, meaning the size of the prediction errors is not constant across the price range.
- **Normality:** Failed on the formal tests (Shapiro-Wilk p-value near zero, Anderson-Darling statistic of 11.2 against a 1% critical value of 1.035).
- **Independence of errors:** Passed. The Durbin-Watson statistic was 1.986, very close to the ideal value of 2.
- **Multicollinearity:** Acceptable. Bedrooms (VIF of 9.45) and bathrooms (7.95) overlap with each other, which makes sense since bigger listings have more of both. They are above a strict cutoff of 5 but below the common cutoff of 10, and both are core price drivers, so no variables were removed and the model was left unchanged.

The model was then used to predict the test set, with the test data prepared using the training medians and training dummy columns. On the log scale, test R-squared was 0.614 (in line with 0.603 in training, so there is no sign of overfitting), with an RMSE of 0.370 and an MAE of 0.289. Converted back to dollars, the typical miss was about $117 per night (MAE), or 29.5% (MAPE), with a dollar RMSE of $210. Before the bedroom and bathroom cutoff was applied, the dollar RMSE was $8,641 because a few listings with 40-50 bathrooms produced extreme predictions.

Lastly, outliers and influential listings were checked. 50 training listings had standardized residuals beyond ±3, and leverage and Cook's distance were used to find listings with outsized pull on the model. To test whether the model depends on a handful of listings, it was refit without the five most influential ones. No coefficient changed sign or significance, the largest shift was under one standard error (a neighborhood effect), and test R-squared and MAE were essentially unchanged (0.6138 vs 0.6132 and $117.35 vs $117.31). The model is stable, so the original was kept.

### Logistic regression (high demand): 