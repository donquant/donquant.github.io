---
layout: post
title: Backtesting Dual Momentum with ETFs. Part I.
header:
  feature: momentum.jpg
tags: 
  - Algorithmic Trading 
  - Momentum 
  - Quantitative Investing 
  - R
comments:
  true
---

About three years ago, in the year of 2015, I read a book by Gary Antonacci entitled ["Dual Momentum Investing"](https://www.amazon.com/Dual-Momentum-Investing-Innovative-Strategy/dp/0071849440).
If that happens you are not familiar with momentum, I would recommend it as an introductory text. 
In about two thirds of his book Gary rewiews and explains momentum in great detail.
At the end the author proposes a variety of strategies under common name _dual momentum_, which basically 
combines absolute and relative momentums with the ultimate goal to enchance investment performance.
Here in a series of posts I'm going to backtest couple of these strategies.

{% include toc.html %}

## Global Equity Momentum

First I'll backtest the simpliest of dual momentum strategies called _Global Equity Momentum_.
In the book and on a website you can find [past performance](http://www.optimalmomentum.com/gem_trackrecord.html) of this 
strategy from 1971 calculated with indexes, however in practice you can't trade pure
indexes and in fact Gary himself recommends to use <abbr title="Exchange-Traded Funds">ETF</abbr>s to 
implement this strategy. And that's exactly what I'm going to do.

In the original once a month you should compare 252 trading days returns of three ETFs representing 
allocations in U.S. equity, non U.S equity and U.S. high quality bonds and buy ETF with the biggest return 
for the next month, then each next month you reevaluate returns and adjust accordingly - stay in the same 
position if there are no changes or switch if changes happened. Backtesting just one strategy is pretty 
boring, so in order to bring more fun I'll extend it by introducing seven additional lookback periods as 
well as one frequency of reevaluation extended to two months. In total there are 16 strategies.

Gary argues that there is only one parameter in the strategy - a lookback period. But I definitely see that 
frequency and exact dates of adjustments may have its effects.

I also have to note that from one side I see the logic why <abbr title="Global Equity Momentum">GEM</abbr> 
is dual momentum strategy but from other side one can clearly simplify it to an ordinary relative strength 
momentum among these three ETFs.

## Methodology

#### Data
Adjusted prices of [SPDR S&P 500 ETF (SPY)](https://us.spdrs.com/etf/spdr-sp-500-etf-SPY),
[SPDR MSCI ACWI ex-US ETF (CWI)](https://us.spdrs.com/etf/spdr-msci-acwi-ex-us-etf-CWI),
[iShares Core U.S. Aggregate Bond ETF (AGG)](https://www.ishares.com/us/products/239458/ishares-core-total-us-bond-market-etf) from 2007-01-17 till
2018-01-01 obtained from Yahoo Finance. I decided to use these ETFs instead of less expensive Vanguard's 
alternatives because this combination has more historical data available for backtesting.

**Note:** For complete reproducibility in case of possible subsequent changes in Yahoo data all
resulted in the original script time series of adjusted prices also available on [github](https://github.com/donquant/code4blog/tree/master/data/GEM) in csv format. To download it in R use 
following commands:  
{: .notice--warning}

```r
require(xts)
ETF_prices = as.xts(read.zoo(file = "https://raw.githubusercontent.com/donquant/code4blog/master/data/GEM/ETF_prices.csv", header = T, index.column = 1, sep = ","))
```   

#### Backtesting period
2008-02-01 - 2018-01-01.

#### Lookback periods
105, 126, 147, 168, 189, 210, 231 and 252 days (5 - 12 months).

#### Reevaluating and switching dates
Last trading day of a month and two months.

#### Trading costs
All trading related costs such as commision, slippage, taxes etc. are set to zero for simplicity. 

## R Code

Such a strategy with multiple parameters can be easily backtested in [R](https://www.r-project.org/) using 
[`xts`](https://cran.r-project.org/web/packages/xts/vignettes/xts.pdf) objects, 
[`list`](https://cran.r-project.org/doc/manuals/r-devel/R-intro.html#Lists) 
data types and [`lapply()`](https://stat.ethz.ch/R-manual/R-devel/library/base/html/lapply.html) function. 
Also special functions from [xts](https://cran.r-project.org/web/packages/xts/index.html), 
[quantmod](http://www.quantmod.com/) 
and [PerformanceAnalytics](https://cran.r-project.org/web/packages/PerformanceAnalytics/index.html) 
packages are of great help. 

```r

# Global Equity Momentum with ETFs.

require(quantmod)

tickers = c("SPY","CWI","AGG")

getSymbols(tickers, from = "2007-01-17", to = "2018-01-01")

ETF_prices = cbind(Ad(SPY), Ad(CWI), Ad(AGG))


# ETFs Returns

look_back = as.list(seq(105, 252, 21))

GEMs = lapply(look_back, function(x) rollapply(ETF_prices, width = x, 
                                               function(y) (coredata(y[[x]]) - y[1]) / y[1], by = 1, fill = NULL))

# Signals/Weights

GEMs = append(lapply(GEMs, function(x) x[endpoints(index(x), on = "months"),]["2008/"]),
              lapply(GEMs, function(x) x[endpoints(index(x), on = "months"),]["2008/"][seq(1, dim(x[endpoints(index(x), on = "months"),]["2008/"])[1], 2)]))


GEMs = lapply(GEMs, function(x) as.xts(t(apply(x, 1, rank)), dateFormat = "Date"))

GEMs = lapply(GEMs, function(x) { y = unlist(x)
                                  y[y != 3] = 0
                                  y[y == 3] = 1 
                                  return(y) })

# GEM Returns

require(PerformanceAnalytics)

ETF_ret = cbind(dailyReturn(ETF_prices[,1]),
                dailyReturn(ETF_prices[,2]),
                dailyReturn(ETF_prices[,3]))

GEM_returns = lapply(GEMs, function(x) Return.portfolio(ETF_ret, x))

```
**Note:** This is an excerpt of code which only computes models returns. Full code with performance stats 
and plots is available on [github](https://github.com/donquant/code4blog/blob/master/GEM_with_plots.R).
{: .notice--warning}

## Momentum Illustrated

Here is an illustration of momentum on the example of GEM strategy with 252 days returns and two month 
revaluation. It's quite simple - the biggest return determines next period allocation. 

<figure>
  {% capture fig_img %}
  ![GEM's Returns](/assets/images/GEM_12_2M_allocations.png)
  {% endcapture %}
    
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption>Figure 1. GEM Illustration on the Example of "GEM 12 2M" Model.</figcaption>
</figure>

## Backtesting Results

{% include _post_scripts/GEMs.html %}

<figure>
  <noscript>  

  {% capture fig_img %}
  ![GEM's Returns](/assets/images/GEM_Equity.png)
  {% endcapture %}
    
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}

  </noscript>
  <figcaption>Figure 2. Equity Curves of GEM models, 2008-2018.</figcaption>
</figure>

In the beginnig SPY is outperformed by all GEMs due to the financial crisis of 2008. There is no surprise 
because GEM is designed to avoid such continious stock market crashes by a flight-to-quality (US bonds). 
But at the end, despite highly disadvantegeous start, SPY was able to make its way to the top and finished 
5th overall. However this shouldn't be a surprise either, because after the 2008 crash market was mostly in 
an impetuous uptrend and GEM misses rebounds due to its trailing nature, which is aknowleged by 
Gary. In my opinion there lies the main weakness of this model, but I will discuss it in some detail at the 
end. For now lets examine models performance statistics and also compare GEMs between themselfs.

| Investment Model | CAGR | Avg. Annual Return | Annual Volatility | Annualized Sharpe | Max Drawdown | Trades Per Year |
|:-----------:|:---------:|:---------:|:--------:|:-----------:|:------------:|:---------------:|
|  GEM 5 M    |**11.77%** |**12.53%** |  13.46   |  **1.12**   |   -15.77%    |       3.2       |
|  GEM 6 M    |   9.96%   |  10.33%   |   9.47   |    1.05     |    -9.92%    |       2.9       |
|  GEM 7 M    |   8.18%   |   8.62%   |  10.04   |    0.85     |   -12.98%    |       2.7       |
|  GEM 8 M    |   6.40%   |   6.92%   |  11.12   |    0.68     |   -21.56%    |       2.2       |
|  GEM 9 M    |   5.07%   |   5.45%   |   9.56   |    0.52     |   -13.60%    |       2.5       |
|  GEM 10 M   |   8.25%   |   8.65%   |   9.88   |    0.88     |   -16.12%    |       2.0       |
|  GEM 11 M   |   4.33%   |   4.91%   |  11.79   |    0.45     |   -21.33%    |       2.3       |
|**GEM 12 M** |   6.49%   |   7.12%   |  12.38   |    0.69     |   -18.00%    |       2.6       |
|  GEM 5 2M   |   7.47%   |   8.02%   |  11.22   |    0.76     |   -19.50%    |       2.8       |
|  GEM 6 2M   |  10.32%   |  10.67%   | **9.14** |    1.09     |  **-8.96%**  |       2.1       |
|  GEM 7 2M   |   9.40%   |   9.86%   |  10.27   |    0.97     |   -13.10%    |       1.9       |
|  GEM 8 2M   |   4.12%   |   4.56%   |  10.05   |    0.44     |   -22.29%    |       1.8       |
|  GEM 9 2M   |   4.96%   |   5.49%   |  11.18   |    0.48     |   -20.77%    |       1.8       |
|  GEM 10 2M  |   5.70%   |   6.11%   |   9.80   |    0.58     |   -22.54%    |       1.6       |
|  GEM 11 2M  |   6.71%   |   7.34%   |  12.41   |    0.65     |   -18.26%    |       1.5       |
|  GEM 12 2M  |   6.52%   |   7.14%   |  12.37   |    0.64     |   -21.12%    |       1.6       |
|  SPY        |   9.12%   |  10.73%   |  18.08   |    0.62     |   -46.32%    |     **0.1**     |

<figure>
  <figcaption>Table 1. Performance Statistics of GEM Models.</figcaption>
</figure>

Gary argues that 252 day lookback period and monthly reevaluation are optimal parameters for GEM model. It 
may be true for much longer period described in the book, but it's definitely not the case on the 2008-2018 
interval. The original GEM is outperformed by many alternative GEMs and SPY. The best performing models are 
with lower lookback periods (5-7 months). Also note that some GEMs with two months revaluations outperform 
monthly reevaluated counterparts. And finally, despite it's goal to avoid large drawdowns, GEM is 
nonetheless vulnerable to them, look at a big drawdown of the best model "GEM 5M" in 2015-2016.

## Final Thoughts

Now lets examine this strategy in some depth. Actually its strengths and weaknesses should be obvious 
straight from it's design. Look at SNP500 index. Roughly speaking you can see long continious (not in a 
mathematical sense) periods of growth and some major market crashes. It is straightforward that avoiding 
such crashes and catching up with uptrends would be very benefitial. And that is exactly what this strategy 
tries to implement. It uses time series momentum as a decision making algorithm. The weakness of time 
series momentum is it's following nature i.e. it needs time to react and therefore it won't totally protect 
from downtrends and also will miss market rebounds. You can artificially reverse engineer highly volatile 
uptrending market data that momentum will only catch dips and miss all ups. As a result you will have your 
capital destroyed very fast. For instance, assume you've implemented a one month lookback period momentum 
strategy revaluated monthly between some arbitrary stock and one month T-Bill with a constant monthly rate 
of two percent. Monthly stock prices and returns, as well as resulting strategy allocations are given in 
next table. 

| Month | Stock Price | Stock Return | T-Bill Return | Allocation |
|:-----:|:-----------:|:------------:|:-------------:|:----------:|
|   1   |    26       |      --      |        --     |    --      |
|   2   |    32       |    23.08%    |       2.00%   |    --      |
|   3   |    28       |   -12.50%    |       2.00%   |   Stock    |
|   4   |    34       |    21.43%    |       2.00%   |  T-Bill    |
|   5   |    30       |   -11.76%    |       2.00%   |   Stock    |
|   6   |    36       |    20.00%    |       2.00%   |  T-Bill    |
|   7   |    32       |   -11.11%    |       2.00%   |   Stock    |
|   8   |    38       |    18.75%    |       2.00%   |  T-Bill    |
|   9   |    34       |   -10.53%    |       2.00%   |   Stock    |
|   10  |    40       |    17.65%    |       2.00%   |  T-Bill    |
|   11  |    36       |   -10.00%    |       2.00%   |   Stock    |
|   12  |    42       |    16.67%    |       2.00%   |  T-Bill    |
|   13  |    38       |    -9.52%    |       2.00%   |   Stock    |
|   14  |    44       |    15.79%    |       2.00%   |  T-Bill    |
|   15  |    40       |    -9.09%    |       2.00%   |   Stock    |
|   16  |    46       |    15.00%    |       2.00%   |  T-Bill    |
|   17  |    42       |    -8.70%    |       2.00%   |   Stock    |
|   18  |    48       |    14.29%    |       2.00%   |  T-Bill    |
|   19  |    44       |    -8.33%    |       2.00%   |   Stock    |
|   20  |    50       |    13.64%    |       2.00%   |  T-Bill    |

<figure>
  <figcaption>Table 2. Hypothetical Momentum Strategy.</figcaption>
</figure>

The behavior of such a strategy is shown on figures 3 and 4. In twenty months this strategy will lose more 
than half of initial invested capital (10.000 $) while a stock price will almost double!

<figure>
  {% capture fig_img %}
  ![GEM's Returns](/assets/images/momentum_dp_920_900.png)
  {% endcapture %}
    
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption>Figures 3-4. Dynamics of Stock Price vs Invested Capital (Upper Figure).
  Stock and T-Bill Returns and Allocations (Lower Figure).</figcaption>
</figure>

Surely this is an exaggerated example and certainly market doesn't move to specially destroy one's strategy 
but nonetheless highly unfavorable conditions might occur (A good real example is SPY and GEM 5M on 
2015-2016 interval). So I would say that in highly volatile frequently oscillating conditions with 
unfavorable lengths of uptrends and downtrends GEM (in fact any momentum strategy) will lose its magic. 
Exact details will depend on data (frequency, length and amplitude of market oscillations) and parameters 
of a model.      

Overall it's a smart model, that aims to exploit past market conditions of continious growth periods and 
deep, long enougth, stock market crashes, but nonetheless is overfitted to these conditions and there is no 
guarantee that future market will behave in favorable fashion. However if you wanna bet that it will, this 
might be an interesting model to consider.
