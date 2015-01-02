---
title: 'Udacity - Intro to Data Science: Project 1'
author: "Fernando Hernandez"
date: "Saturday, December 20, 2014"
output:
  html_document:
    highlight: espresso
    theme: journal
  pdf_document: default
geometry: margin=1in
fontsize: 10pt
---

## Introduction

The Metro Transit Authority (MTA) collects large amounts of data on subway ridership to help aide in decisions for staffing, maintenance, and observation. Accurately predicting trends and changes can save money, and improve safety and efficiency.

The goal of this project is to predict subway ridership in New York City given many different possible factors that can contribute to ridership, such as weather conditions, times of the day/month, and locations.

This project write-up will use some basics of Data Science in the R programming language to answer this question, including:

* data wrangling
* applied statistics and machine learning
* data visualization
* exploratory data analysis
* working with large datasets
* model training and testing
* model performance measurement


### Libraries Used

```{r, message=FALSE}

library(dplyr)
library(ggplot2)
library(scales)
library(gridExtra)
library(corrplot) 
library(stargazer)
library(stringi)
library(caret)

```

### Loading/formatting data

Load data and add some features/variables that were explored.

```{r}
# setwd("P1/data")

weather <- tbl_df(read.csv("turnstile_weather_v2.csv"))

# Add POSIXCT datetime
weather$datetime <- as.POSIXct(weather$datetime)

# Add full name day of the week; Order starting with Monday; the beginning of the work week.
weather$day_week <- factor(format(weather$datetime,format="%A"),
                           levels = c("Monday", "Tuesday", "Wednesday","Thursday", "Friday", "Saturday", "Sunday"))
# Add day of week numbered in order from Monday=1 to Sunday=7
weather$day_number <- as.numeric(weather$day_week)
# Add formatted hour; Order starting with 4:00AM-8:00AM when the trains start for the day. 
# and end with 12:00AM, the last train of the day.
weather$hour_format <- factor(format(weather$datetime, format="%I:00 %p"),
                              levels = c("04:00 AM", "08:00 AM","12:00 PM", "04:00 PM", "08:00 PM", "12:00 AM"))

# Create numeric feature for ordered hourly period of the day. 4AM=1(1st trains) ... 12AM=6(last trains)
weather$hour_number <- as.numeric(weather$hour_format)

# Create a date-only column without hour date
weather$date <- as.POSIXct(weather$date, format="%m-%d-%Y")
weather$fog <- factor(weather$fog, levels=c(0,1), labels=c("No Fog", "Fog"))
weather$rain <- factor(weather$rain, levels=c(0,1), labels=c("No Rain", "Rain"))
weather$weekday <- factor(weather$weekday, levels=c(0,1), labels=c("Weekend", "Weekday"))
weather$date_day_week <- factor(format(weather$datetime,format="%m-%d-%Y:%A"))

```

### Creating training/testing sets

First, we create random sample splits for training and testing sets with stratified random sampling.

Stratified random sampling is done to help build a better predictive model by breaking the random sampling into groups(e.g. low, medium, high) to control for skewed distributions.[1] The default is 5 group sections. 

It is especially helpful for highly skewed dependent variables, like our entries hourly variable that is highly right-skewed.


```{r}

# Set a seed to make reproducible results.
set.seed(808)

# Create a random sampling of ~80% of the dataset for training. 20% for testing. 
# For numeric y, the sample is split into groups sections based on percentiles and sampling is done within these subgroups.
# For createDataPartition, the number of percentiles is set via the groups argument.
inTrain <- createDataPartition(weather$ENTRIESn_hourly, p=0.80, list=FALSE)
```

## Exploratory Data Analysis

<b><i>Disclaimer: We want to subset training/testing/validation sets and explore only the training set so that we are not influenced by our out-of-sample distributions when building our model. This is to ensure better generalizability. The exploration will be done only on the training dataset prior to building our model.</b></i>

Note: Hourly entries were actually entries per 4-hours of time.

### Check for collinearity in features.

<b>Collinearity</b> can be defined as "technical term for the situation where a pair of predictor variables have a substantial correlation with each other." [1] Collinearlity can cause problems in linear regression by causing the collinear coefficients to change erratically, the model to have problems converging to an optimal solution, or individual predictor coefficients to be unreliable. [2]

A quick look at a correlation matrix between all of our numeric variables doesn't show any large correlations between meaningful predictor variables. Longitude/latitude, entries/exits, and measurements/mean_measurements showed strong inter-variable correlations but no pairs were used for prediction. Longitude and mean windspeed also showed a mild correlation of 0.5.

There does not appear to be any evidence for collinear numerical variables that should be removed.

```{r, Figure-1, fig.width=12, fig.height=12}

# Create a training dataframe with our sample split.
training_df <- weather[inTrain,]

# Separate out our numeric variables to check correlations among them.
training_num <- training_df[,sapply(weather, is.numeric)]

# Create a correlation matrix.
M <- cor(training_num)

# Plot visual distributions on the lower diagonal, and numeric correlation values on the upper.
corrplot(M, method = "ellipse", order="hclust", type="lower", tl.cex=0.75, add = FALSE, tl.pos="lower") #plot matrix
corrplot(M, method = "number", order="hclust", type="upper", tl.cex=0.75, add = TRUE, tl.pos="upper")

```

### Day of the week

