---
title: "Improving the performance of an investment fund"
author: "Jakub Hinc"
date: "`r Sys.Date()`"
output: github_document
---

```{r, message=FALSE, results='hide', echo=FALSE, error=FALSE}
library(dplyr)
library(plotly)
library(ggplot2)
library(tidyverse)
library(hrbrthemes)
library(viridis)
library(tidyr)
library(corrplot)
library(lpSolve)
library(tibbletime)
library(hrbrthemes)
library(knitr)
library(PortfolioAnalytics)
library(PerformanceAnalytics)
library(DEoptim)
library(ROI)
library(ROI.plugin.quadprog)
library(doParallel)
registerDoParallel(cores=4)
```

<br /> 

#### <strong> The purpose of the project </strong>
&nbsp;&nbsp;&nbsp;&nbsp; The purpose of the work is to investigate if Bitcoin is a good opportunity to invest. In total, I will analyze three possible investments: Bitcoin, S&P500, and Gold. It should give us information on whether to include Bitcoin in a portfolio from the viewpoint of risk, rate of return, and effectiveness.
<br />

#### <strong> Elements of work </strong>
  *  Data preparation <br />
  *  Characteristics of the investment <br />
  *  Changes in the rates of return of individual investments <br />
  *  Charts <br />
  *  Initial investment analysis <br />
  *  Hedge of inflation <br />
  *  Investment portfolio <br />
  *  Short sale <br />

#### <strong> Data</strong>
&nbsp;&nbsp;&nbsp;&nbsp; Data that I am going to analyze cover the period from September 2014 to November 2021. Bitcoin and S&P500 are daily data but Gold and CPI indicator are monthly. What's more two first investments has the following fields: <br />
  
  *  <strong>"Date"</strong> - The date, from September 2014 to November 2021, <br />
  *  <strong>"Open"</strong> - it's the price at the beginning of the day, <br />
  *  <strong>"High"</strong> - it's the highest price of instrument during day,<br />
  *  <strong>"Low"</strong> - it's the lowest price of instrument during day,<br />
  *  <strong>"Close"</strong> - it's the price at the end of the day which is the most important in our analyses,<br />
  *  <strong>"Volume"</strong> - presents the trading values of a given investment.<br />

#### <strong> Characteristics of the investment</strong>
  * <strong>Bitcoin</strong> - is a cryptocurrency created in 2009. It's based on blockchain which allows working without any administrator. It has become a famous investment opportunity that brings very high returns in a short time. <br />
  * <strong>S&P 500</strong> -  is a stock market index tracking the performance of 500 large companies listed on stock exchanges in the United States. It is one of the most commonly followed equity indices. <br />
  * <strong>Gold</strong> - Of all the precious metals, gold is the most popular as an investment. Investors generally buy gold as a way of diversifying risk, especially through the use of futures contracts and derivatives. <br />

# Data preparation 

&nbsp;&nbsp;&nbsp;&nbsp; First of all we need to load data, change type of date column from char to date and omit empty rows.
```{r, error=FALSE, message=FALSE}
#loading data
setwd("D:/projekty/bitcoin or riskcoin")
bitcoin <- read.csv('bitcoin-usd.csv', sep = ',')
sp500 <- read.csv('sp500.csv', sep = ',')
monthly_data <- read.csv('monthly_data.csv', sep = ',')
gold <- monthly_data[c(1, 2)]
CPI <- monthly_data[c(1, 3)]
#changing type of date column
bitcoin[['date']] <- as.Date(bitcoin[['date']], format = "%Y-%m-%d")
sp500[['date']] <- as.Date(sp500[['date']], format = "%Y-%m-%d")
gold[['date']] <- as.Date(gold[['date']], format = "%Y-%m-%d")
CPI[['date']] <- as.Date(CPI[['date']], format = "%Y-%m-%d")
#omitting empty rows
bitcoin <- na.omit(bitcoin)
gold <- na.omit(gold)
sp500 <- na.omit(sp500)
CPI <- na.omit(CPI)
```

&nbsp;&nbsp;&nbsp;&nbsp; If we want to compare these three possible investments we need to unify the date. First of all, investments must be presented in the same time frame. In our case gold and CPI is in the monthly time frame, so I decided to transform Bitcoin and S&P500 from daily to a monthly. 

```{r message=FALSE}
bitcoin_monthly <- mutate(bitcoin, date = collapse_index(date, "monthly")) 
bitcoin_monthly <- group_by(bitcoin_monthly, date) 
bitcoin_monthly <- summarise(bitcoin_monthly, open = first(open), high = max(high), low = min(low), close 
                             = last(close), volume = sum(volume))

sp500_monthly <- mutate(sp500, date = collapse_index(date, "monthly")) 
sp500_monthly <- group_by(sp500_monthly, date) 
sp500_monthly <- summarise(sp500_monthly, open = first(open), high = max(high), low = min(low), close 
                             = last(close), volume = sum(volume))
```

