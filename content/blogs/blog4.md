---
categories:
- ""
- ""
date: "2020-18-10"
description:
draft: false
image: pic07.jpg
keywords: ""
slug: global
title: Evolution of GDP components over time
---

```{r load-libraries, setup, echo=FALSE}

library(tidyverse)  # Load ggplot2, dplyr, and all the other tidyverse packages
library(mosaic)
library(ggthemes)
library(GGally)
library(readxl)
library(here)
library(skimr)
library(janitor)
library(broom)
library(tidyquant)
library(infer)
library(openintro)
library(tidyquant)
library(scales)

```

# Background Info

We can simplify the main components of gross domestic product, GDP are personal consumption (C), business investment (I), government spending (G) and net exports (exports - imports). You can read more about GDP and the different approaches in calculating at the [Wikipedia GDP page](https://en.wikipedia.org/wiki/Gross_domestic_product).

The GDP data we will look at is from the [United Nations' National Accounts Main Aggregates Database](https://unstats.un.org/unsd/snaama/Downloads), which contains estimates of total GDP and its components for all countries from 1970 to today. We will look at how GDP and its components have changed over time, and compare different countries and how much each component contributes to that country's GDP. The file we will work with is [GDP and its breakdown at constant 2010 prices in US Dollars](http://unstats.un.org/unsd/amaapi/api/file/6).

# Data Cleaning


```{r read_GDP_data}

UN_GDP  <-  read_excel(here::here("data", "Download-GDPconstant-USD-countries.xls"), # Excel filename
                sheet="Download-GDPconstant-USD-countr", # Sheet name
                skip=1) # Number of rows to skip

```

 The first thing we will do tidy the data, as it is in wide format and you must make it into long, tidy format.

```{r reshape_GDP_data}

GDP_tidy<-UN_GDP %>% 
  
  pivot_longer(cols = 4:51, names_to = 'Year', values_to = 'Value') %>% 
  filter(IndicatorName %in% c('Gross capital formation',
                    'Exports of goods and services',
                    'Imports of goods and services',
                    'General government final consumption expenditure',
                    'Household consumption expenditure (including Non-profit institutions serving households)',
                    'Gross Domestic Product (GDP)')) %>%
  mutate(Value = Value/1e9) %>%
  
   # renaming our indicators
  mutate(IndicatorName = case_when(IndicatorName == 'Gross capital formation' ~ 'Gross capital formation',
                 IndicatorName == 'Gross Domestic Product (GDP)' ~ 'GDP',
                 IndicatorName == 'Imports of goods and services' ~ 'Imports',
                 IndicatorName == 'Exports of goods and services' ~ 'Exports',
                 IndicatorName == 'General government final consumption expenditure' ~ 'Government expenditure',
                 IndicatorName == 'Household consumption expenditure (including Non-profit institutions serving households)' ~ 'Household expenditure'))

glimpse(GDP_tidy)

```

# Producing the Plot


```{r gdp_1, out.width="100%"}

#Selecting our 3 countries of interest
country_list <- c("Germany","Spain", "France")

plot1 <- GDP_tidy %>% 
  
  # filtering for our preferred countries and by GDP
  filter(Country %in% country_list,
         IndicatorName != 'GDP') %>%
  
  # ordering the indicators to be in the same order as in the desired plot
  mutate(IndicatorName = factor(IndicatorName, levels = c('Gross capital formation',
                                      'Exports',
                                      'Government expenditure',
                                      'Household expenditure',
                                      'Imports')))  

plot1 %>%   
  ggplot() +
  geom_line(aes(x = Year, y = Value, group = IndicatorName, color = IndicatorName), size = 0.9) +
  scale_x_discrete(breaks = c(1970,1980,1990,2000,2010)) +
  scale_color_discrete(name = 'Components of GDP') +
  facet_wrap(~Country) +
  labs(x = '',
       y = 'Billion US$',
       title = 'GDP components over time',
       subtitle = 'In constant 2010 USD') +
  theme_bw()

```


What is the % difference between what we calculated as GDP and the GDP figure included in the dataframe?

```{r gdp_2, out.width="100%"}
GDP_components <- GDP_tidy %>% 

  pivot_wider(names_from = IndicatorName,
              values_from = Value) %>% 
  
  mutate(`Net Exports` = Exports - Imports,
         GDP_calculated = `Household expenditure` + 
           `Gross capital formation` +
           `Government expenditure` +
           `Net Exports`,
         GDP_diff_percentage = GDP_calculated/GDP - 1)

cat("Mean difference between calculated and given GDPs:\n",
    mean(GDP_components$GDP_diff_percentage, na.rm = TRUE))
```

Perhaps this mean difference arises from reporting standards and currency rates, resulting in different calculations for each country.

We will now break down GDP as the sum of Household Expenditure (Consumption *C*), Gross Capital Formation (business investment *I*), Government Expenditure (G) and Net Exports (exports - imports).



```{r gdp_3, out.width="100%"}
knitr::include_graphics(here::here("images", "gdp2.png"), error = FALSE)

GDP_breakdown %>% 
  
  #Filtering according to our pre-specified country list
  filter(Country %in% country_list) %>% 
  
  #Filtering/ Selecting relevant indicators
  select(`Country`,
         `Year`,
         `Government expenditure`,
         `Gross capital formation`,
         `Household expenditure`,
         `Net Exports`,
         `GDP_calculated`) %>%
  
  #Calculating the proportions of GDP of indicators
  mutate(`Government expenditure` = `Government expenditure` / `GDP_calculated`,
         `Gross capital formation` = `Gross capital formation`/ `GDP_calculated`,
         `Household expenditure` = `Household expenditure`/ `GDP_calculated`,
         `Net Exports` = `Net Exports` / `GDP_calculated`) %>% 
  
  #Removing GDP from selected indicators in plot
  select(-`GDP_calculated`) %>% 

  pivot_longer(cols = 3:6, names_to = 'IndicatorName', values_to = 'Value') %>% 
  
  #Specifying order of indicators
  mutate(IndicatorName = factor(IndicatorName, 
                                levels = c('Government expenditure',
                                           'Gross capital formation',
                                           'Household expenditure',
                                           'Net Exports'))) %>% 
  ggplot() +
  geom_line(aes(x = Year, y = Value, group = IndicatorName, color = IndicatorName), size = 0.9)+
  scale_x_discrete(breaks = c(1970,1980,1990,2000,2010)) +
  scale_y_continuous(labels=label_percent()) +
  scale_color_discrete(name = 'GDP breakdown') +
  facet_wrap(~ Country) +
  labs(title = 'GDP and its breakdown at constant 2010 prices in US Dollars',
       y = 'Proportion',
       x = '',
       caption = 'Source: United Nations') +
  theme_bw()

```

# Some Findings

## Germany's trade surplus

Germany is the only country in the list (and one of few globally) that exhibits a clear, consistent trade surplus, primarily due to strong exports of vehicles and other machinery. This surplus has been labelled as 'toxic' by many, who regard it as a liability rather than an asset - even Donald Trump!

## India's occupation demographics

From the graph it is also clear to see that India's GDP has historically been dependent on household expenditure, but this reliance has been declining in recent years, at the expense of investment. Even before COVID-19 struck, India’s household consumer demand has been vulnerable and unpredictable because of skewed occupation demographics and its clear wealth gap - India’s richest 20% of households account for 36% of consumption expenditure. On the flipside, investment in the country has risen rapidly due to favourable infrastructural factors for investors, such as energy, communication and health, and government-led FDI incentives.










