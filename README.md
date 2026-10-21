# Nashville Airbnb – Price Modeling

This is a follow-up to my Nashville AirBnb Dashboard. The dashboard was aimed at describing what the Nashville short-term rental market looked like. This project moves from description to prediction, building models that estimate a listing's nightly price and identify which property, host, and location factors drive it. Furthermore, my goal is to build a model that predicts which listings are 'high demand'.

## Data
Inside Airbnb, Nashville detailed listings (`listings.csv.gz`), snapshot dated 19 September 2026. 
Not included in this repo. 

## Models
- Linear regression (nightly price): 

__Cleaning Notebook__
In this notebook, the raw dataset was imported and initial cleaning and variable reduction was performed. To start, variables were removed based on business logic, as some columns in the raw dataset cannot be used to predict nightly pricing or would not be known at the time of making a prediction. After checking for columns with over 50% missingness, only the license variable applied. However, it was kept as a predictor because the presence of a license may indicate a more professionally managed listing, which could be associated with higher prices. It was converted to a binary `has_license` flag after confirming that none of the text entries were placeholders such as "Exempt" or "Pending". Following this, variables with low variability were checked. Longitude and latitude showed low variability, but measuring in degrees is misleading: one degree of latitude is approximately 69 miles, and one degree of longitude at Nashville's latitude is around 55.8 miles. It made more sense to keep them and engineer a new feature, distance to downtown, which will be done in the feature engineering notebook. Lastly, all listings with a minimum stay of 28 or more nights were removed (1,369 listings), since Airbnb treats stays of 28+ nights as monthly rentals, which are outside the project scope.

- Logistic regression (high-demand listings): _in progress_