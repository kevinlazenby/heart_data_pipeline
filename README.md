# Healthcare Ethics & Allocation Lab Heart Data Pipeline

Please reach out to Kevin Lazenby at kevinlazenby@uchicago.edu with any questions.

Helpful references for this script are listed below:
- [SRTR data dictionary](https://www.srtr.org/requesting-srtr-data/saf-data-dictionary/)
- [OPTN policies](https://optn.transplant.hrsa.gov/media/eavh5bf3/optn_policies.pdf)

## Code chunk #1: Import data sets from SRTR
To import data, save the SAF files in `.sas7bdat` format to a folder in your working directory. For example, to use the code as currently written, save the SAF files in a folder named `pubsaf2206`, which should be a subfolder within the working directory for the R project. The variable `curr_data_release` stores the name of this folder so that the script loads in the data from the correct folder. The relevant code snippet is below.

```
# change this variable to load in a different release of the SAF
curr_data_release <- "pubsaf2206"
```

## Code chunk #2: Link data sets using JustId, WlregAuditId, and PX_ID
Not all files in the raw SAF share the same linking variables, so this chunk performs several joins to make `PX_ID` the common linking variable for the imported data sets. This chunk also performs several data cleaning and wrangling tasks, which are listed below:
- Converting some instances of the `Change_Dt` variable from a date-time to a simple date
- Removing entries from the risk stratification data that lack a valid `JustId`
- Filtering justification forms to only keep a single justification form per candidate per day. When multiple justification forms are submitted for a single candidate within a single day, the form that was submitted last is kept.
- Removing certain variables that were deemed irrelevant, as in the example below:
```
  select(-(ends_with("St"))) %>% 
  select(-(starts_with("CandHist"))) %>%
  select(-(starts_with("SenData"))) %>%
  select(-(starts_with("CurrTher"))) %>%
  select(-(ends_with("Type"))) %>%
  select(-(ends_with("Perf"))) %>%
```

## Code chunk #3: Tidy input datasets
This chunk applies the inclusion and exclusion criteria to the data sets. The basic criteria are as follows:
- Candidates listed prior to January 1, 2010, were excluded.
- Candidates with age less than 18 at the time of listing were excluded.
- Recipients of lung-only transplants were excluded.

The code snippet below can be edited to customize the behavior of code chunk #3.
```
# change these variables to customize inclusion criteria
filter_date <- mdy("01/01/2010")
include_multi_organ_recipients <- TRUE
missingness_threshold <- 1
```

Editing `filter_date` will change the first inclusion/exclusion criterion above. If `include_multi_organ_recipients` is set to `FALSE`, recipients of multi-organ transplants will be excluded. `missingness_threshold` refers to the maximum percentage of missing values in a given variable in `cand_thor` or `tx_hr`. In the current iteration of this script, `missingness_threshold <- 1`, which means that variables with 100% missing values are removed from these data sets.

## Code chunk #4: Pivot cand_list and stathist by date, then full_join to create full_list
This code chunk begins the primary purpose of this script, which is to assemble a single, long-form dataset arranged by unique dates within a candidate/recipient's record. That is, each row in the final data set represents a unique date in a given candidate/recipient's record on which a significant event occurred. To create a long-form dataset, this script relies on the data transformation called *pivoting*. More information on the tidyr functions `pivot_longer` and `pivot_wider` can be found [here](https://tidyr.tidyverse.org/articles/pivot.html). In this code chunk, the data sets `cand_thor` and `stathist_thor` are pivoted to long form using all columns with date variables (the selection of these columns is operationalized as `cols = ends_with("DT")`). Once the data are in long form, the variables `unique_date` and `unique_event` are created. `unique_date` stores each unique date in the candidate/recipient's record. `unique_event` stores the variable name of the event that occurred on that date. For example, when a candidate is listed for transplant, the listing date is stored in `CAN_LISTING_DT`. After this code chunk, the candidate's listing date is stored in `unique_date`, and the string `CAN_LISTING_DT` is stored in `unique_event`. After creating `unique_date` and `unique_event`, the data sets are pivoted back to wide form, then joined together using the common variables `PX_ID`, `unique_date`, and `unique_event` to create the data set `full_list`. The code chunks following this one will join additional data sets to `full_list`.

## Code chunk #5: Pivot tx_list by date, then join with full_list
This code chunk performs the same sequence of operations as in code chunk #4, except with the data set `tx_hr`.

## Code chunk #6: Join just_form and just_stat1-just_stat4 with risk_strat, pivot by date, then join with full_list
This code chunk performs the same sequence of operations as in code chunk #4, except with the data sets storing justification forms from the new policy era and risk stratification data from the new policy era. Additionally, the following data wrangling steps are also performed:
1. Using the dplyr function `coalesce`, several variables that contain the same type of measurement are coalesced into a new, single variable. For example, across the various SAF data sets, systolic blood pressure is stored in several variables, including `HemoSbp`, `EcmoWithoutHemoSbp`, `McsdWithoutHemoSbp`, and several others. After coalescing these variables, systolic blood pressure measurements are stored in a single variable called `systolicBP`.
2. In the risk stratification data, there are instances in which a single measurement takes on two values for a single candidate on the same date. For example, there are instances in which a single candidate has two different values for mean arterial pressure on a single date. Many or most of these instances were from combinations of initial status justification forms and status extension request forms. Given that OPTN policies specify that candidate data must be measured at a defined time after the initial status listing for a status extension, these instances were assumed to be data entry errors. To preserve as much candidate data as possible, for each instance wherein a candidate had multiple different values for a single variable on a single date, the measurement date from the form submitted first was left unchanged. The dates from forms submitted later were changed to the `Change_Dt` of that form. The variable `date_changed` indicates the rows that contain a changed date. For these rows, `date_changed` contains the variable name corresponding to the changed date. For example, if the original data had two values for mean arterial pressure for a single candidate on the same day, `date_changed` will contain "mean_arterial_pressure".
3. The variable `status_criteria` combines information from many indicator variables and contains the criteria for each candidate to be listed at their assigned status.

## Code chunk #7: Pivot old policy justification forms by date, then join with full_list
This code chunk performs the same sequence of operations as in code chunk #4, except with the data sets storing justification forms from the old policy era. The same three data wrangling steps from code chunk #6 are also applied to the old policy justification forms in this code chunk. Additionally, 

## Code chunk #8: Method to filter full_list to exclude unwanted timepoints

## Code chunk #9: Calculating number of days between each observation in full_list and final_list, by PX_ID