&nbsp;&nbsp;&nbsp;&nbsp;In order to create statistics based on our data, we need to add a column that contains the monthly rates of return on a specific investment. It gives us possibility to compare investment from a profitability point of view. In this case I am going to prepare this data based on logarithmic rate of return. It is better than the arithmetic because it is less sensitive to outliers. The use of an arithmetic rate of return may lead to an overestimation of the results.

```{r, message = FALSE, error=FALSE}
ror = function(value1, value2){
  log(value1/value2)
}
bitcoin_monthly$rate_of_return <- 0
for (i in 2:nrow(bitcoin_monthly)){
  bitcoin_monthly[i, "rate_of_return"] = ror(bitcoin_monthly[i, "close"], bitcoin_monthly[i - 1 ,"close"])
}
sp500_monthly$rate_of_return <- 0
for (i in 2:nrow(sp500_monthly)){
  sp500_monthly[i, "rate_of_return"] = ror(sp500_monthly[i, "close"], sp500_monthly[i - 1 ,"close"])
}
gold$rate_of_return <- 0
for (i in 2:nrow(gold)){
  gold[i, "rate_of_return"] = ror(gold[i, "gold_usd"], gold[i - 1,"gold_usd"])
}
```


# Charts 
&nbsp;&nbsp;&nbsp;&nbsp; Now when data are prepared we can make some analysis and basic statistics. To see how individual instruments have changed in a given period below are presented charts. Thanks to that we can notice performance of particular investments. As first will presented price of Bitcoin:  <br />
  
```{r,  error=FALSE}
fig_bitcoin_monthly <- bitcoin_monthly %>% 
  plot_ly(x = ~date, y = ~close, type = 'scatter', mode = 'lines', width=900, height=500) 
fig_bitcoin_monthly <- fig_bitcoin_monthly %>% 
  layout(title = "Price of Bitcoin", 
         xaxis = list(rangeslider = list(visible = F)))
fig_bitcoin_monthly
```
<br />
&nbsp;&nbsp;&nbsp;&nbsp; You can see that Bitcoin has grown significantly recently. Since 2020 it almost increased its value sixfold. But you can also notice that there is a lot of fluctuations which may be bad for the risk of ours portfolio. This is an investment that is very profitable, but it is very likely that it has reached its maximum, so an investment at this point can take huge losses. <br /><br /> 
Next chart presents the level at the closing of the trading day for S&P 500: <br />

```{r,  error=FALSE}
fig_sp500_monthly <- sp500_monthly %>% 
  plot_ly(x = ~date, y = ~close, type = 'scatter', mode = 'lines', width=900, height=500) 
fig_sp500_monthly <- fig_sp500_monthly %>% 
  layout(title = "The index level of S&P500", 
         xaxis = list(rangeslider = list(visible = F)))
fig_sp500_monthly
```
<br />
&nbsp;&nbsp;&nbsp;&nbsp; S&P 500 also has grown for last years but the level of increases can be assessed as more stable than in the case of Bitcoin. Investing in S&P500 can bring stable rate of return without undue risk. What we must also remember is that this index is diversified between the 500 largest companies in the USA, which additionally makes this instrument more secure.<br /><br />
The next plot shows the price of Gold which is presented by a line chart.  <br />

```{r,  error=FALSE}
fig_gold <- gold %>%
  plot_ly(x = ~date, y = ~gold_usd, type = 'scatter', mode = 'lines',
          width=900, height=500)
fig_gold <- fig_gold %>% 
  layout(title = "Price of Gold", yaxis = list(title = ''))
fig_gold
```
&nbsp;&nbsp;&nbsp;&nbsp; It is the most stable opportunity to invest from all three, you can also notice that in 2020 there was not be a bear market for Gold, because gold is the most-chosen investment during crises. It is considered a value-for-money instrument so we can predict that it will be the safest investment which is least correlated with market and others financial instruments in our portfolio. <br /> 

#### <strong>Conclusion</strong>
&nbsp;&nbsp;&nbsp;&nbsp; As we can see from the charts all of the three investing opportunities have been grown for last years. These are good possibilities to gain some return, especially Bitcoin which grew the most from all of the three but it was associated with increased risk, Especially in 2021 when Bitcoin has decreased almost 50%. Chart analysis doesn't give us enough information to decide if Bitcoin is a good opportunity for our portfolio. To better investigate whether Bitcoin is profitable investment we need to make more analysis, but at the moment we can predict that if we focus on increasing profit, we should decide to increase the share of bitcoin in our investment portfolio. However, if we focus more on risk reduction we should avoid this investment. 


# Initial analysis 
&nbsp;&nbsp;&nbsp;&nbsp; In order to investigate the most relevant parameters relating to investments from a portfolio I prepared descriptive statistics. Interpretation of the results allowed me to evaluate risk of undertaking selected investments and formed the overall picture of the subject the evolution of rates of return on these investments in the past. <br />

