---
author: Dominik Freunberger
date: "2021-05-13"
description: Actual Test
tags:
- test
title: Actual Test
---

Google Transit Data Across the Stockholm region. Or: Show me where you
live and I tell you if you work from home.
================
Dominik Freunberger (<dominik.freunberger@gmail.com>)
3/9/2021

A [friend](https://www.andifugard.info/) recently published [a very nice
analysis](https://inductivestep.github.io/Google-transit-London/) of how
Londoners changed their mobility behavior during the COVID-19 pandemic
as compared to before. I wondered how this looks for the Stockholm
region in Sweden.

``` r
library(tidyverse)
library(magrittr)
library(lubridate)
library(readxl)
library(ggrepel)
library(ggpubr)
```

The data is from [Google](https://www.google.com/covid19/mobility/), so,
to quote [Andy](https://inductivestep.github.io/Google-transit-London/),
“data quality unclear…”

``` r
if(!file.exists("movement_data")) {
        dir.create("movement_data")
}
# dataURL = "https://www.gstatic.com/covid19/mobility/Region_Mobility_Report_CSVs.zip"
# download.file(dataURL, destfile="./movement_data/movement.zip", method = "curl")
# unzip ("./movement_data/movement.zip", exdir = "./movement_data")
```

Let’s read the Swedish (SE) data, get the data for the Stockholm region,
and tidy it up a bit:

``` r
data = read.csv("./movement_data/2020_SE_Region_Mobility_Report.csv")

data %<>% mutate(
        day = parse_date(date, "%Y-%m-%d"),
        day_of_week = wday(day, label = TRUE),
        weekday = ifelse(day_of_week %in% c("Sat", "Sun"), "Weekend", "Weekday")
        )

sthlm_data <- data %>%
        filter(sub_region_1 == "Stockholm County" & sub_region_2 != "") 

sthlm_data %>%
        group_by(sub_region_2) %>%
        tally()
```

    ## # A tibble: 26 x 2
    ##    sub_region_2               n
    ##  * <chr>                  <int>
    ##  1 Botkyrka Municipality    383
    ##  2 Danderyd Municipality    360
    ##  3 Ekerö Municipality       358
    ##  4 Haninge Municipality     379
    ##  5 Huddinge Municipality    385
    ##  6 Järfälla Municipality    382
    ##  7 Lidingö kommun           360
    ##  8 Nacka Municipality       379
    ##  9 Norrtälje Municipality   379
    ## 10 Nykvarn Municipality     250
    ## # … with 16 more rows

``` r
sthlm_data %>%
        select(ends_with("percent_change_from_baseline")) %>%
        names()
```

    ## [1] "retail_and_recreation_percent_change_from_baseline"
    ## [2] "grocery_and_pharmacy_percent_change_from_baseline" 
    ## [3] "parks_percent_change_from_baseline"                
    ## [4] "transit_stations_percent_change_from_baseline"     
    ## [5] "workplaces_percent_change_from_baseline"           
    ## [6] "residential_percent_change_from_baseline"

``` r
tidy_sthlm <- sthlm_data %>%
        pivot_longer(
          cols = ends_with("percent_change_from_baseline"),
          names_to = "location",
          names_pattern = "(.+)_percent_change_from_baseline",
          values_to = "percent_change_from_baseline")
        
tidy_sthlm$sub_region_2 = word(tidy_sthlm$sub_region_2, 1)

tidy_sthlm %>%
        select(location, percent_change_from_baseline) %>%
        filter(!is.na(percent_change_from_baseline)) %>%
        group_by(location) %>%
        summarise(mean_change = mean(percent_change_from_baseline, na.rm = T),
              count = sum(!is.na(percent_change_from_baseline)),
              NAs = sum(is.na(percent_change_from_baseline)))
```

    ## # A tibble: 6 x 4
    ##   location              mean_change count   NAs
    ## * <chr>                       <dbl> <int> <int>
    ## 1 grocery_and_pharmacy        -2.63  6061     0
    ## 2 parks                        6.70   731     0
    ## 3 residential                 10.8   6182     0
    ## 4 retail_and_recreation       -8.61  6415     0
    ## 5 transit_stations           -24.5   8914     0
    ## 6 workplaces                 -31.2   8725     0

As you can see, there’s more data included but here we’ll focus on the
data about mobility trends for places of work. Let’s grab that data and
plot it:

``` r
tidy_work = tidy_sthlm %>%
  filter(location %in% c("workplaces"))

plot1 <- tidy_work %>%
        mutate(location = gsub("_", " ", location)) %>%
        ggplot(aes(x = day, 
                   y = percent_change_from_baseline,
                   color = sub_region_2)) +
        theme(legend.position = "none") +
        labs(x = "Date", y = "% Change from Baseline") +
        theme_minimal()
```

``` r
plot1 +
  geom_smooth(se = F, size = 1.2) +
        scale_color_discrete(name = "Municipality")
```

![](Sthlm_COVID_files/figure-gfm/unnamed-chunk-5-1.png)<!-- -->

Which areas have more people who still have to travel to their
workplace?

``` r
tidy_sthlm %>%
        filter(location %in% c("workplaces")) %>%
        filter(day >= as.Date("2021-01-01") &
                       day < as.Date("2021-03-01")) %>%
        group_by(sub_region_2) %>%
        summarise(mean_change = mean(percent_change_from_baseline)) %>%
        na.omit() %>%
        mutate(sub_region_2 = fct_reorder(sub_region_2, mean_change)) %>%
        ggplot(aes(y = sub_region_2, x = mean_change)) +
        geom_point(size = 3, col = "#e02c24") +
        labs(y = "Municipality", 
             x = "% Change from baseline", 
             title = "Visits to workplaces",
             subtitle = "Mean change from pre-Covid baseline, Jan to Feb 2021") +
        theme_minimal()
```

![](Sthlm_COVID_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->

## **Interesting.**

You don’t have to be living in Stockholm for a long time to see that
this order is not random. Rather, it seems to have to do with how posh,
or, how wealthy a municipality is. Let’s get the Swedish income data
from
[SCB](https://www.scb.se/hitta-statistik/statistik-efter-amne/hushallens-ekonomi/inkomster-och-inkomstfordelning/inkomster-och-skatter/pong/tabell-och-diagram/inkomster--individer-lankommun/sammanraknad-forvarvsinkomst-2019--per-kommun-efter-percentiler/).

Read in the data and grab the data for the Stockholm area:

``` r
income = read_excel("./income_data/income.xls", skip = 3)
income = income[,c(1,2,4)]
income = income %>%
        rename(municipality = ...1, 
               mean_income = `Kvinnor och män`,
               median_income = ...4)

sthlm_income = income %>%
        filter(municipality %in% tidy_sthlm$sub_region_2)

sthlm_work = tidy_sthlm %>%
        filter(location %in% c("workplaces")) %>%
        filter(day >= as.Date("2021-01-01") & day < as.Date("2021-03-01")) %>%
        group_by(sub_region_2) %>%
        summarise(mean_change = mean(percent_change_from_baseline)) %>%
        na.omit()

sthlm_work$sub_region_2 = word(sthlm_work$sub_region_2, 1)

sthlm_work = sthlm_work %>%
        rename(municipality = sub_region_2) %>%
        inner_join(sthlm_income, by = "municipality")

sthlm_work$mean_income = round(as.numeric(sthlm_work$mean_income), 2)
sthlm_work$median_income = round(as.numeric(sthlm_work$median_income), 2)
```

``` r
ggplot(sthlm_work, aes(x = mean_change, y = median_income)) +
    geom_point(size= 3) +
    stat_smooth(method = "lm",
        col = "#e02c24",
        se = T,
        size = 2) +
        labs(title = "Workplace Mobility: A Matter of Wealth",
             x = "% Mobility Change from Baseline",
             y = "Median Income of Municipality in SEK (2019)") +
                ggrepel::geom_text_repel(aes(label = municipality), size = 3) +
        theme_minimal() +
        stat_cor(method="pearson", label.x = -28, label.y = 370000, size = 5)
```

![](Sthlm_COVID_files/figure-gfm/unnamed-chunk-9-1.png)<!-- -->

So, one could say people in these areas care less about restriction in
general and travel more anyway. So how does mobility in other domains
look in relation to the municipalities’ wealth?

``` r
sthlm_recr = tidy_sthlm %>%
        filter(location %in% c("retail_and_recreation")) %>%
        filter(day >= as.Date("2021-01-01") & day < as.Date("2021-03-01")) %>%
        group_by(sub_region_2) %>%
        summarise(mean_change = mean(percent_change_from_baseline)) %>%
        na.omit()

sthlm_recr$sub_region_2 = word(sthlm_recr$sub_region_2, 1)

sthlm_recr = sthlm_recr %>%
        rename(municipality = sub_region_2) %>%
        inner_join(sthlm_income, by = "municipality")

sthlm_recr$mean_income = round(as.numeric(sthlm_recr$mean_income), 2)
sthlm_recr$median_income = round(as.numeric(sthlm_recr$median_income), 2)
```

``` r
ggplot(sthlm_recr, aes(x = mean_change, y = median_income)) +
    geom_point(size= 3) +
    stat_smooth(method = "lm",
        col = "#e02c24",
        se = T,
        size = 2) +
        labs(title = "Retail and recreation mobility as a function of income",
             x = "% Mobility Change from Baseline",
             y = "Median Income of Municipality in SEK (2019)") +
                ggrepel::geom_text_repel(aes(label = municipality), size = 3) +
        theme_minimal() +
        stat_cor(method="pearson", label.x = -10, label.y = 390000, size = 5)
```

![](Sthlm_COVID_files/figure-gfm/unnamed-chunk-11-1.png)<!-- -->

``` r
sthlm_groc = tidy_sthlm %>%
        filter(location %in% c("grocery_and_pharmacy")) %>%
        filter(day >= as.Date("2021-01-01") & day < as.Date("2021-03-01")) %>%
        group_by(sub_region_2) %>%
        summarise(mean_change = mean(percent_change_from_baseline)) %>%
        na.omit()

sthlm_groc$sub_region_2 = word(sthlm_groc$sub_region_2, 1)

sthlm_groc = sthlm_groc %>%
        rename(municipality = sub_region_2) %>%
        inner_join(sthlm_income, by = "municipality")

sthlm_groc$mean_income = round(as.numeric(sthlm_groc$mean_income), 2)
sthlm_groc$median_income = round(as.numeric(sthlm_groc$median_income), 2)
```

``` r
ggplot(sthlm_groc, aes(x = mean_change, y = median_income)) +
    geom_point(size= 3) +
    stat_smooth(method = "lm",
        col = "#e02c24",
        se = T,
        size = 2) +
        labs(title = "Grocery and pharmacy mobility as a function of income",
             x = "% Mobility Change from Baseline",
             y = "Median Income of Municipality in SEK (2019)") +
                ggrepel::geom_text_repel(aes(label = municipality), size = 3) +
        theme_minimal() +
        stat_cor(method="pearson", label.x = -10, label.y = 390000, size = 5)
```

![](Sthlm_COVID_files/figure-gfm/unnamed-chunk-13-1.png)<!-- -->
