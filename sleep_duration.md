# Sleep Duration by US State
Brian High  
![CC BY-SA 4.0](cc_by-sa_4.png)  

## Prevalence of healthly sleep duration

The paper:

Liu Y, Wheaton AG, Chapman DP, Cunningham TJ, Lu H, Croft JB. Prevalence of Healthy Sleep Duration among Adults — United States, 2014. MMWR Morb Mortal Wkly Rep 2016;65:137–141. DOI: http://dx.doi.org/10.15585/mmwr.mm6506a1.

Reports:

> The first state-specific estimates of the prevalence of a ≥7 hour sleep duration in a 24-hour period show geographic clustering of lower prevalence estimates for this duration of sleep in the southeastern United States and in states along the Appalachian Mountains, which are regions with the highest burdens of obesity and other chronic conditions. 

## Reproduce the choropleth

Let's try and reproduce the choropleth from the paper:

![](http://www.cdc.gov/mmwr/volumes/65/wr/figures/m6506a1f.gif)

FIGURE. Age-adjusted percentage of adults who reported ≥7 hours of sleep per 24-hour period, by state — Behavioral Risk Factor Surveillance System, United States, 2014

## Install Packages and Set Options

Load the required R packages, installing as necessary.


```r
for (pkg in c("knitr", "RMySQL", "dplyr", "choroplethrMaps", "choroplethr")) {
    if (! suppressWarnings(require(pkg, character.only=TRUE)) ) {
        install.packages(pkg, repos="http://cran.fhcrc.org", dependencies=TRUE)
        if (! suppressWarnings(require(pkg, character.only=TRUE)) ) {
            stop(paste0(c("Can't load package: ", pkg, "!"), collapse = ""))
        }
    }
}
```

Set `knitr` rendering options and the default number of digits for printing.


```r
opts_chunk$set(tidy=FALSE, cache=FALSE)
options(digits=4)
```

## Connect to MySQL Database

We will connect to the `localhost` and `brfss` database using an `anonymous` 
account.


```r
library(RMySQL)

con <- dbConnect(MySQL(), 
                 host="localhost", 
                 username="anonymous", 
                 password="Ank7greph-", 
                 dbname="brfss")
```

It's generally a *bad* idea to put your connection credentials in your script,
and an even *worse* idea to publish these on Github. *Don't be like me!*


```r
if (file.exists("con.R")) source("con.R")
```

A lesser evil is to put them in a separate file that you keep secure and private.

Even better would be to configure your system to prompt you for the password.

## Fetch the data

We need the state (`X_STATE`), age (`X_AGE80`) and sleep (`SLEPTIM1`) variables 
for 2014.


```r
sql <- "SELECT X_STATE AS StateNum, X_AGE80 AS Age, 
        COUNT(*) AS Respondents,
        COUNT(IF(SLEPTIM1 BETWEEN 7 AND 24, 1, NULL)) AS HealthySleepers 
        FROM brfss 
        WHERE IYEAR = 2014 
        GROUP BY X_STATE, X_AGE80 
        ORDER BY X_STATE, X_AGE80;"

rs <- dbGetQuery(con, sql)
dbDisconnect(con)
```

```
## [1] TRUE
```

## Calculate prevalence

Calculate percent prevalence by age for each state, then calculate the mean of
those prevalences by state. This is a crude form of "age-adjustment". These 
means will become the value of the shading in the choropleth.


```r
rs %>% group_by(Age) %>% 
    mutate(Prevalence = 100 * (HealthySleepers/Respondents)) -> sleepers
sleepers %>% select(StateNum, Age, Prevalence) %>% 
    group_by(StateNum) %>% 
    summarize(value=mean(Prevalence)) -> sleep.state
```

## Get state data

Get the list of states and their numbers from the 
[codebook](http://www.cdc.gov/brfss/annual_data/2014/pdf/codebook14_llcp.pdf).

We will do this in Bash. Todo: Do this in R, instead, for portability.


```bash
curl -o codebook14_llcp_states.pdf \
    http://www.cdc.gov/brfss/annual_data/2014/pdf/codebook14_llcp.pdf
pdftk codebook14_llcp.pdf cat 2-3 output codebook14_llcp_states.pdf
pdftotext -nopgbrk -layout codebook14_llcp_states.pdf states.txt
cat states.txt | awk '{ print $1, ",", $2, $3, $4 }' | \
    sed -e 's/ , /,/g' -e 's/ [ 0-9,.]*$//g' | \
    egrep '^[0-9]+,[A-Z][a-z]+' > states_list.csv
```

## Make choropleth map

Read in the state names and their number codes from the previous step. Merge 
with the prevalence values by state number. Make the choropleth map with 
these values. Use five bins to match the original figure.


```r
states <- read.csv("states_list.csv", header=F)
names(states) <- c("StateNum", "State")
states <- mutate(states, region=tolower(State))
merge(states, sleep.state) %>% select(region, value) %>% 
    filter(! region %in% c("guam", "puerto rico")) -> map.values
state_choropleth(map.values, num_colors = 5)
```

![](sleep_duration_files/figure-html/unnamed-chunk-7-1.png)\

The shading and bins are not exactly the same as the figure in the journal
article, but they are very similar. 

We did *not* "age-adjust" in the same way that the authors described:

> The age-adjusted prevalence and 95% confidence interval (CI) of the recommended healthy sleep duration (≥7 hours) was calculated by state and selected characteristics, and adjusted to the 2000 projected U.S. population aged ≥18 years.

We mearly took the mean of the prevelance of each age level by state, 
without taking into account any other "selected characteristics" or 
the "2000 projected U.S. population".