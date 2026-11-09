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

### Logistic regression (high demand): 