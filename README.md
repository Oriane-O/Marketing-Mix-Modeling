# Implementation of Marketing Mix Modeling

This project is based on the pymc implementation of the MMM presented in the paper [ Jin, Yuxue, et al. “Bayesian methods for media mix modeling with carryover and shape effects.” (2017)](https://research.google/pubs/pub46001/).
Here we work on simulated data to set the parameters ourselves and allow us to conduct a parameter recovery exercise.The data generation process is as an adaptation of the blog posts
[“Media Effect Estimation with PyMC: Adstock, Saturation & Diminishing Returns” ](https://juanitorduz.github.io/pymc_mmm/.) and [Mastering Marketing Mix Modelling In Python](https://medium.com/data-science/mastering-marketing-mix-modelling-in-python-7bbfe31360f9).
It also uses as references this other sources:

🔎 https://github.com/sibylhe/mmm_stan/tree/main  
🔎 https://www.pymc-marketing.io/en/stable/notebooks/mmm/mmm_example.html

## 1. Introduction

Marketing mix models or Media mix models are used to understand how media spending affects sales, so as to optimize future budget allocation. These models are usually based on weekly or monthly aggregated national or geo level data. The data may include sales, price, product distribution, media spend in different channels, and external factors such as macroeconomic forces, weather, seasonality, and market competition.
ROAS (return on ad spend) and mROAS (marginal ROAS) are the key metrics to look at. High ROAS indicates the channel is efficient, high mROAS means increasing spend in the channel will yield a high return based on current spending level.
The ultimate goal is to create the best funnel to have the best ROI (return on investment).

### 1.1 A completer

---

## 2.Business Case Study

Let's first define the business problem we are trying to solve. We want to optimize the marketing budget allocation of our client with the following characteristics:

- Sales data: weekly sales of the client.
- Media spend data: weekly spend on 5 different media channels
- Domain knowledge:
  - We know that there has a been an positive sales trend which we believe comes from a strong economic growth.
  - We also know that there is a yearly seasonality effect.

#IMAGE
There is a causal relationship between marketing and sales, but what is the nature of that relationship? We have to take into account that there is :

- a carry-over effect (adstock). Meaning, the effect of spend on sales is not instantaneous but accumulates over time.
- a saturation effect. Meaning, the effect of spend on sales is not linear but saturates at some point

### 2.1 Data Generation

> 📄 Find all the Generation process in the Notebook `1-Data_generation.ipynb`  
> 📄 Find a simplified Data_generation function in the Script `data_generator_function.py`

As described in the section above, we want a dataset with:

- Sales variables:
  Sales ( the target variable)

- Media Variables:

  - ooh (Out of home spend)
  - tv (Television spend)
  - print (Print media spend)
  - facebook ( Facebook ads spend)
  - search (Google search ads spend)
  - facebook_I (facebook impressions)
  - search_clicks_P (Google search ads performence,number of clicks)

- Control Variables:
  competitor_sales_B (competitor sales baseline)

To construct our dataset we considered 4 years of weekly data.

From what we know from the domain knowledge, we have described the demand with an increasing trend for organic growth, with a seasonality (oscillation) in the demand each year.

![alt text](images/image.png)

We also created a proxy for demand as in reality, the true demand is never observable, but we can find proxies.

![alt text](images/image-1.png)

After that, we created synthetic data for each marketing channel.The different channel spends are correlated with demand, and also are designed by different marketing strategy (for example high budget, bursty campaigns for TV, or relatively consistent with moderate noise for out-of-home).

![alt text](images/image-2.png)

Next, we pass the raw signal through the two transformations: first the geometric adstock (carryover effect) and then the logistic saturation.
For the adstock, we set a maximum lag effect of 8 weeks, and we chose our alpha parameter accordingly to the media:
| Channel | Type |alpha | Justification |
| ---------- | --------------------- | ------------------- | ------------------------------------------------------------------- |
| `tv` | Offline, mass media | **0.5 – 0.8** | Strong long-term effect (brand awareness, memorability) |
| `ooh` | Offline, visual | **0.4 – 0.7** | Moderately lasting impact, repeated exposure in public spaces |
| `print` | Offline, print media | **0.2 – 0.5** | Lower memorability, short-lived effect, rarely drives direct action |
| `facebook` | Digital, paid social | **0.1 – 0.4** | Short-term performance focus, quick decay of impact |
| `search` | Digital, intent-based | **0.0 – 0.2** | No carryover effect: impact is immediate (direct response channel) |

Same for the saturation:
| Channel | Type | λ | Justification |
| ---------- | --------------------- | -------------------------- | ---------------------------------------------------------------------------------- |
| `tv` | Offline, mass media | **0.5 – 1.5** | Strong saturation: TV reach saturates quickly (broad audience) |
| `ooh` | Offline, visual | **1.0 – 2.0** | Moderate saturation, especially in high-exposure urban areas |
| `print` | Offline, print media | **1.5 – 3.0** | Low saturation (narrow audience); hard to reach saturation point |
| `facebook` | Digital, paid social | **0.5 – 1.5** | Can saturate fast with high budget, algorithmically optimized |
| `search` | Digital, intent-based | **2.0 – 4.0** | Very low saturation: conversion effectiveness remains linear longer (pull channel) |

![alt text](images/image-3.png)

And we can visualize the effect signal for each channel after each transformation:

![alt text](images/image-4.png)

We then add the Facebook impressions and google search clicks, and then the control variable that is the competitor sales baseline ( that also follows trend and seasonality)

Finally, we create our target value, the sales, that we assume it is a linear combination of the effect signal, the trend and the seasonal components, plus the two events and an intercept. We also add some Gaussian noise.

![alt text](images/image-5.png)

### 2.2 Exploratory Data Analysis