We can look at ridership by day of the week. We have to be careful here because some days total times recorded may be represented more often in our dataset by having more data for some days since there are 31 days worth of data. Missing data for time slots could also play some role. 

When normalizing the total entries for each day by the total days that the entries were recorded, we can see a more normal distribution of ridership centered around the middle of the work week. Ridership drops heavily on the weekend, then increases quickly for the work week.


```{r results='asis'}
# Just looking at total entries and days can be misleading, since there can be more of one day than others in our dataset.
dow <- training_df %>%
  group_by(day_week) %>%
  summarise(total_entries = sum(ENTRIESn_hourly))

# Create a smaller summary dataframe of the total entries for each day of the week to compare daily distributions.
# Here we can calculate the average ridership for each day of the week.
# There may be a more efficient workflow, but this quick and dirty way works for now.
dow2 <- training_df %>%
  # Get total entries counts for each day of the dataset
  group_by(date_day_week, day_week) %>%
  summarise(total_entries = sum(ENTRIESn_hourly)) %>%
  ungroup() %>%
  # Get total count of each day of the week from the dataset
  group_by(day_week) %>%
  # Get the mean total entries for each day of the week divided by the total # of days of each day in the dataset.
  mutate(total_days = n(),
         avg_entries_per_day = sum(total_entries) / mean(total_days)) %>%
  # Select only the relevant columns
  select(day_week, total_days, avg_entries_per_day) %>%
  # Remove duplicates of rows
  distinct(day_week, total_days, avg_entries_per_day)

```

```{r Day-riders, fig.width=12, fig.height=7, fig.margin=TRUE}
day_total <- ggplot(dow, aes(x=day_week, y=total_entries, fill=day_week)) + 
  geom_bar(stat="identity") + 
  scale_fill_brewer(palette="Set2") +
  theme(axis.text.x = element_text(angle=45)) + 
  guides(fill=FALSE) + 
  scale_y_continuous(labels=comma) + 
  xlab("") + ylab("Total Entries") + ggtitle("Total Ridership for Each Day of the Week \n") + 
  geom_text(aes(x=day_week, y=500000, label= paste0("Total    \nDays: ", total_days)), size=3 , data=dow2)

day_avg <- ggplot(dow2, aes(x=day_week, y=avg_entries_per_day, fill=day_week)) + 
  geom_bar(stat="identity") + 
  scale_fill_brewer(palette="Set2") +
  theme(axis.text.x = element_text(angle=45)) + 
  guides(fill=FALSE) + 
  scale_y_continuous(labels=comma) + 
  xlab("") + ylab("Total Entries") + ggtitle("Average Ridership per Day \nfor Each Day of the Week") + 
  geom_text(aes(x=day_week, y=100000, label= paste0("Total    \nDays: ", total_days)), size=3 , data=dow2)


grid.arrange(day_total, day_avg, ncol=2)

```

### Hours of the day

Hours should be expected to be more robust to changes in distributions since more of one day shouldn't equal more of one hour than another.

The distribution of entries seems to be bi-modal when arranging into daily schedules, where the first train is typically in the 4:00 AM to 8:00AM slot, and the last train of the day is in the 12:00AM to 4:00AM time slot. There seems to be a bi-modal distribution with peaks at noon/lunchtime and 8:00pm/after work. 

Normalizing doesn't look to have had a dramatic impact on the distribution.

```{r fig.width=12, fig.height=7}
dow_hour <- training_df %>%
  group_by(hour_format) %>%
  summarise(total_entries = sum(ENTRIESn_hourly),
            n = n(),
            normalized_entries = total_entries/n)

hour_total <- ggplot(dow_hour, aes(x=hour_format, y=total_entries, fill=hour_format)) +
  geom_bar(stat="identity") +
  scale_y_continuous(labels=comma) + 
  scale_fill_brewer(palette="Set3") + 
  guides(fill=FALSE) +
  xlab("") + ylab("Total Entries") + ggtitle("Total Ridership for each 4-hour Time Period") +
  theme(axis.text.x = element_text(angle=45)) 

hour_normalized <- ggplot(dow_hour, aes(x=hour_format, y=normalized_entries, fill=hour_format)) +
  geom_bar(stat="identity") +
  scale_y_continuous(labels=comma) + 
  scale_fill_brewer(palette="Set3") + 
  guides(fill=FALSE) +
  xlab("") + ylab("Total Entries") + ggtitle("Average Ridership for each 4-hour Time Period") +
  theme(axis.text.x = element_text(angle=45)) 

grid.arrange(hour_total, hour_normalized)

````

### Day and hour interaction

There is definite order in these two variables. A quick look at both together shows there may be a few interactions between the different levels of these variables. Most notably, visually, is if it's 8am and the weekend, ridership drops more significantly than either conditions alone.

This implies that day and hour, and possibly their interactions, may be important factors in hourly ridership prediction.

```{r fig.width=12, fig.height=7}
dow_hour <- training_df %>%
  group_by(day_week, hour_format) %>%
  summarise(total_entries = sum(ENTRIESn_hourly),
            n = n(),
            normalized_entries = total_entries/n)

ggplot(dow_hour, aes(x=hour_format, y=normalized_entries, fill=hour_format)) +
  geom_bar(stat="identity") +
  facet_grid(day_week~.) +
  scale_y_continuous(labels=comma) + 
  scale_fill_brewer(palette="Set3") +
  theme(legend.position="None") + 
  xlab("") + ylab("Total Entries")