```{r, message = FALSE, fig.align='center', error=FALSE}
summary_bitcoin <- data.frame(unclass(summary(bitcoin_monthly$rate_of_return)), 
                              check.names = FALSE, stringsAsFactors = FALSE)
summary_sp500 <- data.frame(unclass(summary(sp500_monthly$rate_of_return)), 
                            check.names = FALSE, stringsAsFactors = FALSE)
summary_gold <- data.frame(unclass(summary(gold$rate_of_return)), check.names = FALSE, stringsAsFactors = FALSE)
summary <- cbind(summary_bitcoin, summary_sp500, summary_gold) 
names(summary) <- c("Bitcoin", "S&P500", "Gold")
row.names(summary) <- c("Minimum", "First quartile(Q1)", "Median", "Mean",
                        "Third quartile(Q1)", "Maximum")
kable(summary)
```

&nbsp;&nbsp;&nbsp;&nbsp; On the basis of the above table, it can be seen that the lowest average monthly rate return is characterized by investment in Bitcoin and amounts to about <strong> -45,27%</strong>. On the other hand, the highest monthly ror was recorded also for Bitcoin, as it is around <strong> 52,84%</strong>. If we look at the mean and median we can notice that Bitcoin has the highest rate of return from all three, it's much bigger than the rest. This confirms previous assumptions that Bitcoin is characterized by the highest volatility from all three investments. Of course if we we want to increase rate of return of our portfolio we should invest more money in Bitcoin, but we should remember that this is associated with increased risk.<br />
&nbsp;&nbsp;&nbsp;&nbsp; On the other hand Gold has the lowest ror  <strong>10,36% </strong> but also minimum  <strong>-7,54%</strong>. This shows that this investment has reduced volatile but we have to remember that this is assotiated with devalue profitability. This is also proven by median <strong> (0,07%) </strong> and mean <strong> (0,42%) </strong> which are also the lowest for this investment from all three. 
<br />
&nbsp;&nbsp;&nbsp;&nbsp; This is also shown in the chart below which presents the distribution of returns for all three investments. We notice as before that Bitcoin can be the most profitable investment but also the most variable. The distribution of its rates of return is very different from traditional investments like S&P500 and Gold, so at this moment if we want to decrease risk of our portfolio we shouldn't include Bitcoin in our investment set. <br />

```{r, fig.align='center', message = FALSE, error=FALSE}
bitcoin_monthly$name <- "Bitcoin"
sp500_monthly$name  <- "s&p500"
gold$name  <-  "Gold"

data_boxplot11 <- data.frame(value = bitcoin_monthly$rate_of_return,
                           name = bitcoin_monthly$name)
data_boxplot21 <- data.frame(value = sp500_monthly$rate_of_return,
                           name = sp500_monthly$name)
data_boxplot31 <- data.frame(value = gold$rate_of_return,
                           name = gold$name)
data_boxplot1 <- rbind(data_boxplot11, data_boxplot21, data_boxplot31)

data_boxplot1 %>%
  ggplot( aes(x=name, y=value, fill=name)) +
    geom_violin() +
    scale_fill_viridis(discrete = TRUE, alpha=0.6, option="A") +
    theme_ipsum() +
    theme(
      legend.position="none",
      plot.title = element_text(size=11)
    ) +
    ggtitle("Boxplot of rates of returns") +
    theme(plot.title = element_text(hjust = 0.5)) +
    ylab("")+
    xlab("")
```

#### <strong>Value of correlation coefficients</strong>

&nbsp;&nbsp;&nbsp;&nbsp; To minimize the risk of our portfolio we should join instruments that are negative or very low correlated. Therefore it is necessary to check at the beginning connections between all of them. As I mentioned previous all of the investment have been growing for last years, it is points that there probably will be positively correlated. As we can notice from chart below all of the investment are highly positive correlated:<br />

  *  Bitcoin and S&P500 - <strong> 0,91, </strong> <br />
  *  Bitcoin and Gold - <strong> 0,72, </strong> <br />
  *  S&P500 and Gold - <strong> 0,85. </strong> <br />
It may have a bad influence on the shape of our investment portfolio from the point of view of risk. Especially Bitcoin which is strongly correlated with S&P500. 
```{r, message = FALSE, fig.align='center', error=FALSE}
data_correlation <- data.frame(Bitcoin = bitcoin_monthly$close,
                   SP500 = sp500_monthly$close,
                   Gold = gold$gold_usd)
corrplot(cor(data_correlation),
  method = "number",
  type = "upper" 
)
```


# Hedge against inflation 
&nbsp;&nbsp;&nbsp;&nbsp; Now I will investigate whether Bitcoin is a good way to hedge against inflation. As we can see from the chart below CPI has been also grown for the last years so we can predict that it will be positively corelated with all three investment. In case to best hedge against inflation we should find instrument which will be the most corelated. At first, I found that gold should be the best the most fitted investment to CPI but we should investigate this by checking all three investments. 

