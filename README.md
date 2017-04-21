# Data Enrichment Demonstration

A demonstration of data enrichment using the open Iowa Class 3 Liquor sales data.

# Documentation

I created a [PowerPoint presentation](docs/) and a [YouTube Video](https://youtu.be/PL_GBlcBNCk).

# Wikipedia

I gathered a [list of colleges in Iowa from Wikipedia](https://en.wikipedia.org/wiki/List_of_colleges_and_universities_in_Iowa). From this, I searched Facebook for college athletic department Facebook accounts.
Output: iowa_colleges_facebook.csv

# Python REST API Queries

I used the REST API from a number of different sources to gather data about the Iowa class 3 liquor license sales and related data. The code that gathers the data is [located in the source folder](/src/IowaDataLibrary.ipynb).

## Base Data
Source: https://data.iowa.gov/resource/spsw-4jax.json
<Need appToken>
Output: iowa_liquor_sales_by_day.csv

## Census Data
Input: 2010_us_census_places.txt
Source: http://api.census.gov/data/2015/pep/population?get=POP,GEONAME
<Need appTokenUSCB>
Output: iowa_town_census_2015.csv

## Holiday Data
No inputs
Output: holiday_proximity.csv

## Geocode Data
Source: https://maps.googleapis.com/maps/api/geocode/json
<Need appkey>
Output: iowa_city_geocodes.csv

## GDELT Data
Inputs: GDELT results from Google BigQuery saved as csv files in a gdelt sub-directory
- Limited results to 16000 due to the csv export limit
- Adjusted timestamps to get the data over the entire time period in a number of files

Query:

```
SELECT DATE(TIMESTAMP(string(SQLDATE))) as eventdate, ActionGeo_FullName, avg(AvgTone*NumArticles)/sum(NumArticles) as tone, ActionGeo_Lat, ActionGeo_Long
FROM [gdelt-bq:full.events]
WHERE ActionGeo_CountryCode='US' and 
REGEXP_MATCH(ActionGeo_FullName, '.*, Iowa,.*') and
TIMESTAMP(string(SQLDATE)) <= TIMESTAMP('20160531') and
TIMESTAMP(string(SQLDATE)) >= TIMESTAMP('20120101')
GROUP BY ActionGeo_FullName, eventdate, ActionGeo_Lat, ActionGeo_Long
ORDER BY eventdate DESC
LIMIT 16000;
```
Output: gdelt_iowa_data.csv

## Facebook Data
Input: iowa_colleges_facebook.csv
Utilizing `facepy` library
Need `application_access_token`
Output: college_fb_posts.csv

## Weather Data
Source: http://www.ncdc.noaa.gov/cdo-web/api/v2/data?datasetid=GHCNDMS
Need <appTokenNOAA>
Output: iowa_weather_data_by_county.csv

A [second python file](/src/IowaDataBrandQuery.ipynb) gathers data specific to the top selling brands of liquor.

## Brand Data
Source: https://data.iowa.gov/resource/spsw-4jax.json
Need <appToken>
Output: iowa_liquor_sales_by_brand.csv

Output: brand_data/*_iowa_liquor_sales_brand.csv

## Twitter Data
Input: brand_twitter_id.csv
Using twitter python library
Need ConsumerKey, ConsumerSecret, AccessToken, and TokenSecret from twitter.

Output: brand_data/*_tweets.csv


# Alteryx Workflow

I used two Alteryx workflows to clean and join the data.

## General Data Workflow

The [first workflow](/src/IowaDataWorkflow.yxmd) does most of the data cleaning. It is built from the following nodes.

### Nodes:

Input (a): iowa_liquor_sales_by_day.csv
Input (b): iowa_town_census_2015.csv

Join (c): Join (a) and (b) (left on city, right on city), keeping only the right "population" column

Formula (d): from (c) "J" output: Create a two new columns:
"LitersPerCapita" = sum_sale_liters/population
"salesPerCapita" = sum_sale_dollars/population

Filter (e): from (d) output: "LitersPerCapita" >= 0

Summarize (f): from (e) "True" output: GroupBy "city", GroupBy "month", Sum "LitersPerCapita", Sum "salesPerCapita", Sum "population"

Output (g): from (f): iowa_sales_per_capita_by_city.csv

Input (h): iowa_colleges_facebook.csv

Auto Field (i): from (h): select all fields

Summarize (j): from (i): GroupBy "City", Sum "Enrollment", Concat "Name"

Join (k): Join (e) and (j) (left on city, right on City), renaming (e).Sum_Enrollment as "Enrollment" and (j).Concat_Name as School(s)

Formula (l): from (k) right output: "Enrollment" is 0

Union (m): from (k) join and (l), Output all fields

Formula (n): from (m)
"studentpercapita" = Enrollment / (Enrollment + population)

Output (o): from (n) city_with_college_sales.csv

Summarize (p): from (n): GroupBy "city", Avg "studentpercapita", Avg "LitersPerCapita", Avg "population", First "School(s)"

Output (q): from (p) iowa_sales_with_college.csv

Input (r): holiday_proximity.csv

Auto Field (s): from (r) keep date, nearHoliday

Join (t): Join (e) and (s) (left on date, right on date) keep left (date, sum_sale_liters, sum_sale_dollars, year, population) keep right (nearHoliday)

Filter (u): from (t) join: population > 1000

Summarize (v): from (u): GroupBy "nearHoliday", Sum "sum_sale_liters", Sum "sum_sale_dollars"

Output (w): from (v): iowa_sales_holiday.csv

Input (x): gdelt_iowa_data.csv

Auto Field (y): from (x): keep all

Sample (z): from (y): Random 1 in N Chance for each Record (N=10)

Output (aa): from (z): gdelt_iowa_data_subset.csv

Join (ab): Join (e) and (z) (left on week, right on eventweek, left on city, right on city) keeping left (date, city, sum_sale_liters, sum_sale_dollars, week, population, LitersPerCapita, salesPerCapita) right (tone, geo)

Summarize (ac): from (ab) join: GroupBy "week", GroupBy "city", Sum "LitersPerCapita", Sum "salesPerCapita", Avg "tone", First "geo"

Output (ad): from (ac): gdelt_tone_sales.csv

Summarize (ae): from (c) join: GroupBy "month", GroupBy "county", Sum "sum_sale_liters", Sum "sum_sale_dollars", Sum "population"

Formula (af): from (ae):
"LitersPerCapita" = sum_sale_liters/population
"salesPerCapita" = sum_sale_dollars/population

Input (ag): iowa_weather_data_by_county.csv

Select (ah): from (ag) keeping all columns by *Unknown

Join (aj): Join (af) and (ah) (left on month, right on month, left on county, right on county) keeping all by right (county and month)

Output (ak): iowa_sales_monthly_weather_country.csv

Input (al): college_fb_posts.csv

Configuration (am): from (al): keep keyCount, kweek, city, 

Summarize (an): from (am): GroupBy "week", GroupBy "city", Sum "keyCount"

Summarize (ao): from (e): GroupBy "week", GroupBy "city", Sum "LitersPerCapita", Sum "salesPerCapita"

Join (ap): from (ao) and (an), (left on week, right on week, left on city, right on city) keep right "Sum_keyCount" and all left

Output (aq): from (ap) join: iowa_sales_college_facebook.csv

Input (ar): iowa_city_geocodes.csv

AutoField (as): all selected

Join (at): from (e) and (as) (left on city, right on city) output (left city, year, week, population, LitersPerCapita, salesPerCapita), (right lat, lon)

Summarize (au): from (at) join: GroupBy "year", GroupBy "city", GroupBy "lat", GroupBy "lon", Sum "LitersPerCapita", Sum "salesPerCapita", Avg "population"

Output (av): iowa_sales_by_geocode.csv


## Brand Workflow

The [second workflow](/src/IowaBrandWorkflow.yxmd) cleans and joins the brand data.

### Nodes:

Input (a): brand_data/*_iowa_liquor_sales_brand.csv

Parse (b): from (a): RegEx: ([^_]+) 

Select (c): from (b): all but FileName and *Unknown

Summarize (d): from (c): GroupBy "weekStarting", GroupBy "Brand", Sum "sum_sale_liters", Sum "sum_sale_dollars"

Output (e): from (d): iowa_sales_brand_week.csv

Input (f): brand_data/*_tweets.csv

Parse (g): from (f): RegEx: ([^_]+) 

Select (h): from (g): "tweet", "weekStarting", "Brand"

Summarize (i): from (h): GroupBy "weekStarting", GroupBy "Brand", Count "tweet"

Join (j): (d) and (i) (left on weekStarting right on weekStarting, left on Brand, right on Brand), keep all left and right "TweetCount"

Output (k): from (j) join, iowa_brand_tweets_by_week.csv


# Data Visualization

I used a [PowerBI](/src/IowaDataVisualization.pbix) file to visualize the datasets. It queries the following files from the data folder:

Queries:
- iowa_liquor_sales_by_day.csv (from Python)
- iowa_sales_per_capita_by_city.csv (from Alteryx)
- iowa_sales_with_college.csv (from Alteryx)
- city_with_college_sales.csv (from Alteryx)
- iowa_sales_holiday.csv (from Alteryx)
- iowa_sales_by_geocode.csv (from Alteryx)
- gdelt_iowa_data_subset.csv (from Alteryx)
- gdelt_tone_sales.csv (from Alteryx)
- iowa_sales_college_facebook.csv (from Alteryx)
- iowa_sales_monthly_weather_country.csv (from Alteryx)
- iowa_liquor_sales_by_brand.csv (from Python)
- iowa_sales_brand_week.csv (from Alteryx)
- iowa_brand_tweets_by_week.csv (from Alteryx)

The PowerBI dashboard has a number of reports that visualize the data in a variety of ways.