```

### Total entries among different units

Unit number/location also looks to have a huge impact on entry counts. Each unit can have very different amounts of entries.

```{r fig.width=12, fig.height=7, warning=FALSE}

less_unit_names <- as.character(unique(training_df$UNIT))[seq(1,240,by=4)]

units <- training_df %>%
  group_by(UNIT) %>%
  summarise(total_entries = sum(ENTRIESn_hourly))

g1 <- ggplot(units, aes(x=UNIT, y=total_entries)) +
  geom_bar(stat="identity", fill="lightBlue", color="black") +
  theme(axis.text.x = element_text(angle=90)) +
  scale_y_continuous(labels=comma) + 
  scale_x_discrete(breaks=less_unit_names, labels=less_unit_names) + 
  xlab("Units in order (every 4th unit label shown)") + ylab("Total Entries per Unit")

g1

```

### Units in order.

When arranged by total entries, we see very popular stations taking up the lion's share of the entries. This implies that UNIT will be an important factor for hourly entry prediction.

'Top 40'/'Bottom 40' units show a right skewed distribution. There are 240 units in this data set.

```{r fig.width=12, fig.height=7}

units_order <- units %>%
  arrange(-total_entries)

u1 <- ggplot(units_order[1:40,], aes(x=reorder(UNIT, -total_entries), y=total_entries)) + 
  geom_bar(stat="identity", fill="lightBlue", color="black") +
  theme(axis.text.x = element_text(angle=90)) + 
  scale_y_continuous(labels=comma, limits=c(0, 1600000)) +
  xlab("Unit Number") + ylab("Total Entries") + ggtitle("Total Entries for each Unit - Top 40 Units")

u2 <- ggplot(units_order[200:240,], aes(x=reorder(UNIT, -total_entries), y=total_entries)) + 
  geom_bar(stat="identity", fill="lightBlue", color="black") +
  theme(axis.text.x = element_text(angle=90), axis.text.y = element_blank()) +
  scale_y_continuous(labels=comma, limits=c(0, 1600000)) +
  xlab("Unit Number") + ylab("") + ggtitle("Total Entries for each Unit - Bottom 40 Units")

grid.arrange(u1, u2, ncol=2)

```

#### Abandoned unit!

One unit, R464, has 0 entries for the entire month. Either data is missing, or this unit isn't in a very popular spot. ;)

```{r, results='asis'}

missing2 <- units %>%
  mutate(UNIT = as.character(UNIT)) %>%
  select(UNIT, total_entries)

stargazer(tail(missing2), summary=FALSE, header=FALSE, type="html", title="Bottom 6 Units")

```


### Total Entries Among Different Stations 

'Top 40'/'Bottom 40' stations show the same right skewed distribution. There are 207 subway stations in this data set.

```{r fig.width=12, fig.height=7}
stations <- training_df %>%
  mutate(station = stri_trans_totitle(station)) %>%
  group_by(station) %>%
  summarise(total_entries = sum(ENTRIESn_hourly)) %>%
  arrange(-total_entries)

s1 <- ggplot(stations[1:40,], aes(x=reorder(station, -total_entries), y=total_entries)) +
  geom_bar(stat="identity", fill="orange", color="black") +
  theme(axis.text.x = element_text(angle = 90)) +
  xlab("Station Name") + ylab("Total Entries") + ggtitle("Total Entries for each Station - Top 40 Stations") + 
  scale_y_continuous(labels=comma, limits=c(0, 2500000)) 

s2 <- ggplot(stations[167:207,], aes(x=reorder(station, -total_entries), y=total_entries)) +
  geom_bar(stat="identity", fill="orange", color="black") +
  theme(axis.text.x = element_text(angle = 90), axis.text.y = element_blank()) +
  xlab("Station Name") + ylab("") + ggtitle("Total Entries for each Station - Bottom 40 Stations") +
  scale_y_continuous(labels=comma, limits=c(0, 2500000)) 

grid.arrange(s1, s2, ncol=2)

```

#### Abandoned Station!

Curiously, one whole station, Aqueduct Track, has 0 entries for the entire month as well!

```{r results='asis'}

stargazer(tail(stations), summary=FALSE, header=FALSE, type="html", title="Bottom 6 Stations")

```

.

### Weather station entries

Weather stations also show similar patterns in popularity, with certain ones being closer to high traffic units and stations. This can be better visualized later, in a map of stations and units.

Here we create arbitrary numbers for each weather station based on its unique longitude/latitude coordinates.


```{r fig.width=12, fig.height=7}
# Then sum up total entries for each weather station. 
# Will be useful to plot on a map later.
weather_stations <- training_df %>%
  mutate(weather_station = factor(abs(weather_lon) + abs(weather_lat))) %>%
  mutate(weather_station = as.factor(as.numeric(weather_station))) %>%
  group_by(weather_station) %>%
  summarise(total_entries=sum(ENTRIESn_hourly)) %>%
  arrange(-total_entries)

ggplot(weather_stations, aes(x=reorder(weather_station, -total_entries), y=total_entries)) +
  geom_bar(stat="identity", fill="brown", color="black") + 
  theme(axis.text.x = element_text(angle = 90)) + scale_y_continuous(labels=comma) + 
  xlab("Unique Weather Station Number") + ylab("Total Entries") + 
  ggtitle("Total Entries for All Units belonging to each Weather Station")