```{r, message = FALSE, fig.align='center', error=FALSE}
fig_gold <- CPI %>%
  plot_ly(x = ~date, y = ~cpi_us, type = 'scatter', mode = 'lines',
          width=900, height=500)
fig_gold <- fig_gold %>% 
  layout(title = "CPI", yaxis = list(title = ''))
fig_gold

```

&nbsp;&nbsp;&nbsp;&nbsp; Below, we see a scattering graph of the CPI and the researched investments. At this moment we can notice that S&P500 is the most fitted investment with CPI. These are almost linear ratios, indicating that the growth of one component is related to the growth of the other. It tells us that S&P500 is the best investment opportunity to protect money against depreciation of our funds. It is also an investment that is less risky than Bitcoin which is also a big advantage of inflation hedge.

```{r, message = FALSE, fig.align='center',  error=FALSE}
data_correlation1 <- data.frame( Bitcoin = bitcoin_monthly$close,
                                SP500 = sp500_monthly$close,
                                Gold = gold$gold_usd,
                                CPI = CPI$cpi_us)

data_correlation1 %>%
  gather(-CPI, key = "var", value = "value") %>% 
  ggplot(aes(x = value, y = CPI)) +
    facet_wrap(~ var, scales = "free") +
    geom_point() +
    stat_smooth(formula = y ~ x, method = "loess")
```
&nbsp;&nbsp;&nbsp;&nbsp; Here we can see precise values of particular correlation. The results confirm that S&P500 is the best option for hedge of inflation, it is almost full correlation <strong>(0.96)</strong>. Second the best fitted investment is Gold <strong> (0,83) </strong> and the third Bitcoin <strong> (0,82) </strong>. The latter opportunity is the least suitable for hedging against inflation, as also indicated by the high standard deviation. <strong>If we want to choose the best investment to protect our funds against inflation we should decide to pick S&P500.</strong> Bitcoin is an investment that is the least correlated with CPI and the riskiest of all three so we should avoid this investment instrument.

```{r, message = FALSE, fig.align='center', error=FALSE}
corrplot(cor(data_correlation1),
  method = "number",
  type = "upper"
)
```


# Investment portfolio
&nbsp;&nbsp;&nbsp;&nbsp;An investment portfolio is an investor's collection of financial and real assets. Currently the most popular is the portfolio theory, which was published by Harry Markowitz in a magazine The Journal of Finance. The basis of this theory is diversification, that is, the division of capital into several specific investments in order to reach a trade-off between the rate of return and risk. Efficiency of the portfolio we test by using the Sharp's coefficient, which examines the dependence of the rate of return on the borne risk. In this indicator, we assumed the risk-free rate as the interest rate on 10-year US government bonds, which was 1,56% in November 2021. <br />
```{r, fig.align='center', message = FALSE, error=FALSE}
setwd("D:/projekty/bitcoin or riskcoin")
bitcoin1 <- bitcoin_monthly$rate_of_return
sp5001 <- sp500_monthly$rate_of_return
gold1 <- gold$rate_of_return
#importing weights from file
weights3inv <- read.table("weights3inv.txt",dec=",", header=TRUE, quote="\"",stringsAsFactors=FALSE)
w1 <- weights3inv$W1			
w2 <- weights3inv$W2
w3 <- weights3inv$W3
#calculating SD
s1 <- sd(bitcoin1)
s2 <- sd(sp5001)
s3 <- sd(gold1)
#Calculating corellation
corr12 <- cor(bitcoin1, sp5001)
corr13 <- cor(bitcoin1, gold1)
corr23 <- cor(sp5001, gold1)
#calculating ip
iportfolio <- mean(bitcoin1)*w1+mean(sp5001)*w2+mean(gold1)*w3
#portfolio risk
sdp <- (w1^2*s1^2 + w2^2*s2^2 + w3^2*s3^2 + 2*w1*w2*s1*s2*corr12 + 2*w1*w3*s1*s3*corr13 + 2*w2*w3*s2*s3*corr23)^0.5
#calculating effectivness
rf = 0.0156
sharp=(iportfolio-rf)/sdp
#preparing df with results
data <- cbind(w1, w2, w3, iportfolio, sdp, sharp)
data <- as.data.frame(data)
#finding interesting portfolios
min.risk <- subset(data, data$sdp==min(data$sdp))
max.effectivness <- subset(data, data$sharp==max(data$sharp))
max.ip <- subset(data, data$iportfolio==max(data$iportfolio))
max.w1 <- subset(data, data$w1==1)
max.w2 <- subset(data, data$w2==1)
max.w3 <- subset(data, data$w3==1)
des <- c("Minimal risk portfolio", "Maximum efficiency portfolio", "Maximum rate of return portfolio", "Max weight of first investment", "Max weight of second investment", "Max weight of third investment")
#Creating table with results 3 portfolios and showing results in console
results_without_ss <- cbind(rbind(min.risk, max.effectivness, max.ip, max.w1, max.w2, max.w3), des)
results_without_ss <- data.frame(unclass(results_without_ss), check.names = FALSE, stringsAsFactors = FALSE)
names(results_without_ss) <- c("w1", "w2", "w3", "Rate of return", "Risk", "Sharp", "Description")
df <- data.frame(sdp, iportfolio)
p <- ggplot(df, aes(sdp, iportfolio), type = "p") + geom_point(alpha = 0.5, color = "bisque1") + 
  ggtitle("Set of investment opportunities") + theme(plot.title = element_text(hjust = 0.5)) + 
  xlab("Portfolio risk") + 
  ylab("Portfolio rate of return") + xlim(0, 0.25) + ylim(0, 0.06) + 
  geom_point(aes(min.risk$sdp, min.risk$iportfolio), pch=19, col="darkgreen") + 
  geom_point(aes(max.effectivness$sdp, max.effectivness$iportfolio), pch=19, col="blue") +  
  geom_point(aes(max.w1$sdp, max.w1$iportfolio), pch=19, col="black") +
  geom_point(aes(max.w2$sdp, max.w2$iportfolio), pch=19, col="black") +
  geom_point(aes(max.w3$sdp, max.w3$iportfolio), pch=19, col="black") +
  annotate("text", x=0.22, y=0.048, label= "Maximum \n RoR portfolio \n and  efficiency \n Bitcoin (100%)") +
  annotate("text", x=0.008, y=0.01, label= "Minimum risk \nportfolio") +
  annotate("text", x=0.07, y=0.01, label= "S&P500 (100%)") +
  annotate("text", x=0.06, y=0.004, label= "Gold (100%)")
p
```
&nbsp;&nbsp;&nbsp;&nbsp; As we can see our set of investment opportunities is very distorted by Bitcoin. The one element portfolio of this instrument is very separate from the rest. This is due to higly risk and rate of return from this investment. Because of this fact this is also the most profitable portfolio.<br />
&nbsp;&nbsp;&nbsp;&nbsp; The surprising thing is that the maximum efficiency portfolio is also located in the same place as the one-element portfolio of Bitcoin. This tells us that despite the high risk the profitability of this investment is so high that we should risk. We can notice that the rest of the one-element portfolios are very close to the minimum risk portfolio, so we can predict that if we want to minimize the risk of our funds we should choose between these two investments. 

