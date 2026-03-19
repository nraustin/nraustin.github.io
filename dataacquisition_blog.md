# Analyzing Wages vs. Inflation in the Post-Pandemic Era

The economic landscape from 2022 to 2025 has been defined by dramatic shifts, from rapid inflation and subsequent tightening cycles to persistent labor shortages. In order to better understand the real-world impact of these processes, I've collected and organized data reports from the Bureau of Labor Statistics (BLS) Public Data API. The final result is a cleaned state and industry level dataset that can serve to provide insights into the relationship between wage growth and inflation.

## Motivating Questions

This analysis is designed to answer several economic questions:

- **Measuring Real Wage Growth:** Which states and industries achieved wage growth that outpaced inflation, creating real purchasing power gains, and where did workers fall behind?
- **Evaluating Labor Market Tightness:** How did the correlation between wage increases and unemployment rates vary across regions and sectors?
- **Assessing Resilience:** Which state/industry combinations demonstrated the most resilience during periods of economic shock, such as the 2023 financial sector stress or the 2025 government shutdown?

## Data Gathering

### Ethics and Allowable Use

This project utilizes the BLS Public Data API Version 2.0. Use of this data is strictly governed by the BLS Data Integrity and Allowable Use policies.

Key ethical considerations included:

- **Public Domain Data:** All data provided by the BLS is in the public domain, ensuring it is ethical and legal to gather and analyze.
- **Quotas:** I registered for an API key to use V2 of the API, which increases usage limits. This raised the daily query limit to 500 requests, or 25,000 time series data points.
- **Rate Limits:** The BLS API has a concurrency limit of one request at a time. I implemented `time.sleep(1)` between API calls to gentle with the API and prevent requests from being flagged or blocked.

### Data Retrieval Process

For researchers looking to build similar economic datasets, the challenge is not just scraping for data but mapping abstract economic concepts to the specific series IDs required by the BLS API. The BLS API does not allow for simple queries such as "state = Florida, measure = wages"; instead, you must construct an exact 20-character identifier.

Highly specific code parameters for values such as state, period, and area are necessary to build these query strings. After some difficulty finding where these could be retrieved/scraped, I found a publicly available file resource on the BLS website contained files mapping the codes to their respective values. After specifying an identity through User-Agents, the mappings were loaded in directly from the server.

`https://download.bls.gov/pub/time.series/`

This remaining retrieval process followed four repeatable steps.

#### 1. Defining the Taxonomy

The economic concepts (surveys), the geographies (areas), and the sectors (industries) of interest are defined here:

- **Surveys:** Current Employment Statistics (CES for wages) and Local Area Unemployment Statistics (LAUS for unemployment)
- **Geographies:** All 50 states, using Federal Information Processing System (FIPS) codes
- **Industries:** The 15 major economic supersectors, such as Manufacturing, Professional Services, and Leisure & Hospitality

#### 2. Building the Series ID Strings

Each BLS survey uses a strict, fixed-width formatting rule for its IDs.

For example, a CES state/industry wage series ID must follow this pattern:

`SM + U + State(2 digits) + Area(5 digits) + Industry(8 digits) + DataType(2 digits)`

Examples:

- **Total Private Wages in Texas** (`TX = 48`, `Ind = 05000000`): `SMU48000000500000011`
- **Texas Unemployment Rate:** `LAUST480000000000003`

Using a python script, I iterated through series ID codes contained in metadata lists of states and industries and programmatically generated thousands of these unique 20-character strings.

#### 3. Iterative API Queries (Chunking)

The API accepts a maximum of 50 series IDs per request. To comply with this limit, I batched the ID lists into groups of 50 and iteratively sent them via a POST request to the BLS endpoint:

`https://api.bls.gov/publicAPI/v2/timeseries/data/`

#### 4. Parsing and Standardizing

The API returns a nested JSON object. Parsing this response yields a structured list of dictionaries, where each entry contains:

- `series_id`
- `year`
- `period` (month/quarter)
- `value`