```

## Rain

There is evidence of a statistically significant difference between rainy and non-rainy times, but this doesn't seem to be an appreciable difference when it comes to prediction when other more expressive features are used.

### Rain visualizations

First, can see the totals for each 4-hour time period period, separated by the presence of rain. The distribution of totals for each time period seems to be unaffected by the rainy days from the 15th to the 20th of the month.

```{r Rain-Stacked-Barchart, fig.width=11, fig.height=7}

rain <- training_df %>%
  group_by(datetime, rain) %>%
  summarise(total_entries = sum(ENTRIESn_hourly)) 

ggplot(rain, aes(x=datetime, y=total_entries, fill=rain)) + 
  geom_bar(stat="identity") +
  scale_x_datetime(expand=c(0,0), breaks = date_breaks("1 day"), labels = date_format("%m-%d-%y (%a)")) +
  theme(axis.text.x = element_text(angle = 90)) +
  guides(fill = guide_legend("Weather")) +
  xlab("") + ylab("Total Entries") +
  ggtitle("Cumulative Ridership per Hour\n Colored by Presence of Rain at Each Unit (Labeled by Day)")

```

Here we can see a histogram of hourly counts broken into bin sizes of 100 entries, given the presence of rain. The two distributions look to be very similar right-skewed distributions.

```{r Rain-Histogram, fig.width=11, fig.height=8}
ggplot(training_df, aes(x=ENTRIESn_hourly, fill=rain, order=-as.numeric(rain))) + 
  geom_histogram(position="dodge", alpha=1, binwidth=100, color="black") + 
  coord_cartesian(xlim=c(0,5000), ylim=c(0, 4000)) + 
  xlab("Hourly Entries - Bins of Size 100 (Limited to 5000)") + 
  ylab("Hourly Entries - Frequency per Bin") + 
  ggtitle("Hourly Entries Histogram - Rain vs. No Rain (Dodged)") + 
  guides(fill=guide_legend("Weather\nCondition")) +
  scale_y_continuous(breaks=seq(0, 5000, by=1000))
```

The distributions don't look very different, but perhaps they are still statistically different.

Note: All exploratory data analysis was done only on the 80% sample training set of 34,121 training examples so as not to have the test set influence the final model/features used for prediction. 

## Statistical Tests

First, we can test the normality of the distribution with the Shapiro-Wilk test.

The test has a maximum sample size limit of 5,000. This protects the test from erroneously rejecting the null hypothesis simply by feeding it more and more data. But we can at least get an intuition from this test by obtaining a uniformly random sample of the data and testing those for normality.
[3]

The null-hypothesis of this test is that the population is normally distributed. Thus if the p-value is less than the chosen alpha level, then the null hypothesis is rejected and there is evidence that the data tested are not from a normally distributed population.[4] 

Our p-value is very close to zero, (p-value < 2.2e-16) so we reject the null hypothesis that the data is normally distributed. This is not the same as outright accepting the alternative hypothesis that the data is not normally distributed, but the is seems very very unlikely to be normally distributed based on the extremely low p-value.


```{r Shapiro}

training_df %>%
  group_by(rain) %>%
  summarise(mean_entries = mean(ENTRIESn_hourly),
            variance_entries = var(ENTRIESn_hourly))

set.seed(808)
shapiro.test(sample(training_df$ENTRIESn_hourly, 5000))

```

We can use Welch's t-test to check the null hypothesis that there is no difference in hourly entries between rainy and non-rainy days. We can use a one-sided t-test to test whether one mean (rainy) is greater than another (non-rainy.)

Note: The t-test is invalid for small samples from non-normal distributions, but it should be valid for large enough samples from non-normal distributions.[5]

Here, we reject the NULL hypothesis that there is no difference between the means.

By specifying "greater", we are testing a one-tailed hypothesis that hourly entries are greater when it is raining than not raining.

A p-value of 3.346e-06 and large t-statistic of 4.5053 makes it very highly unlikely that we are getting these results simply due to chance.

The mean of 2034.506 hourly entries for rainy days is statistically different than the mean of 1850.125 for non-rainy days.

```{r T-test}

t.test(x = training_df[training_df$rain == "Rain",]$ENTRIESn_hourly,
       y = training_df[training_df$rain == "No Rain",]$ENTRIESn_hourly,
       paired=FALSE, var.equal=FALSE, alternative="greater")
```

# Linear Regression

## Gradient Descent

We'll choose linear regression to model our data, where each feature$(x_i)$ and a bias term$x_0$ is multiplied by a weight$(\theta_i)$ and added linearly to make a prediction.

$h_\theta(x) = \theta_0 + \theta_1x_1 + \theta_2x_2 + ... + \theta_nx_n$

$h(x) = \displaystyle \sum_{i=0}^n\theta_ix_i = \theta^Tx$

To assess our model's progress in approximating the observed values, we can define a cost function $J(\theta)$ which is basically variation of one half the sum of squared errors. [6]

$J(\theta) = \frac{1}{2} \displaystyle \sum_{i=1}^{m} (predicted_i - observed_i)^2$

$J(\theta) = \frac{1}{2} \displaystyle \sum_{i=1}^{m}(h_\theta(x^{(i)} - y^{(i)})^2$

### Running Gradient Descent

```{r Cost-Function}