#### <strong>The share of instruments for the portfolio with the lowest risk [%]</strong>

&nbsp;&nbsp;&nbsp;&nbsp; The chart below presents the composition of the investment portfolio for the lowest risk. As we can see if we want to avoid risk we should invest <strong> 58,7%</strong> in Gold, <strong> 40%</strong> in S&P500 and <strong> 1,3% </strong> in Bitcoin. We may be surprised after the initial analysis that bitcoin is in this portfolio. This investment is highly risky but our model decided to diversify the portfolio also among this instrument. In turn, the largest share in the portfolio corresponds to investments in Gold, which may suggest that the investment is safe, nevertheless, it is lower than the others portfolio components' rate of return.


```{r, fig.align='center', message = FALSE, fig.height = 5, fig.width = 5, error=FALSE}
min_ryz <- data.frame(
  instrument = c("Bitcoin", "S&P500", "Gold"),
  wagi = c(results_without_ss$w1[1], results_without_ss$w2[1], results_without_ss$w3[1])
)
min_ryz <- min_ryz %>%
  arrange(desc(instrument)) %>%
  mutate(lab.ypos = cumsum(wagi) - 0.5*wagi)
por_min_risk1 <- ggplot(min_ryz, aes(x="", y=wagi, fill = instrument)) +
  geom_bar(stat="identity", width=1) +
  coord_polar("y", start=0) + 
  scale_fill_manual(values=c("gray48", "bisque1", "cornsilk3"))+
  theme(plot.title = element_text(hjust = 1)) + 
  geom_text(aes(y = lab.ypos, label = wagi*100 ), color = "black") + 
  theme(legend.position = c(1.11, 0.9)) + 
  theme_minimal()
por_min_risk1
```

#### <strong>The share of instruments for the portfolio with maximum efficiency [%]</strong>

&nbsp;&nbsp;&nbsp;&nbsp; This chart presents share of instruments for maximum efficiency portfolio. As I mentioned before for this case, Bitcoin's share is 100%. It's can be suprprised because of the fact that this investment is highly risky. But as I mentioned before this shows that the rate of return is so high that it rewards the risk incurred.