## Data Transformation and Tidying

The raw data returned by the API is in a long, stacked format, which is efficient for retrieval but difficult to analyze. To move from raw data to a single readable table, there were several transformation operations to make use of.

First, the data was first restructured so that each unique state, industry, and date combination occupied a single row. The metric type was moved from a row identifier to dedicated columns, such as wages, unemployment, and CPI.

Secondly, the BLS uses codes like `M01` for January and `S01` for semi-annual data. To standardize the timeline across all metrics, these entries were converted into datetime objects, such as `2025-01-01`.

After tidying the raw dataframes seperately, the pivoted table and metadata DataFrames were merged together, using state FIPS and industry codes to replace abstract numeric identifiers with friendly names. For example:

- `48` becomes `Texas`
- `20000000` becomes `Construction`

## Table Overview

This process generated a unified, tidy dataset covering the period from January 2022 to December 2025. This table provides a consistent view regional labor conditions.

### Final Sample Size and Features

- **Total Observations:** 18936 observations  
  One row per date, state, and supersector
- **Number of Features:** 8 columns

## Data Dictionary

| Feature Name | Type | Description / Representation |
|---|---|---|
| `date` | Datetime | First day of the observation month (`YYYY-MM-01`) |
| `state_name` | String | The U.S. state, such as Texas or California |
| `industry_name` | String | The economic supersector, such as Total Private or Manufacturing |
| `avg_hourly_earnings` | Float | Mean hourly earnings for production/non-supervisory employees (nominal USD) |
| `unemployment_rate` | Float | State-level unemployment rate (percentage) |
| `cpi_national` | Float | Consumer Price Index for All Urban Consumers (CPI-U), U.S. City Average |
| `wage_growth_yoy` | Float | Calculated year-over-year percentage change in `avg_hourly_earnings` |
| `real_wage_index` | Float | Calculated as `avg_hourly_earnings / cpi_national * 100` |

## Data Quality and Notable Limitations

There were several notable limitations encountered that should be considered during analysis.

### Missing Data

The extended federal government shutdown in 2025 severely impacted data collection. The BLS did not conduct its normal Household Survey for unemployment or Price Collection for CPI in October 2025. Official data for this month does not exist.

According to reccomendations from the BLS, BEA, and economists, linear interpolation was used to bridge more dynamic variables such as `cpi_national`, while forward willing was more appropriate for variables such as `unemployment_rate`.

### Metropolitan Bias

National and regional CPI data are robust, but local metropolitan-level CPI, such as data specific to Houston or Seattle, is often only reported semi-annually. This introduces an anti-aliasing requirement if attempting local real-wage analysis.

### Industry Sparsity

The resulting data only contains industries representing supersectors, and does not account for highly specific industries. While supersectors are reliable, niche industries such as "Software Publishers in Idaho," often have missing or suppressed data points at the state level due to small sample sizes. 

## Code and Further Information

The full Python code for this project, including metadata mapping, API query logic, and the final pivoting and cleaning script, is available in the GitHub repository below.

- **GitHub Repository:** `https://github.com/nraustin/data-acquisition-stat386`
- **BLS Public Data API V2 Documentation:** `https://www.bls.gov/developers/api_signature_v2.htm`
- **BLS Handbook of Methods:** The official handbook for U.S. labor statistics. It provides the exact definitions for the [Establishment Survey (Wages)](https://www.bls.gov/opub/hom/ces/home.htm) and the [Household Survey (Unemployment)](https://www.bls.gov/opub/hom/lau/home.htm).
- **FRED (Federal Reserve Economic Data):** A tool for [visualizing historical context](https://fred.stlouisfed.org/) and understanding the difference between "Real" (inflation-adjusted) and "Nominal" economic values.
- **Investopedia – Real Wage vs. Nominal Wage:** A highly accessible breakdown of how [purchasing power](https://www.investopedia.com/terms/r/realwage.asp) is calculated and why it matters for the average worker.