# Cost function for gradient descent.
# Takes as input: h0, outcome outcome values, and theta vector.
compute_cost <- function(h0=NULL, values=NULL, theta=NULL){
    m = length(values)
    sum_of_square_errors <- sum( ( h0 - values)^2 ) 
    cost = 1/(2*m) * sum_of_square_errors 
    return(cost)
}
```

We would like to minimize this error so that our function predicts values as close to the real observed values as possible. 

We can do this with using an iterative method of Least Mean Squares. We will continually update $\theta$ as follows:

$\theta_j := \theta_j-\alpha\frac{\partial}{\partial \theta_j}J(\theta)$

where $\alpha$ is called the learning rate and is used to control the size of the step in the direction of the negative gradient to minimize $J(\theta)$.[6]

We start with an initial guess for $\theta$ (in our case all 0's) and update each theta in the direction to decrease $J$ by the partial derivative with respect to each theta.


```{r Gradient-Descent}
# Compute gradient descent, and cost function.
gradient_descent <- function(features=NULL, values=NULL, theta=NULL, alpha=NULL, num_iterations=NULL){
  m = length(values)
  # Initialize empty vector to keep track of cost for plotting.
  cost_history = rep(0, num_iterations)
  # Run gradient descent for num_iterations.
  for(i in seq(num_iterations)){
    # Compute the model function. Simple additive linear model in this case.
    # Competed outside the cost function and theta update to only compute once.
    h0 <- (features%*%theta)
    # Save each cost value for plotting.
    cost_history[i] <- compute_cost(h0, values, theta)
    # Update theta values 
    theta <- theta - t( (alpha/m) * t(h0 - values) %*% features)
  }
  # Return final theta values and the cost history in a list.
  return(list(theta = theta,
              cost_history = cost_history))
}

```

#### Creating dummy variables matrices for gradient descent.

Here we'll use only the categorical features and their interactions for our gradient descent algorithm. 

It turns out much of the variance can be explained by these features, and adding others won't add enough in terms of prediction compared to the added complexity of having more features.

```{r dummy-vars}

# Create a dummy matrix from the column of values. 
# Subtract 1 to remove the intercept column of 1's that is created by default. 
# This will be added in later only once.
unit_factor <- factor(weather$UNIT)
day_week_factor <- factor(weather$day_week)
hour_factor <- factor(weather$hour_format)
d1 <- model.matrix(~unit_factor-1)
d2 <- model.matrix(~day_week_factor-1)
d3 <- model.matrix(~hour_factor-1)

head(d3)

# Create dummy variables for the interaction between UNIT-x-day, UNIT-x-hour, and hour-x-day. 
unit_day <- paste0(as.character(weather$UNIT), "-", as.character(weather$day_week))
unit_hour <- paste0(as.character(weather$UNIT), "-", as.character(weather$hour_format))
day_hour <- paste0(as.character(weather$day_week), "-", as.character(weather$hour_format))

# Change these to factors and create dummy variables.
d4 <- model.matrix(~factor(unit_day)-1)
d5 <- model.matrix(~factor(unit_hour)-1)
d6 <- model.matrix(~factor(day_hour)-1)

# Combine all dummy variables into one matrix.
dummies <- cbind(d1, d2, d3, d4, d5, d6)

#Add the intercept term.
predictors <- cbind(1, dummies)

```

#### Creating training and testing sets.

```{r}

# Create both training and testing sets.
training <- predictors[inTrain,]
testing <- predictors[-inTrain,]

# Create outcome variables with the same random samples.
training_entries <- weather[inTrain,]$ENTRIESn_hourly
testing_entries <- weather[-inTrain,]$ENTRIESn_hourly

```

There are a total of 34,121 training examples, and 3416 dummy variables representing the main effects and interactions between the variables.
```{r}
dim(training)
```


```{r Run-Grad-Desc, cache=TRUE}
# Initialize n x 1 matrix of 0-valued theta values to start gradient descent with.
initial_theta <- matrix(rep(0, ncol(training)))

# Alpha learning rate of 1.4 is used, since 1.5 is too large and 
# causes an increase in the cost function after a few epoch iterations.
initial_alpha = 1.4
theta_cost <- gradient_descent(features=training, 
                               values=training_entries,
                               theta=initial_theta,
                               alpha=initial_alpha,
                               num_iterations=2500)

# Save theta values from returned list from gradient descent as a separate vector.
cost_history <- theta_cost$cost_history

```

Here we can see the cost history; the value of the cost function at every iteration. As long as the cost is going down at each iteration, we can be assured we are headed toward a minimum optima. 

If the cost starts going back up, our learning rate($\alpha$) may be too high and we should lower it. If we are only slowing declining, it may be too low and we can raise the learning rate($\alpha$).

At $\alpha$ = 1.5, the cost function started to increase after a few iterations, so we settled on $\alpha$ = 1.4

```{r Cost-History}
# Plot the cost history.
ggplot() + geom_point(aes(x=1:length(cost_history), y=cost_history)) + 
  xlab("Cost History") + ylab("Cost Value") +
  ggtitle(expression(paste("Cost history for gradient descent with ", alpha ," = 1.4"))) + 
  scale_y_continuous(labels=comma) 

# Save theta values from our returned gradient descent list to a separate vector. 
theta <- theta_cost$theta

