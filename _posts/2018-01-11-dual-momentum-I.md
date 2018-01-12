---
layout: post
title: Backtesting Dual Momentum with ETFs. Part I.
header:
  feature: Momentum.jpg
tags: 
  - Algorithmic Trading 
  - Momentum 
  - Quantitative Investing 
  - R
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
indexes and in fact Gary himself recommends to use <abbr title="Exchange-Traded Funds">ETF</abbr>s to implement this strategy. And that's exactly what I'm going to do.

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
2018-01-01 obtained with yahoo finance database. I decided to use these ETFs instead of less expensive Vanguard's alternatives because this combination has more historical data available for backtesting.

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
              lapply(GEMs, function(x) x[endpoints(index(x), on = "months"),]["2008/"]
                                        [seq(1, dim(x[endpoints(index(x), on = "months"),]["2008/"])[1], 2)]))

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

**Note:** This is an excerpt of code which only computes models returns. Full code with performance stats and plots is 
available on github.
{: .notice--warning}

## Backtesting Results

{% include _post_scripts/GEMs.html %}

<figure>
  <noscript>  

  {% capture fig_img %}
  ![GEM's Returns](/assets/images/GEMs.png)
  {% endcapture %}
    
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}

  </noscript>
  <figcaption>Figure 1. Returns of GEM models.</figcaption>
</figure>

In the beginnig SPY is outperformed by all GEMs due to the financial crisis of 2008. There is no surprise 
because GEM is designed to avoid such continious stock market crashes by a flight-to-quality (US bonds). 
But at the end, despite highly disadvantegeous start, SPY was able to make its way to the top and finished 
5th overall. However this shouldn't be a surprise either, because after the 2008 crash market was mostly in 
an impetuous uptrend and GEM misses rebounds due to its trailing nature, which is aknowleged by 
Gary. In my opinion there lies the main weakness of this model, but I will discuss it in some detail at the 
end. For now lets examine models performance statistics and also compare GEMs between themselfs.

| Investment Model | CAGR | AAR | Annual Volatility | Annualized Sharpe | Max Drawdown | Trades Per Year |
|:-----------:|:--------:|:------------:|:----------:|:------------:|:--------------:|:---------------:|
|   GEM 5 M   |**0.1171**|**0.1246**    |  0.1341     |  **1.1140** |    0.1577      |       3.2       |
|   GEM 6 M   |  0.0989  |  0.1025      |  0.0943     |    1.0466   |    0.0992      |       2.9       |
|   GEM 7 M   |  0.0810  |  0.0854      |  0.1000     |    0.8460   |    0.1298      |       2.7       |
|   GEM 8 M   |  0.0628  |  0.0678      |  0.1096     |    0.6710   |    0.2156      |       2.3       |
|   GEM 9 M   |  0.0495  |  0.0532      |  0.0935     |    0.5109   |    0.1360      |       2.5       |
|   GEM 10 M  |  0.0813  |  0.0851      |  0.0967     |    0.8713   |    0.1612      |       2.0       |
|   GEM 11 M  |  0.0421  |  0.0477      |  0.1159     |    0.4345   |    0.2133      |       2.3       |
| **GEM 12 M**|  0.0637  |  0.0698      |  0.1218     |    0.6763   |    0.1800      |       2.6       |
|   GEM 5 2M  |  0.0741  |  0.0796      |  0.1115     |    0.7589   |    0.1950      |       2.8       |
|   GEM 6 2M  |  0.1025  |  0.1059      |**0.0911**   |    1.0827   |  **0.0913**    |       2.1       |
|   GEM 7 2M  |  0.0932  |  0.0978      |  0.1025     |    0.9602   |    0.1327      |       1.9       |
|   GEM 8 2M  |  0.0400  |  0.0442      |  0.0983     |    0.4232   |    0.2229      |       1.8       |
|   GEM 9 2M  |  0.0484  |  0.0536      |  0.1098     |    0.4664   |    0.2077      |       1.8       | 
|   GEM 10 2M |  0.0558  |  0.0597      |  0.0958     |    0.5701   |    0.2254      |       1.6       |
|   GEM 11 2M |  0.0659  |  0.0720      |  0.1225     |    0.6423   |    0.1826      |       1.5       |
|   GEM 12 2M |  0.0640  |  0.0700      |  0.1221     |    0.6258   |    0.2112      |       1.6       |
|   SPY       |  0.0907  |  0.1067      |  0.1804     |    0.6115   |    0.4632      |     **0.1**     |

<figure>
  <figcaption>Table 1. Performance statistics of GEM models.</figcaption>
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
capital destroyed very fast. Certainly market doesn't move to specially destroy one's strategy, but 
nonetheless highly unfavorable conditions might occur (A good example is SPY and GEM 5M on 2015-2016 
interval). So I would say that in highly volatile frequently oscillating conditions with unfavorable 
lengths of uptrends and downtrends GEM will lose its magic. Exact details will depend on data (frequency, 
length and amplitude of market oscillations) and parameters of the model.      

Overall it's a smart model, that aims to exploit past market conditions of continious growth periods and 
deep, long enougth, stock market crashes, but nonetheless is overfitted to these conditions and there is no 
guarantee that future market will behave in favorable fashion. However if you wanna bet that it will, this 
might be an interesting model to consider.
