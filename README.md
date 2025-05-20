# Why is my train delayed?

When browsing through the [MTA's open datasets](https://data.ny.gov/browse?Dataset-Information_Agency=Metropolitan+Transportation+Authority&sortBy=relevance&page=1&pageSize=20), I expected to find a dataset with information on what causes train delays, but despite scrolling through all 196 datasets twice, I didn't find one! What I did find, though, is a [dataset of service alerts](https://data.ny.gov/Transportation/MTA-Service-Alerts-Beginning-April-2020/7kct-peq7/about_data) sent by the MTA to notify passengers about delays or schedule changes. The goal of this project was to build a dataset that allows more direct analysis on the _cause_ of alerts on different lines, times of days, et cetera.

Read on to see what the data contains, and how it was processed!

_(This project was submitted as an entry to the [MTA's 2024 Open Data challenge](https://new.mta.info/article/mta-open-data-challenge). Note that the [data analysis frontend](https://why-is-my-train-delayed.netlify.app) was created rather quickly in order to submit by the deadline, since most of the effort was actually put into the data analysis.)_

## Final dataset schema

- `event_id`: The event id (from the [alerts dataset](https://data.ny.gov/Transportation/MTA-Service-Alerts-Beginning-April-2020/7kct-peq7/about_data)) associated with this delay event
- `first_alert_datetime`: The datetime (`YYYY-MM-DD HH:MM:SS`, UTC) of the first alert pertaining to this delay event
- `last_alert_datetime`: The datetime (`YYYY-MM-DD HH:MM:SS`, UTC) of the last alert pertaining to this delay event
- `num_alerts`: The number of alerts sent pertaining to this delay event
- `affected_services`: A pipe-delimited list of services (e.g. `A`, `R`) affected by this event
- `alert_descriptions`: A pipe-delimited list of descriptions of all alerts sent about this event
- `delay_issue`: A fine-grained reason for the delay, such as `"Brakes activated"` or `"Disruptive passenger"`
- `delay_issue_category`: A coarse-grained category for the delay issue, such as `"Mechanical issue"` or `"Passenger issue"`
- `station_*`: Columns related to the station at which the delay was reported, if any. These columns are defined in the [MTA subway stations dataset](https://data.ny.gov/Transportation/MTA-Subway-Stations/39hk-dx4f/about_data)
- `affected_services_basic`: The same as `affected_services`, except certain internal line names are replaced with their common ones (e.g. using `S` instead of `H` for the Far Rockaway Shuttle)
- `year_month`: For convenience, contains just the year and month in `YYYY-MM` format
- `day_of_week`: For convenience, contains the day of the week (`0` is Monday)

## Directory structure

```
- data-analysis/notebooks
  - delay-events-data-preparation.ipynb: Complete, documented walkthrough of the data preparation process
- data
  - raw-data
    - *.csv: Datasets used during data processing
  - processed-data
    - nyct-delay-events.csv: The final result of the data preparation
    - nyct-delay-events.json: The same output data, but formatted as JSON records
- requirements.txt: The Python requirements, which can be installed with `pip install -r requirements.txt`
```

## Data processing summary

After basic pre-processing of the data, I combined the alert title and description into a single string to make searching easier.

### Identifying the causes

The first issue I ran into is the question of, "How do I choose which categories to put each alert into?" Essentially, how should we [code](<https://en.wikipedia.org/wiki/Coding_(social_sciences)>) the data?

The approach I settled on was an iterative one: I would look through a few randomly sampled alerts and identify common themes. Then, I would define a regex pattern to match the title/description of those alerts, such as `/disruptive|assault|altercation/` or `/(broken rail|replac[^\.]+rail|rail repair)/`. Then, I would randomly sample more delays that were _not_ matched and repeat the process. I repeated this process until I felt that the remaining delays did not represent any larger theme.

Once I had a list of fine-grained delay issues, I defined some coarse-grained categories to group similar issues together, like `HUMAN_DISRUPTION` and `EMS_NYPD_FDNY_RESPONSE`.

### A note on causation

A slightly philosophical issue that I encountered while constructing this dataset is that it's not trivial to answer the question of _why_ a train is delayed, even with all the information given. For example, an alert might be tagged as `EMS_NYPD_FDNY_RESPONSE` because the alert mentions FDNY, but in that case, the root cause was probably a fire. But what caused the fire? If it was caused by an electrical malfunction, should it be classified as a `MECHANICAL_ISSUE` instead?

Here, the best I could do was be careful about the order in which I searched for patterns in the title/description, i.e. searching for issues that are more likely to be root-cause (e.g. fire, disruptive passengers) before issues that are more likely to be reactionary (e.g. removing a train from service, EMS/NYPD/FDNY).

### Associating delays with stations

In order to identify the station associated with each delay, I first created a regex pattern that matches every station listed in the [MTA subway stations dataset](https://data.ny.gov/Transportation/MTA-Subway-Stations/39hk-dx4f/about_data) and used it to tag each delay alert. However, this still isn't enough, since there can be multiple stations with the same name (e.g. there are five stations called 23 St). To deal with this, we _also_ take a look at the `affected_services` column to see which lines are affected by the delay, and use that to disambiguate which station it must be. There can still be other tricky edge cases (see the notebook for more detail, but this is mostly sufficient).

### Final steps

Finally, we group alerts by Event ID and aggregate certain columns (such as `alert_descriptions` and `num_alerts`) and join the data with the subway stations dataset in order to enrich the dataset with other information, such as whether the station is elevated or underground. Additionally, we perform a convenience step of providing an additional column where internal service designations (such as `FX` (`F` express) and `FS` (Franklin Shuttle)) are replaced with their more common designations.