```

### Measuring training model performance - Gradient Descent

$R^2$ can be used as a measure of "how well observed outcomes are replicated by the model, as the proportion of total variation of outcomes explained by the model."[7] 


$R^2 = 1 - \frac{ \displaystyle \sum_{i=1}^{m} \left( observed_i - predicted_i \right)^2 }{ \displaystyle \sum_{i=1}^{m} \left( observed_i - mean(observed) \right)^2 }$


$R^2 = 1 - \frac{ \displaystyle \sum_{i=1}^{m} \left( y_i - h(x_i) \right)^2 }{ \displaystyle \sum_{i=1}^{m} \left( y_i - \bar{y}_i \right)^2 }$


Adjusted $R^2$ behaves in much the same way as $R^2$, but is adjusted with a penalty for the number of features we have in our model as way to account for the added complexity when adding more features.[7]

Adjusted $R^2$ is computed as: $1 - (1 - R^2) \frac{n - 1}{n - p - 1}$


```{r}

# Compute the R^2 given the outcome values and predicted values.
compute_r_squared <- function(values, predictions){
  r_squared <- 1 - ( sum( (values - predictions)^2) / sum( (values - mean(values) )^2 ) )
}

# Compute the adjusted R^2 given R^2,number of examples used, and number of features used.
compute_adj_R <- function(r_square, num_examples, features_num){
  adj_R <- 1 - (1-r_square) * ((num_examples-1) /(num_examples - features_num - 1))
  return(adj_R)
}

# Compute predicted values for the training set from our resulting theta values.
prediction_train_gd <- training %*% theta 

# Compute the R^2 of the training predictions.
R_2_gd <- compute_r_squared(training_entries, prediction_train_gd)
R_2_gd

# Compute predictions for the test set based on our theta values from the model trained on the training set.
prediction_test_gd <- testing %*% theta

adj_R_square_gd  <- compute_adj_R(R_2_gd, 34121, 3416)
adj_R_square_gd

```

It looks like gradient descent is still decreasing the cost function very slightly after 2500 iterations with an $R^2$ of 0.8693 and adjusted $R^2$ of 0.8548. We could continue running more iterations and improve the fit slightly, but since this particular data set can fit into memory, we can find an exact solution with R's lm() function. 

## Ordinary Least Squares Linear Regression

Different variables were tested in an ordinary least squares model. By far the most important were those discussed earlier. 

Dummy variables were created internally in the lm() function for UNIT, hour, and day of the week. Interaction variables were also included for all variables. For instance, this could be done manually by creating interaction dummy variables for '8AM & Monday', '8AM & Tuesday', etc. as done earlier with gradient descent.

Temperature, precipitation, wind-speed and pressure added very little to the $R^2$ with coefficients that were not statistically significant or added very little with the presence of these 3 previous features and interactions, and were left out.

Some of these variables which were left out may have some predictive power, but it seems that much of the variance could be explained by the three time and location features (hour, day, and unit) that were explored.

### Measuring training model performance

lmFinal was chosen as the final model. Other variables such as rain were tested, but added very little (or even reduced the adjusted $R^2$.) and were not used. The following table shows the process of adding the features to the model.

```{r lm_models, results='asis', cache=TRUE}

# Create our testing dataframe for making predictions with the random sample indices generated earlier.
testing_df <- weather[-inTrain,]

training_weather_predictors <- training_df %>%
  select(ENTRIESn_hourly, hour_format, day_week, UNIT)
 

lm1 <- lm(ENTRIESn_hourly ~ UNIT, data=training_weather_predictors)

lm2 <- lm(ENTRIESn_hourly ~ UNIT + hour_format, data=training_weather_predictors)

lm3 <- lm(ENTRIESn_hourly ~ UNIT + hour_format + day_week, data=training_weather_predictors)

lmFinal <- lm(ENTRIESn_hourly ~ .*., data=training_weather_predictors)

```

```{r lm-stargazer-hang, eval=FALSE}
stargazer(lm1, lm2, lm3, lmFinal, title="Regression Results", header=FALSE, single.row = TRUE,  
          type="html", font.size="tiny", digits.extra=4, digits=4, keep="NULL",
          column.labels=c("UNIT only", "add 'hour'", "add 'day of week'", "add interactions"))