```{r, fig.align='center',  message = FALSE, fig.height = 5, fig.width = 5, error=FALSE}
max_ef <- data.frame(
  instrument = c("Bitcoin", "S&P500", "Gold"),
  wagi = c(results_without_ss$w1[2], results_without_ss$w2[2], results_without_ss$w3[2])
)
max_ef <- max_ef %>%
  arrange(desc(instrument)) %>%
  mutate(lab.ypos = cumsum(wagi) - 0.5*wagi)
por_max_eff1 <- ggplot(max_ef, aes(x="", y=wagi, fill = instrument)) +
  geom_bar(stat="identity", width=1) +
  coord_polar("y", start=0) + 
  scale_fill_manual(values=c("gray48", "bisque1", "cornsilk3"))+
  theme(plot.title = element_text(hjust = 1)) + 
  geom_text(aes(y = lab.ypos, label = wagi*100 ), color = "black") + 
  theme(legend.position = c(1.11, 0.9)) + 
  theme_minimal()
por_max_eff1
```

&nbsp;&nbsp;&nbsp;&nbsp;Below we can see the table with accurate results for the most relevant portfolios marked on the chart. We should focus on sharpe ratio results which are very significant in our analysis. If we look at this column we can notice that only for one element portfolio of Bitcoin this indicator is positive. It tells us that from all investment portfolios below only this one is effective. Practically so in terms of sharpe ratio if we want to reduce volatility we shouldn't choose a portfolio with the lowest risk which is presented above because we can simply invest this money in government bonds which can bring us higher profits with lower risk. 

```{r,  message = FALSE, echo=FALSE, error=FALSE}
kable(results_without_ss)
```


### <strong>Short sale</strong>

&nbsp;&nbsp;&nbsp;&nbsp; We can also create an opportunity set with the short sale which means investing in such a way that the investor will profit if the value of the asset falls. This is the opposite of a more conventional "long" position, where the investor will profit if the value of the asset rises. 
```{r, fig.align='center',  message = FALSE, error=FALSE}
setwd("D:/projekty/bitcoin or riskcoin")
weights3inv <- read.table("weights3invss.txt",dec=",", header=TRUE, quote="\"",stringsAsFactors=FALSE)
w1 <- weights3inv$W1
w1 <- as.numeric(w1)
w2 <- weights3inv$W2
w2 <- as.numeric(w2)
w3 <- weights3inv$W3
w3 <- as.numeric(w3)
#calculating SD
s1 <- sd(bitcoin1)
s2 <- sd(sp5001)
s3 <- sd(gold1)
#Calculating corellation
corr12 <- cor(bitcoin1, sp5001)
corr13 <- cor(bitcoin1, gold1)
corr23 <- cor(sp5001, gold1)
#calculating ip
iportfolio <- mean(bitcoin1)*w1+mean(sp5001)*w2+mean(gold1)*w3
#portfolio risk
sdp <- (w1^2*s1^2 + w2^2*s2^2 + w3^2*s3^2 + 2*w1*w2*s1*s2*corr12 + 2*w1*w3*s1*s3*corr13 + 2*w2*w3*s2*s3*corr23)^0.5
#calculating effectivness
rf = 0.0156
sharp=(iportfolio-rf)/sdp
#preparing df with results
data <- cbind(w1, w2, w3, iportfolio, sdp, sharp)
data <- as.data.frame(data)
#finding interesting portfolios
min.risk <- subset(data, data$sdp==min(data$sdp))
max.effectivness <- subset(data, data$sharp==max(data$sharp))
max.ip <- subset(data, data$iportfolio==max(data$iportfolio))
max.w1 <- subset(data, data$w1==1 & data$w2==0 & data$w3==0)
max.w2 <- subset(data, data$w1==0 & data$w2==1 & data$w3==0)
max.w3 <- subset(data, data$w1==0 & data$w2==0 & data$w3==1)
des <- c("Minimal risk portfolio", "Maximum efficiency portfolio", "Maximum rate of return portfolio", "Max weight of first investment", "Max weight of second investment", "Max weight of third investment")
#Creating table with results 3 portfolios and showing results in console
results_with_ss <- cbind(rbind(min.risk, max.effectivness, max.ip, max.w1, max.w2, max.w3), des)
names(results_with_ss) <- c("w1", "w2", "w3", "Rate of return", "Risk", "Sharp", "Description")
#creating and saving OS
datawithoutSS <-subset(data, data$w1>=0)
datawithoutSS <-subset(datawithoutSS, datawithoutSS$w2>=0)
datawithoutSS <-subset(datawithoutSS, datawithoutSS$w3>=0)
df <- data.frame(sdp, iportfolio)
df2 <- data.frame(datawithoutSS$sdp, datawithoutSS$iportfolio)
p1 <- ggplot(df, aes(sdp, iportfolio), type = "p") + geom_point(alpha = 0.5, color = "gray48") + 
  ggtitle("Opportunity set for three risky assets with short sale") + theme(plot.title = element_text(hjust = 0.5)) + 
  xlab("Portfolio risk") + 
  ylab("Portfolio rate of return") + xlim(0, 0.25) + ylim(-0.06, 0.065) +
  geom_point(data = df2, aes(x = datawithoutSS$sdp, y = datawithoutSS$iportfolio), color = "bisque1") + 
  geom_point(aes(max.ip$sdp, max.ip$iportfolio), pch=19, col="yellow") +
  geom_point(aes(min.risk$sdp, min.risk$iportfolio), pch=19, col="darkgreen") + 
  geom_point(aes(max.effectivness$sdp, max.effectivness$iportfolio), pch=19, col="blue") +  
  geom_point(aes(max.w1$sdp, max.w1$iportfolio), pch=19, col="black") +
  geom_point(aes(max.w2$sdp, max.w2$iportfolio), pch=19, col="black") +
  geom_point(aes(max.w3$sdp, max.w3$iportfolio), pch=19, col="black") +
  annotate("text", x=0.24, y=0.053, label= "Maximum \nRoR portfolio \n and efficiency")+
  annotate("text", x=0.19, y=0.065, label= "Bitcoin (100%)") +
  annotate("text", x=0.008, y=0.01, label= "Minimum risk \nportfolio") +
    annotate("text", x=0.07, y=0.01, label= "S&P500 (100%)") +
    annotate("text", x=0.06, y=0.003, label= "Gold (100%)") 
p1
```

