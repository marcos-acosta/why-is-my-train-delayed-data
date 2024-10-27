# Why is my train delayed?

This is the data processing repository behind [https://why-is-my-train-delayed.netlify.app/](https://why-is-my-train-delayed.netlify.app/), which was submitted as an entry to the [MTA's 2024 Open Data challenge](https://new.mta.info/article/mta-open-data-challenge).

## Directory structure

```
- notebooks
  - delay-events-data-preparation.ipynb: Complete, documented walkthrough of the data preparation process
- data
  - raw-data
    - *.csv: Datasets used during data processing
  - processed-data
    - nyct-delay-events.csv: The final result of the data preparation
    - nyct-delay-events.json: The same output data, but formatted as JSON records
```