```
<table style="text-align:center"><caption><strong>Regression Results</strong></caption>
<tr><td colspan="5" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left"></td><td colspan="4"><em>Dependent variable:</em></td></tr>
<tr><td></td><td colspan="4" style="border-bottom: 1px solid black"></td></tr>
<tr><td style="text-align:left"></td><td colspan="4">ENTRIESn_hourly</td></tr>
<tr><td style="text-align:left"></td><td>UNIT only</td><td>add 'hour'</td><td>add 'day of week'</td><td>add interactions</td></tr>
<tr><td style="text-align:left"></td><td>(1)</td><td>(2)</td><td>(3)</td><td>(4)</td></tr>

<tr><td colspan="5" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">Observations</td><td>34,121</td><td>34,121</td><td>34,121</td><td>34,121</td></tr>

<tr><td style="text-align:left">R<sup>2</sup></td><td>0.3756</td><td>0.5166</td><td>0.5434</td><td>0.8718</td></tr>

<tr><td style="text-align:left">Adjusted R<sup>2</sup></td><td>0.3712</td><td>0.5131</td><td>0.54</td><td>0.8598</td></tr>

<tr><td style="text-align:left">Residual Std. Error</td><td>2362 (df = 33881)</td><td>2078 (df = 33876)</td><td>2020 (df = 33870)</td><td>1115 (df = 31211)</td></tr>

<tr><td style="text-align:left">F Statistic</td><td>85.27<sup>***</sup> (df = 239; 33881)</td><td>148.4<sup>***</sup> (df = 244; 33876)</td><td>161.2<sup>***</sup> (df = 250; 33870)</td><td>72.95<sup>***</sup> (df = 2909; 31211)</td></tr>
<tr><td colspan="5" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left"><em>Note:</em></td><td colspan="4" style="text-align:right"><sup>*</sup>p<0.1; <sup>**</sup>p<0.05; <sup>***</sup>p<0.01</td></tr>
</table>
```{r lm-stargazer-note, eval=FALSE, echo=FALSE}
# Added in html output from stargazer directly since it would hang when passed in the very large 'lmFinal' model 
# The template was created from the smaller models, then the values from the lmFinal were parsed in.
```

.

We get the same $R^2$(0.8718) and adjusted $R^2$(0.8598) for our model when we use our own $R^2$ function. We also get the exact same adjusted $R^2$ as our previous gradient descent model of 2,500 iterations.

```{r R2-test}
train_predictions <- predict(lmFinal, data=training_df)
train_entries <- training_df$ENTRIESn_hourly

train_R_squared <- compute_r_squared(values = train_entries, predictions = train_predictions)
train_R_squared

# Our lm() call on our training set had 34,121 training examples and 2,910 variables.
train_adjusted_R_squared <- compute_adj_R(train_R_squared,34121 ,2910)
train_adjusted_R_squared


```

## Evaluating the model's predictive power

<b>Mean square error</b> can be used as a metric to determine how well the model is making its predictions, when compared to the training set.[9]

This is very similar to the cost function in gradient descent. The difference here is instead of simply summing up all of the residual errors between the actual values and our predictions, we get the average squared error for each prediction made.

Mean squared error can be formally written as: $\textrm{MSE} = \frac{1}{m} \displaystyle \sum_{i=1}^{m} (y_i - \hat{y}_i)^2$

This is a measure of how of the variance of our predictions from the actual values. If the MSE stays approximately the same from our training set to our test set, our model generalizes well, and we are at much lower risk for over-fitting. [9]

```{r MSE}

test_predictions <- predict(lmFinal, newdata=testing_df) 
test_entries <- testing_df$ENTRIESn_hourly

# test_R_squared <- compute_r_squared(values=test_entries, predictions=test_predictions)
# test_R_squared

mean_square_error <- function(pred=NULL, actual=NULL){
  return( mean( (pred-actual)^2 ) )
}

train_MSE <- mean_square_error(pred=train_predictions, actual=train_entries)
train_MSE

test_MSE <- mean_square_error(pred=test_predictions, actual=test_entries)
test_MSE

```

We can also test the predictions from gradient descent after 2500 iterations. The results are very similar and look to be converging to the results from the OLS model.

```{r}
# Calculate MSE for training set
train_MSE_gd <- mean_square_error(pred=prediction_train_gd, actual=train_entries)
train_MSE_gd

# Calculate MSE for testing set
test_MSE_gd <- mean_square_error(pred=prediction_test_gd, actual=test_entries)
test_MSE_gd

```

.

We can also take the square root of the MSE to get the data back into comparable units.

```{r}
sqrt(train_MSE)

sqrt(test_MSE)

range(training_df$ENTRIESn_hourly)

```

Considering the range of the hourly entries, (0 - 32,814), our linear model is on average 1066.5 away from predicting correctly on the training set, and 1129.1 away from predicting on the test set. This seems to be pretty good for such a large range with our linear model, with data that looks to be inherently non-linear.

### Residuals

Another metric for evaluating the model's fit on the training data, as well as out-of-sample test data, is to look at residuals plots. We want residuals to be normally distributed around 0. [1]

Plots of the residuals show fairly normal distributions with compression close to 0, and fanning out at the upper ends. This also shows an under-prediction for higher ridership levels.

```{r lm-Residuals, fig.width=11, fig.height=8, size="small"}
pred1 <- predict(lmFinal)

r1 <- ggplot() + geom_histogram(aes(x=lmFinal$residuals), binwidth=200, fill="orange", color="black") + 
  xlab("Residuals") + ylab("Frequency") + ggtitle("Histogram of Residuals - Training Set")

r2 <- ggplot() + geom_point(aes(x=pred1, y=lmFinal$residuals)) +
  geom_hline(yintercept=0, linetype="dotted", size = 1.2, color = "red") + 
  xlab("Predictions") + ylab("Residuals") + ggtitle("Residuals against Predictions - Training Set")

r3 <- ggplot() + geom_point(aes(x=pred1, y=training_df$ENTRIESn_hourly)) + 
  geom_abline(slope = 1, linetype="dotted", color="red", size=1.2) + 
  xlab("Predictions") + ylab("Actual Hourly Entries") + ggtitle("Predictions against Hourly Entries - Training Set")

