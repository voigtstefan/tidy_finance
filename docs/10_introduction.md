# Introduction to Tidy Finance

The main aim of this chapter is to familiarize yourself with the `tidyverse`. We start out by downloading and visualizing stock data before we move to a simple portfolio choice problem. These examples introduce you to our approach of *tidy finance*.

## Download and work with stock market data {#stock_market_data}



To download price data you can use the convenient `tidyquant`package. 
If you have trouble using `tidyquant`, check out the [documentation](https://cran.r-project.org/web/packages/tidyquant/vignettes/TQ01-core-functions-in-tidyquant.html#yahoo-finance). Start the session by loading the `tidyverse` and the `tidyquant` package as shown below. 


```r
# install.packages("tidyverse")
# install.packages("tidyquant")
library(tidyverse)
library(tidyquant)
```

We start and download daily prices for one stock market ticker, e.g. *AAPL*, directly from data provider Yahoo!Finance. To download the data you can use the command `tq_get`. If you do not know how to use it, make sure you read the help file by calling `?tq_get`. We especially recommended to take a look in the examples section in the documentation. 


```r
prices <- tq_get("AAPL", get = "stock.prices")
prices %>% head() # Take a glimpse on the data
```

```
## # A tibble: 6 x 8
##   symbol date        open  high   low close    volume adjusted
##   <chr>  <date>     <dbl> <dbl> <dbl> <dbl>     <dbl>    <dbl>
## 1 AAPL   2012-01-03  14.6  14.7  14.6  14.7 302220800     12.6
## 2 AAPL   2012-01-04  14.6  14.8  14.6  14.8 260022000     12.7
## 3 AAPL   2012-01-05  14.8  14.9  14.7  14.9 271269600     12.8
## 4 AAPL   2012-01-06  15.0  15.1  15.0  15.1 318292800     12.9
## # ... with 2 more rows
```

`tq_get` downloads stock market data from Yahoo!Finance if you do not specify another data source. The function returns a tibble with 8 quite self-explanatory columns: *symbol*, *date*, the market prices at the *open, high, low* and *close*, the daily *volume* (in number of shares) and the *adjusted* price in USD which factors in anything that might affect the stock price after the market closes, e.g. stock splits, repurchases and dividends.  

Next, we use `ggplot` to visualize the time series of adjusted prices. 

```r
prices %>% # Simple visualization of the downloaded price time series
  ggplot(aes(x = date, y = adjusted)) +
  geom_line() +
  labs(
    x = NULL,
    y = NULL,
    title = "AAPL stock prices",
    subtitle = "Prices in USD, adjusted for dividend payments and stock splits"
  ) +
  theme_bw()
```

<img src="10_introduction_files/figure-html/unnamed-chunk-4-1.png" width="768" style="display: block; margin: auto;" />
Next, we compute daily returns defined as $(p_t - p_{t-1}) / p_{t-1}$ where $p_t$ is the adjusted day $t$ price. 

```r
returns <- prices %>%
  arrange(date) %>%
  mutate(ret = adjusted / lag(adjusted) - 1) %>%
  select(symbol, date, ret)
returns %>% head()
```

```
## # A tibble: 6 x 3
##   symbol date            ret
##   <chr>  <date>        <dbl>
## 1 AAPL   2012-01-03 NA      
## 2 AAPL   2012-01-04  0.00537
## 3 AAPL   2012-01-05  0.0111 
## 4 AAPL   2012-01-06  0.0105 
## # ... with 2 more rows
```

The resulting tibble contains three columns where the last contains the daily returns. Note that the first entry naturally contains `NA` because there is no leading price. Note also that the computations require that the time series is ordered by date - otherwise, `lag` would be meaningless.
For the upcoming examples we remove missing values as these would require careful treatment when computing, e.g., sample averages. In general, however, make sure you understand why `NA` values occur and if you can simply get rid of these observations. 


```r
returns <- returns %>%
  drop_na(ret)
```
Next, we visualize the distribution of daily returns in a histogram. Just for fun, we also add a dashed red line that indicates the 5\% quantile of the daily returns to the histogram - this value is a (crude) proxy for the worst return of the stock with a probability of at least 5\%. 

```r
quantile_05 <- quantile(returns %>% pull(ret), 0.05) # Compute the 5 % quantile of the returns

returns %>% # create a histogram for daily returns
  ggplot(aes(x = ret)) +
  geom_histogram(bins = 100) +
  geom_vline(aes(xintercept = quantile_05),
    color = "red",
    linetype = "dashed"
  ) +
  labs(
    x = NULL, y = NULL,
    title = "Distribution of daily AAPL returns (in percent)",
    subtitle = "The dotted vertical line indicates the historical 5% quantile"
  ) +
  theme_bw()
```

<img src="10_introduction_files/figure-html/unnamed-chunk-7-1.png" width="768" style="display: block; margin: auto;" />

Here, `bins = 100` determines the number of bins and hence implicitly the width of the bins. Before proceeding, make sure you understand how to use the geom `geom_vline()` to add a dotted red line that indicates the 5\% quantile of the daily returns. 
A typical task before proceeding with *any* data is to compute summary statistics for the main variables of interest. 


```r
returns %>%
  summarise(across(
    ret,
    list(
      daily_mean = mean,
      daily_sd = sd,
      daily_min = min,
      daily_max = max
    )
  )) %>%
  kableExtra::kable(digits = 3)
```

<table>
 <thead>
  <tr>
   <th style="text-align:right;"> ret_daily_mean </th>
   <th style="text-align:right;"> ret_daily_sd </th>
   <th style="text-align:right;"> ret_daily_min </th>
   <th style="text-align:right;"> ret_daily_max </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:right;"> 0.001 </td>
   <td style="text-align:right;"> 0.018 </td>
   <td style="text-align:right;"> -0.129 </td>
   <td style="text-align:right;"> 0.12 </td>
  </tr>
</tbody>
</table>


```r
# Alternatively: compute summary statistics for each year
returns %>%
  group_by(year = year(date)) %>%
  summarise(across(
    ret,
    list(
      daily_mean = mean,
      daily_sd = sd,
      daily_min = min,
      daily_max = max
    )
  )) %>%
  kableExtra::kable(digits = 3)
```

<table>
 <thead>
  <tr>
   <th style="text-align:right;"> year </th>
   <th style="text-align:right;"> ret_daily_mean </th>
   <th style="text-align:right;"> ret_daily_sd </th>
   <th style="text-align:right;"> ret_daily_min </th>
   <th style="text-align:right;"> ret_daily_max </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:right;"> 2012 </td>
   <td style="text-align:right;"> 0.001 </td>
   <td style="text-align:right;"> 0.019 </td>
   <td style="text-align:right;"> -0.064 </td>
   <td style="text-align:right;"> 0.089 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2013 </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 0.018 </td>
   <td style="text-align:right;"> -0.124 </td>
   <td style="text-align:right;"> 0.051 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2014 </td>
   <td style="text-align:right;"> 0.001 </td>
   <td style="text-align:right;"> 0.014 </td>
   <td style="text-align:right;"> -0.080 </td>
   <td style="text-align:right;"> 0.082 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2015 </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 0.017 </td>
   <td style="text-align:right;"> -0.061 </td>
   <td style="text-align:right;"> 0.057 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2016 </td>
   <td style="text-align:right;"> 0.001 </td>
   <td style="text-align:right;"> 0.015 </td>
   <td style="text-align:right;"> -0.066 </td>
   <td style="text-align:right;"> 0.065 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2017 </td>
   <td style="text-align:right;"> 0.002 </td>
   <td style="text-align:right;"> 0.011 </td>
   <td style="text-align:right;"> -0.039 </td>
   <td style="text-align:right;"> 0.061 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2018 </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 0.018 </td>
   <td style="text-align:right;"> -0.066 </td>
   <td style="text-align:right;"> 0.070 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2019 </td>
   <td style="text-align:right;"> 0.003 </td>
   <td style="text-align:right;"> 0.016 </td>
   <td style="text-align:right;"> -0.100 </td>
   <td style="text-align:right;"> 0.068 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2020 </td>
   <td style="text-align:right;"> 0.003 </td>
   <td style="text-align:right;"> 0.029 </td>
   <td style="text-align:right;"> -0.129 </td>
   <td style="text-align:right;"> 0.120 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2021 </td>
   <td style="text-align:right;"> 0.001 </td>
   <td style="text-align:right;"> 0.016 </td>
   <td style="text-align:right;"> -0.042 </td>
   <td style="text-align:right;"> 0.054 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2022 </td>
   <td style="text-align:right;"> -0.002 </td>
   <td style="text-align:right;"> 0.016 </td>
   <td style="text-align:right;"> -0.027 </td>
   <td style="text-align:right;"> 0.025 </td>
  </tr>
</tbody>
</table>

## Scale the analysis up: tidyverse-magic
As a next step, we want to take the code from before and generalize it such that all the computations are performed for an arbitrary vector of tickers or even for all stocks that represent an index. Following tidy principles, it turns out to be actually quite easy to automate the download, generate the plot of the price time series and the table of summary statistics for an arbitrary number of assets.

This is where the `tidyverse` magic starts: tidy data and a tidy workflow make it extremely easy to generalize the computations from before to as many assets you like. The following code takes any vector of tickers, e.g., `ticker <- c("AAPL", "MMM", "BA")`, and automates the download as well as the plot of the price time series. At the end, we create the table of summary statistics for an arbitrary number of assets. 

Figure @ref(fig:prices) illustrates the time series of downloaded *adjusted* prices for each of the 30 constituents of the Dow Jones index. Make sure you understand every single line of code! (What is the purpose of `%>%`? What are the arguments of `aes()`? Which alternative *geoms* could you use to visualize the time series? Hint: if you do not know the answers try to change the code to see what difference your intervention causes). 


```r
ticker <- tq_index("DOW") # tidyquant delivers all constituents of the Dow Jones index
index_prices <- tq_get(ticker,
  get = "stock.prices",
  from = "2000-01-01"
) %>% # Exactly the same code as in the first part
  filter(symbol != "DOW") # Exclude the index itself

index_prices <- index_prices %>% # Remove assets that did not trade since January 1st 2000
  group_by(symbol) %>%
  mutate(n = n()) %>%
  ungroup() %>%
  filter(n == max(n)) %>%
  select(-n)

index_prices %>%
  ggplot(aes(
    x = date,
    y = adjusted,
    color = symbol
  )) +
  geom_line() +
  labs(
    x = NULL,
    y = NULL,
    color = NULL,
    title = "DOW index stock prices",
    subtitle = "Prices in USD, adjusted for dividend payments and stock splits"
  ) +
  theme_bw() +
  theme(legend.position = "none")
```

<img src="10_introduction_files/figure-html/fig:prices-1.png" width="768" style="display: block; margin: auto;" />

Do you notice the small differences relative to the code we used before? `tq_get(ticker)` is able to return a tibble for several symbols as well. All we need to do to illustrate all tickers instead of only one is to include `color = symbol` in the `ggplot` aesthetics. In this way, we can generate a separate line for each ticker.

The same holds for returns as well. Before computing returns as before, we use `group_by(symbol)` such that the `mutate` command is performed for each symbol individually. Exactly the same logic applies for the computation of summary statistics: `group_by(symbol)` is the key to aggregate the time series into ticker-specific variables of interest. 


```r
all_returns <- index_prices %>%
  group_by(symbol) %>% # we perform the computations per symbol
  mutate(ret = adjusted / lag(adjusted) - 1) %>%
  select(symbol, date, ret) %>%
  drop_na(ret)

all_returns %>%
  group_by(symbol) %>%
  summarise(across(
    ret,
    list(
      daily_mean = mean,
      daily_sd = sd,
      daily_min = min,
      daily_max = max
    )
  )) %>%
  kableExtra::kable(digits = 3)
```

<table>
 <thead>
  <tr>
   <th style="text-align:left;"> symbol </th>
   <th style="text-align:right;"> ret_daily_mean </th>
   <th style="text-align:right;"> ret_daily_sd </th>
   <th style="text-align:right;"> ret_daily_min </th>
   <th style="text-align:right;"> ret_daily_max </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> AMGN </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 0.020 </td>
   <td style="text-align:right;"> -0.134 </td>
   <td style="text-align:right;"> 0.151 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> AXP </td>
   <td style="text-align:right;"> 0.001 </td>
   <td style="text-align:right;"> 0.023 </td>
   <td style="text-align:right;"> -0.176 </td>
   <td style="text-align:right;"> 0.219 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> BA </td>
   <td style="text-align:right;"> 0.001 </td>
   <td style="text-align:right;"> 0.022 </td>
   <td style="text-align:right;"> -0.238 </td>
   <td style="text-align:right;"> 0.243 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> CAT </td>
   <td style="text-align:right;"> 0.001 </td>
   <td style="text-align:right;"> 0.020 </td>
   <td style="text-align:right;"> -0.145 </td>
   <td style="text-align:right;"> 0.147 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> CSCO </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 0.024 </td>
   <td style="text-align:right;"> -0.162 </td>
   <td style="text-align:right;"> 0.244 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> CVX </td>
   <td style="text-align:right;"> 0.001 </td>
   <td style="text-align:right;"> 0.017 </td>
   <td style="text-align:right;"> -0.221 </td>
   <td style="text-align:right;"> 0.227 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> DIS </td>
   <td style="text-align:right;"> 0.001 </td>
   <td style="text-align:right;"> 0.019 </td>
   <td style="text-align:right;"> -0.184 </td>
   <td style="text-align:right;"> 0.160 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> GS </td>
   <td style="text-align:right;"> 0.001 </td>
   <td style="text-align:right;"> 0.023 </td>
   <td style="text-align:right;"> -0.190 </td>
   <td style="text-align:right;"> 0.265 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> HD </td>
   <td style="text-align:right;"> 0.001 </td>
   <td style="text-align:right;"> 0.019 </td>
   <td style="text-align:right;"> -0.287 </td>
   <td style="text-align:right;"> 0.141 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> HON </td>
   <td style="text-align:right;"> 0.001 </td>
   <td style="text-align:right;"> 0.020 </td>
   <td style="text-align:right;"> -0.174 </td>
   <td style="text-align:right;"> 0.282 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> IBM </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 0.017 </td>
   <td style="text-align:right;"> -0.155 </td>
   <td style="text-align:right;"> 0.120 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> INTC </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 0.024 </td>
   <td style="text-align:right;"> -0.220 </td>
   <td style="text-align:right;"> 0.201 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> JNJ </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 0.012 </td>
   <td style="text-align:right;"> -0.158 </td>
   <td style="text-align:right;"> 0.122 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> JPM </td>
   <td style="text-align:right;"> 0.001 </td>
   <td style="text-align:right;"> 0.024 </td>
   <td style="text-align:right;"> -0.207 </td>
   <td style="text-align:right;"> 0.251 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> KO </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 0.013 </td>
   <td style="text-align:right;"> -0.101 </td>
   <td style="text-align:right;"> 0.139 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> MCD </td>
   <td style="text-align:right;"> 0.001 </td>
   <td style="text-align:right;"> 0.015 </td>
   <td style="text-align:right;"> -0.159 </td>
   <td style="text-align:right;"> 0.181 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> MMM </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 0.015 </td>
   <td style="text-align:right;"> -0.129 </td>
   <td style="text-align:right;"> 0.126 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> MRK </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 0.017 </td>
   <td style="text-align:right;"> -0.268 </td>
   <td style="text-align:right;"> 0.130 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> MSFT </td>
   <td style="text-align:right;"> 0.001 </td>
   <td style="text-align:right;"> 0.019 </td>
   <td style="text-align:right;"> -0.156 </td>
   <td style="text-align:right;"> 0.196 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> NKE </td>
   <td style="text-align:right;"> 0.001 </td>
   <td style="text-align:right;"> 0.019 </td>
   <td style="text-align:right;"> -0.198 </td>
   <td style="text-align:right;"> 0.155 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> PG </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 0.013 </td>
   <td style="text-align:right;"> -0.302 </td>
   <td style="text-align:right;"> 0.120 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> TRV </td>
   <td style="text-align:right;"> 0.001 </td>
   <td style="text-align:right;"> 0.018 </td>
   <td style="text-align:right;"> -0.208 </td>
   <td style="text-align:right;"> 0.256 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> UNH </td>
   <td style="text-align:right;"> 0.001 </td>
   <td style="text-align:right;"> 0.020 </td>
   <td style="text-align:right;"> -0.186 </td>
   <td style="text-align:right;"> 0.348 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> VZ </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 0.015 </td>
   <td style="text-align:right;"> -0.118 </td>
   <td style="text-align:right;"> 0.146 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> WBA </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 0.018 </td>
   <td style="text-align:right;"> -0.150 </td>
   <td style="text-align:right;"> 0.166 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> WMT </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 0.015 </td>
   <td style="text-align:right;"> -0.102 </td>
   <td style="text-align:right;"> 0.117 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> AAPL </td>
   <td style="text-align:right;"> 0.001 </td>
   <td style="text-align:right;"> 0.025 </td>
   <td style="text-align:right;"> -0.519 </td>
   <td style="text-align:right;"> 0.139 </td>
  </tr>
</tbody>
</table>

Note that you are now also equipped with all tools to download price data for *each* ticker listed in the SP500 index with the same number of lines of code. Just use `ticker <- tq_index("SP500")` which provides you with a tibble that contains each symbol that is (currently) part of the SP500. However, don't try this if you are not prepared to wait for a couple of minutes.

Sometimes, aggregation across other variables than `symbol` makes sense as well. For instance, suppose you are interested in the question: are days with high aggregate trading volume followed by high aggregate trading volume days? To provide some initial analysis on this question we take the downloaded tibble with prices and compute aggregate daily trading volume for all Dow Jones constituents in USD. Recall that the column *volume* is denoted in the number of traded shares. We multiply the trading volume with the daily closing price to get a measure of the aggregate trading volume in USD. Scaling by `1e9` denotes daily trading volume in billion USD.  


```r
volume <- index_prices %>%
  mutate(volume_usd = volume * close / 1e9) %>%
  group_by(date) %>%
  summarise(volume = sum(volume_usd))

volume %>% # Plot the time series of aggregate trading volume
  ggplot(aes(x = date, y = volume)) +
  geom_line() +
  labs(
    x = NULL, y = NULL,
    title = "Aggregate trading volume (billion USD)"
  ) +
  theme_bw()
```

<img src="10_introduction_files/figure-html/unnamed-chunk-11-1.png" width="768" style="display: block; margin: auto;" />

One way to illustrate the persistence of trading volume would be to plot volume on day $t$ against volume on day $t-1$ as in the example below:


```r
volume %>%
  ggplot(aes(x = lag(volume), y = volume)) +
  geom_point() +
  geom_abline(aes(intercept = 0, slope = 1), linetype = "dotted") +
  labs(
    x = "Previous day aggregate trading volume (billion USD)",
    y = "Aggregate trading volume (billion USD)",
    title = "Persistence of trading volume"
  ) +
  theme_bw() +
  theme(legend.position = "None")
```

```
## Warning: Removed 1 rows containing missing values (geom_point).
```

<img src="10_introduction_files/figure-html/aggregate-volume-1.png" width="768" style="display: block; margin: auto;" />

Do you understand where the warning `## Warning: Removed 1 rows containing missing values (geom_point).
` comes from and what it means? Pure eye-balling reveals that days with high trading volume are often followed by similarly high trading volume days.  

## Portfolio choice problems

A very typically question in Finance is how to optimally allocate wealth across different assets. The standard framework for optimal portfolio selection is based on investors that dislike portfolio return volatility and like higher expected returns: the mean-variance investor. An essential tool to evaluate portfolios is the efficient frontier, the set of portfolios which satisfy the condition that no other portfolio exists with a higher expected return but with the same standard deviation of return (i.e., the risk). Let us compute and visualize the efficient frontier for a number of stocks. First, we use our dataset to compute the *monthly* returns for each asset. 


```r
returns <- index_prices %>%
  mutate(month = floor_date(date, "month")) %>%
  group_by(symbol, month) %>%
  summarise(price = last(adjusted), .groups = "drop_last") %>%
  mutate(ret = price / lag(price) - 1) %>%
  drop_na(ret) %>%
  select(-price)
```

Next, we transform the returns from a tidy tibble into a $(T \times N)$ matrix with one column for each ticker to compute the covariance matrix $\Sigma$ and also the expected return vector $\mu$.
We compute the vector of sample average returns and the sample variance covariance matrix. 


```r
returns_matrix <- returns %>%
  pivot_wider(
    names_from = symbol,
    values_from = ret
  ) %>%
  select(-month)

sigma <- cov(returns_matrix)
mu <- colMeans(returns_matrix)
```

Then, we compute the minimum variance portfolio weight $\omega_\text{mvp}$ as well as the expected return $\omega_\text{mvp}'\mu$ and volatility $\sqrt{\omega_\text{mvp}'\Sigma\omega_\text{mvp}}$ of this portfolio. Recall that the minimum variance portfolio is the vector of portfolio weights that are the solution to 
$$\arg\min w'\Sigma w \text{ s.t. } \sum\limits_{i=1}^Nw_i = 1.$$
It is easy to show analytically, that $\omega_\text{mvp} = \frac{\Sigma^{-1}\iota}{\iota'\Sigma^{-1}\iota}$ where $\iota$ is a vector of ones.


```r
N <- ncol(returns_matrix)
iota <- rep(1, N)
wmvp <- solve(sigma) %*% iota
wmvp <- wmvp / sum(wmvp)

c(t(wmvp) %*% mu, sqrt(t(wmvp) %*% sigma %*% wmvp)) # Expected return and volatility
```

```
## [1] 0.008493932 0.031390401
```

Note that the *monthly* volatility of the minimum variance portfolio is of the same order of magnitude than the *daily* standard deviation of the individual components. Thus, the diversification benefits are tremendous!
Next we compute the efficient portfolio weights which achieves 3 times the expected return of the minimum variance portfolio. If you wonder where the solution $\omega_\text{eff}$ comes from: The efficient portfolio is chosen by an investor who aims to achieve minimum variance *given a minimum desired expected return* $\bar{\mu}$ such that her objective function is to choose $\omega_\text{eff}$ as the solution to
$$\arg\min w'\Sigma w \text{ s.t. } w'\iota = 1 \text{ and } \omega'\mu \geq \bar{\mu}.$$
The code below implements the analytic solution to this optimization problem, we encourage you to verify that it is correct. 


```r
# Compute efficient portfolio weights for given level of expected return
mu_bar <- 3 * t(wmvp) %*% mu # some benchmark return: 3 times the minimum variance portfolio expected return

C <- as.numeric(t(iota) %*% solve(sigma) %*% iota)
D <- as.numeric(t(iota) %*% solve(sigma) %*% mu)
E <- as.numeric(t(mu) %*% solve(sigma) %*% mu)

lambda_tilde <- as.numeric(2 * (mu_bar - D / C) / (E - D^2 / C))
weff <- wmvp + lambda_tilde / 2 * (solve(sigma) %*% mu - D / C * solve(sigma) %*% iota)
```

The two mutual fund theorem states that as soon as we have two efficient portfolios (such as the minimum variance portfolio and the efficient portfolio for another required level of expected returns like above), we can characterize the entire efficient frontier by combining these two portfolios. This is done in the code below. Make sure to familiarize yourself with the inner workings of the `for` loop!.


```r
# Use the two mutual fund theorem
c <- seq(from = -0.4, to = 1.9, by = 0.01) # Some values for a linear combination of two efficient portfolio weights
res <- tibble(
  c = c,
  mu = NA,
  sd = NA
)

for (i in seq_along(c)) { # A for loop
  w <- (1 - c[i]) * wmvp + (c[i]) * weff # A portfolio of minimum variance and efficient portfolio
  res$mu[i] <- 12 * 100 * t(w) %*% mu # Portfolio expected return (annualized, in percent)
  res$sd[i] <- 12 * 10 * sqrt(t(w) %*% sigma %*% w) # Portfolio volatility (annualized, in percent)
}
```

Finally, it is simple to visualize everything within one, powerful figure using `ggplot2`. 

```r
# Visualize the efficient frontier
res %>%
  ggplot(aes(x = sd, y = mu)) +
  geom_point() + # Plot all sd/mu portfolio combinations
  geom_point(
    data = res %>% filter(c %in% c(0, 1)),
    color = "red",
    size = 4
  ) + # locate the minimum variance and efficient portfolio
  geom_point(
    data = tibble(mu = 12 * 100 * mu, sd = 12 * 10 * sqrt(diag(sigma))),
    aes(y = mu, x = sd), color = "blue", size = 1
  ) + # locate the individual assets
  theme_bw() + # make the plot a bit nicer
  labs(
    x = "Annualized standard deviation (in percent)",
    y = "Annualized expected return (in percent)",
    title = "Dow Jones asset returns and efficient frontier",
    subtitle = "Red dots indicate the location of the minimum variance and efficient tangency portfolio"
  )
```

<img src="10_introduction_files/figure-html/unnamed-chunk-17-1.png" width="768" style="display: block; margin: auto;" />
The black efficient frontier indicates the set of portfolio a mean-variance efficient investor would choose from. Compare the performance relative to the individual assets (the blue dots) - it should become clear that diversifying yields massive performance gains (at least as long as we take the parameters $\Sigma$ and $\mu$ as given).