&nbsp;&nbsp;&nbsp;&nbsp;The opportunities with short sale are marked on chart with grey color. As we can see, thanks to this option, additional investment portfolios have been created. The maximum ror and efficiency portfolio moved more to the up-right corner which means that due to this method we can create a portfolio with a higher rate of return but also with a higher risk. At first sight, we can say that the minimum risk portfolio stays in the same place but only precise values can confirm it. The portfolios at the bottom of the chart are not efficient because with the same risk we get lower profit so we shouldn't take them into account. 

#### <strong>The share of instruments with short sale for the portfolio with the lowest risk [%]</strong>

&nbsp;&nbsp;&nbsp;&nbsp; The chart below presents the share of particular investments in our portfolio with the lowest risk. We can notice that there are some changes compared to previous set without short sale. The share of instruments for this portfolio is respectively for investments in Gold - <strong>58%</strong> (58,7% previous), S&P500 - <strong>41%</strong> (40%) and Bitcoin - <strong>1%</strong> (1,3%).

```{r, fig.align='center', message = FALSE, fig.height = 5, fig.width = 5, error=FALSE}
min_ryz_short <- data.frame(
  instrument = c("Bitcoin", "S&P500", "Gold"),
  wagi = c(results_with_ss$w1[1], results_with_ss$w2[1], results_with_ss$w3[1])
)
min_ryz_short <- min_ryz_short %>%
  arrange(desc(instrument)) %>%
  mutate(lab.ypos = cumsum(wagi) - 0.5*wagi)

ggplot(min_ryz_short, aes(x="", y=wagi, fill = instrument)) +
  geom_bar(stat="identity", width=1) +
  coord_polar("y", start=0) + 
  scale_fill_manual(values=c("gray48", "bisque1", "cornsilk3"))+
  theme(plot.title = element_text(hjust = 1)) + 
  geom_text(aes(y = lab.ypos, label = wagi*100 ), color = "black") + 
  theme(legend.position = c(1.11, 0.9)) + 
  theme_minimal()
```

&nbsp;&nbsp;&nbsp;&nbsp; As previously below are presented values for particular portfolios. There are not many changes but we can notice that maximum efficiency and ror portfolio changed their structure, thanks to we can gain more profit from this portfolio but we have to be aware that this is connected with increased risk. 

```{r,  message = FALSE, echo=FALSE, error=FALSE}
kable(results_with_ss)
```

# Change of optimal portfolios over time

&nbsp;&nbsp;&nbsp;&nbsp; Now we will study how the optimal portfolio has changed over time. It will provide us with information on how the portfolio has changed due to the risk in recent years and the volatility of our portfolio. Thanks to that we can find out if our wallet is stable over time and whether we have to change its structure frequently

```{r, message = FALSE, warning=FALSE, error=FALSE}
returns_test_trial <- data.frame(dates = bitcoin_monthly$date,
                                    Bitcoin = bitcoin1,
                                    SP500 = sp5001,
                                    Gold = gold1) 

dates <- seq(as.Date("2014-03-17"), length=87, by="months")
returns_test_trial.xts <- xts(x = returns_test_trial[,-1], order.by = dates)
returns <- data.frame(dates = bitcoin_monthly$date,
                                    Bitcoin = bitcoin1,
                                    SP500 = sp5001,
                                    Gold = gold1) 

returns.xts <- xts(x = returns[,-1], order.by = as.Date(returns[,1]))

names.inw <- colnames(x = returns.xts)
pspec <-portfolio.spec(assets = names.inw)

pspec <- add.constraint(portfolio = pspec, type = "full_investment")
pspec <- add.constraint(portfolio = pspec, type = "long_only")


pspec <-add.objective(portfolio = pspec, type = "risk", name = "StdDev")
pmr <- optimize.portfolio(R=returns.xts,portfolio = pspec, optimize_method = "ROI")
pmr_rebal <- optimize.portfolio.rebalancing(R = returns.xts, portfolio = pspec, opimize_method = "ROI", rebalance_on = "years", training_period = 24, rolling_window = 24)
```