r4 <- ggplot() + geom_point(aes(x=1:nrow(training_df), y=lmFinal$residuals), data=training_df) +
  geom_hline(yintercept=0, linetype="dotted", size = 1.2, color = "red") + 
  xlab("Index") + ylab("Residuals") + ggtitle("Residuals in Sequential Order\nPer Hour Per Unit - Training Set")

grid.arrange(r1, r2, r3, r4, ncol=2)

```

```{r test-residuals, fig.width=11, fig.height=8}
test_residuals <- test_entries - test_predictions 

tr1 <- ggplot() + geom_histogram(aes(x=test_residuals), binwidth=200, fill="orange", color="black") + 
  xlab("Residuals") + ylab("Frequency") + ggtitle("Histogram of Residuals - Test Set")

tr2 <- ggplot() + geom_point(aes(x=test_predictions, y=test_residuals)) +
  geom_hline(yintercept=0, linetype="dotted", size = 1.2, color = "red") + 
  xlab("Predictions") + ylab("Residuals") + ggtitle("Residuals against Predictions - Test Set")

tr3 <- ggplot() + geom_point(aes(x=test_predictions, y=test_entries)) + 
  geom_abline(slope = 1, linetype="dotted", color="red", size=1.2) + 
  xlab("Predictions") + ylab("Actual Hourly Entries") + ggtitle("Predictions against Hourly Entries - Test Set")

tr4 <- ggplot() + geom_point(aes(x=1:nrow(testing_df), y=test_residuals), data=testing_df) +
  geom_hline(yintercept=0, linetype="dotted", size = 1.2, color = "red") + 
  xlab("Index") + ylab("Residuals") + ggtitle("Residuals in Sequential Order\nPer Hour Per Unit - Test Set")

grid.arrange(tr1, tr2, tr3, tr4, ncol=2)
```

### Conclusions

We achieved fairly high adjusted $R^2$ (0.8548) for our model, modeling a good portion of the variance in the data. We also had fairly good Root Mean Squared Error in our out-of-sample testing (1129.)

But, this data appears to be non-linear so it may be better served with non-linear transformations to some features or with non-linear models such as Multivariate Adaptive Regression Splines, Random Forests or Support Vector Regression. 

And perhaps more importantly, this is time-series data with definite temporal trends, and may be even better served with some form of time-series analysis. 

## Further Exploration

### Exploring the station locations

We can use JavaScript D3 visualization libraries in R with the Leaflet library to make calls to the OpenStreetMap API. [7]

Here we can see the locations of the most popular stations, and the weather units nearest them using the longitude/latitude coordinates for the stations and weather stations. Weather stations are purple circles, and subway stations are green. The size of the circle represents the total ridership for each station.

Clicking on each will also pop up the name and number of total entries. 

```{r Leaflet-Map, message=FALSE, fig.width=8, fig.height=7}

lonlat <- weather %>%
  group_by(longitude, latitude, station) %>%
  summarise(total_entries=sum(ENTRIESn_hourly)) %>%
  ungroup() %>%
  arrange(-total_entries) %>%
  mutate(station = paste0("Station: ", stri_trans_totitle(station) ),
         popups = paste0(station, "<br />Total Entries: ", comma(total_entries) ))

weather_lonlat <- weather %>%
  mutate(weather_station = factor(abs(weather_lon) + abs(weather_lat))) %>%
  mutate(weather_station = as.factor(as.numeric(weather_station))) %>%
  group_by(weather_lon, weather_lat, weather_station) %>%
  summarise(total_entries = sum(ENTRIESn_hourly)) %>%
  ungroup() %>%
  arrange(-total_entries) %>%
  mutate(weather_station = paste0("Weather Station: ", weather_station),
         popups = paste0(weather_station, "<br />Total Entries of Recorded Stations: ", comma(total_entries) ))

library(leaflet)

pal <- colorQuantile("YlGn", NULL, n=8)
pal2 <- colorQuantile("RdPu", NULL, n=8)
leaflet(lonlat) %>%
  addTiles() %>%
  addCircleMarkers(data=weather_lonlat, lng= ~weather_lon, lat=~weather_lat,
                   radius = ~total_entries/sum(total_entries)*100,
                   color= ~pal2(total_entries),
                   opacity= ~total_entries/sum(total_entries)*100,
                   popup= ~popups) %>%
  addCircleMarkers(radius = ~total_entries/sum(total_entries)*200, 
                   color = ~pal(total_entries),
                   opacity= ~total_entries/sum(total_entries)*200,
                   popup = ~popups) %>%
  setView(lng= -73.93, lat=40.74, zoom = 12)

```

### References

[1] Kuhn, M. & Johnson, K. Applied Predictive Modeling, ISBN 978-1-4614-6849-3 (eBook), Springer New York, 2013 

[2] http://en.wikipedia.org/wiki/Multicollinearity

[3] http://stackoverflow.com/questions/15427692/perform-a-shapiro-wilk-normality-test/15427746#15427746

[4] http://en.wikipedia.org/wiki/Shapiro%E2%80%93Wilk_test

[5] http://stats.stackexchange.com/questions/9573/t-test-for-non-normal-when-n50

[6] http://cs229.stanford.edu/notes/cs229-notes1.pdf

[7] http://en.wikipedia.org/wiki/Coefficient_of_determination

[8] Schutt, R. & O'Niel, C., Doing Data Science: Straight Talk from the Frontline, pp.66, Published by O’Reilly Media, 2014

[9] http://rstudio.github.io/leaflet/
