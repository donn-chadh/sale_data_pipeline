# sale_data_pipeline
## Overview
This project builds an ETL pipeline on a dirty cafe sales dataset. 
The goal is to practise the full extract, transform and load process using Python and pandas on real world messy data found on Kaggle.

## Tools and Libraries
- Python 3.11
- pandas
- numpy

## ETL Process
### Extract
The raw CSV is read into a pandas dataframe. A initial inspection is run using `shape`, `dtypes`, `isnull().sum()` and `value_counts()` across all columns. `value_counts()` is used alongside `isnull()` because `isnull()` cannot detect strings like ERROR and UNKNOWN

### Transform
**Item column**
Missing values and strings ERROR and UNKNOWN were replaced with null. Missing items were then filled by mapping from Price Per Unit where the price equated to only one item. 
Prices mapping to multiple items could not be filled confidently and were flagged for manual review.

**Price Per Unit column**
The same approach was applied in reverse — missing prices were filled by mapping from Item since every item has one unique price. 
Remaining nulls were flagged for manual review.

**Quantity column**
The column was converted to numeric using `pd.to_numeric` with `errors='coerce'` which handles string placeholders by converting them to null, this was something new I learned after many attempts to get the results I wanted.  
Missing values were filled by dividing Total Spent by Price Per Unit where both values existed, using a mask to limit the calculation to safe rows only.

**Total Spent column**
Missing values were filled by multiplying Quantity by Price Per Unit where both existed. 
Existing values were cross referenced against the calculation to identify incorrect entries — a non zero result indicating the original value did not match the expected calculation.

**Payment Method, Location and Transaction Date**
These columns could not be filled from other columns in the dataset, I could not see any patterns that may have eluded to a connection unlike the other columns. 
All missing values were flagged for manual review and tracing back to the source system using Transaction ID.

### Load
All flagged rows across every column are copied into a review dataframe for manual investigation. 
The original dataframe keeps all rows — nothing is dropped — so that manual fixes can be merged back. 
Check columns used during validation are dropped before the clean file is saved. 
Both the clean and review dataframes are written to CSV in `data/clean`.

## Changes in Approach
**Initial inspection understated missing values**
When inspecting the Item and Price Per Unit columns I noticed that ERROR and UNKNOWN were appearing as valid values rather than as missing. 
I realised that `isnull().sum()` only catches actual null values and that stringss wouldn't be detected this way. 
I added `value_counts(dropna=False)` to every column in the inspection to see the true count of values from the start.

**Flagged rows were kept in the original dataframe** 
I changed the approach so that all rows stay in the original DF and flagged rows are copied into a separate review dataframe, allowing them to be merged back into the DF at a future point

**isnull() was not sufficient for the Total Spent check**
When I ran the Total Spent check using `isnull()` it returned 0 which made no sense. 
I realised that all the null values had already been filled by the calculation so there was nothing left for `isnull()` to catch. 
I switched to subtracting Quantity x Price Per Unit from Total Spent instead, which returned 298 rows where the original value did not match the expected result.

## Limitations
**Payment Method and Location have the most missing values**
Payment Method has 2579 missing values and Location has 3265 missing values out of 10000 rows. 
These cannot be filled from other columns and require manual investigation against the source system. 

**Total Spent cross reference identified incorrect values**
The cross reference between Total Spent and Quantity x Price Per Unit returned 298 mismatched rows. 
These rows had values that did not add up correctly and were flagged for review.

## Still to Complete
- Define the ETL process around the code
- Investigate and resolve the 298 mismatched Total Spent rows
- Manually review and fill Payment Method, Location and Transaction Date where possible using Transaction ID against the source system
- Merge manually corrected review data back into the clean dataset
- Load final clean dataset to a PostgreSQL database using sqlalchemy