&nbsp;&nbsp;&nbsp;&nbsp; Here we can see that over this period there hasn't been big changes in our portfolio. The main thing was the ratio between gold and S&P500. In 2020 during crises Coronavirus Gold was the main component of our investment portfolio it's proof that this investemnt is highly safe and we should maintain this in our portfolio if we want to reduce risk. Bitcoin had a very small share in the structure each year which leads to the conclusion that an investor wishing to reduce risk should strongly reduce the share of this investment in the portfolio. An investor choosing this strategy, should be prepared for numerous changes in the portfolio and constant monitoring of its performance. 
```{r,  message = FALSE, fig.align='center', error=FALSE}
chart.Weights(pmr_rebal, colors = c("gray48", "bisque1", "cornsilk3", "cornsilk1"), 
              las = 1, ylab = "Portfolio weights", main = NULL)
```

&nbsp;&nbsp;&nbsp;&nbsp;Below we can see the cumulative return based on the above optimal portfolios. We can notice that it can bring stable profit without greater deviations so it is, therefore, an investment targeted at a risk avoidance group of investors. What's more we gain the knowledge about how our portfolio performs against downtrend and how long it can recover. In this case, our investment does not experience sudden sharp drops.

```{r,  message = FALSE, fig.align='center', error=FALSE}
rr <- Return.portfolio(R = returns.xts, weights = extractWeights(pmr))
charts.PerformanceSummary(rr)
```

# Summary

&nbsp;&nbsp;&nbsp;&nbsp; The purpose of the work was to investigate if Bitcoin is a good opportunity to invest. At the beginning of the project I analyzed performance of all investments. The main conclusions that could be drawn are that Bitcoin is the most profitable from all three investing but also it can bring the highest risk. But this was only charts analyses which doesn't bring much information but only familiar with performance of all investments. <br />
&nbsp;&nbsp;&nbsp;&nbsp; Then I prepared an initial analysis that could bring more information about the performance of Bitcoin compared to other investments. <strong> At this moment we can say that Bitcoin is the most variable investment which was also the most profitable.</strong> The distribution of Bitcoin returns was significantly different from the traditional investments which also confirmed that Bitcoin is very risky. So after initial analysys I can recomend that composition of our portfolio should depend on our investor profile. If we want to gain higher return we should invest more money in Bitcoin but we should remember that it can also bring huge loses. On the other hand, if we want to reduce risk we should chose between Gold and S&P500. <br />
&nbsp;&nbsp;&nbsp;&nbsp; <strong> If we are looking for an investment that can hedge our money against inflation we should choose S&P500.</strong> Bitcoin is not a good way to protect our funds because due to fact that it's very risky and has the lowest correlation with CPI from all three investments.  <br />
&nbsp;&nbsp;&nbsp;&nbsp; The last and the most important part of my analysis was building a portfolio with all given investments. It provided us information about all possible portfolios in which we can invest, thanks to we could specify investment sets which are strongly suited to our needs. We listed mainly 3 portfolios:  maximum return, minimum risk, and maximum efficiency. The first set is of course one element portfolio because we focus on investments that can bring the highest profit (Bitcoin). It is a highly risky strategy that can lead to huge losses. What I focus on the most is looking for the portfolio which will lower volatility in the fund. <strong> I built an investment set which is composed of: 58.7% of Gold, 40% of S&P500, and 1.3% of Bitcoin. It is a portfolio that is the safest and can bring 0.72% of return which is a very low value. </strong> I noticed that this is also lower than the risk-free rate so investing in this portfolio is practically pointless. If we want to lower volatile we should invest in 10-year US government bonds which are risk-free and also can bring higher profit from investing. But if we want to choose between those investments we should decide to pick up the portfolio I presented above. Additionally, I also checked how has been changed the structure of the minimum risk portfolio over time, we can notice relations were quite stable within this time but the weights changed frequently so we should be prepared for numerous changes in the portfolio and constant monitoring of its performance. <br />
<br />




## References <br />
  * <https://en.wikipedia.org/wiki/S%26P_500> <br />
  * <https://ycharts.com/indicators/us_10year_government_bond_interest_rate> <br />
  * <https://en.wikipedia.org/wiki/Short_(finance)> <br />
  * <https://coincentral.com/11-bitcoin-memes/>  <br />
  * The Journal of Finance, Markowitz H., 1952, p. 77-91 <br />
  
  
  
  
 