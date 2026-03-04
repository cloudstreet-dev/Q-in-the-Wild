# R and Q: Statistical Power Meets Time-Series Performance

The R-Q relationship is one of complementary strengths that coexist awkwardly. R is genuinely excellent at statistics, visualization, and the kind of exploratory analysis where you want twelve different regression models and a scatterplot matrix in thirty lines. Q is genuinely excellent at storing and querying large time-series datasets efficiently. Together, they cover a lot of ground. Separately, they each have a large blind spot.

The practical question is: how do you get R talking to Q without it being painful?

## The Options

**`rkdb`**: The most direct path. R package that connects to kdb+ via IPC. Wraps the C API. The one you want.

**`rqpython` / qPython bridge**: Use Python as a middleman. Q → Python → R. Occasionally done in organizations that already have Python infrastructure. More moving parts than necessary if you just want R talking to Q.

**Flat files / HDF5**: Export from Q, import in R. Works. Terrible for anything real-time. Mentioned only to be dismissed.

We'll focus on `rkdb`. It's the direct, performant option and the one that's actually maintained.

## rkdb: Setup and Connection

```r
# Install from CRAN (or GitHub for latest)
install.packages("rkdb")
# or
devtools::install_github("KxSystems/rkdb")
```

Connecting to a kdb+ process:

```r
library(rkdb)

# Open a connection
h <- open_connection("localhost", 5001)

# Execute a q expression
result <- execute(h, "til 10")
print(result)
# [1] 0 1 2 3 4 5 6 7 8 9

# Close when done
close_connection(h)
```

Use a tryCatch block to ensure the connection is closed:

```r
library(rkdb)

with_qconnection <- function(host, port, expr) {
  h <- open_connection(host, port)
  on.exit(close_connection(h))
  expr(h)
}

# Usage
result <- with_qconnection("localhost", 5001, function(h) {
  execute(h, "select from trade where date=2024.01.15")
})
```

## Type Mapping

Q types map to R types reasonably cleanly, with a few exceptions that matter:

| Q type | R type |
|--------|--------|
| boolean | logical |
| byte | raw |
| short | integer |
| int | integer |
| long | numeric (R has no 64-bit int) |
| real | numeric |
| float | numeric |
| char | character |
| symbol | character |
| timestamp | POSIXct |
| date | Date |
| time | difftime |
| list | list |
| table | data.frame |
| dictionary | named list |

The important one: **Q longs become R numerics**. R's integer type is 32-bit. Q's long is 64-bit. Values up to 2^31 survive the conversion without issue; larger values lose precision silently. If you're working with large integer IDs or nanosecond timestamps as integers, be aware of this.

```r
h <- open_connection("localhost", 5001)

# Q table → R data.frame (this is the good news)
df <- execute(h, "select date, sym, price, size from trade where date=2024.01.15")
class(df)          # "data.frame"
str(df)
# 'data.frame':   1000 obs. of  4 variables:
#  $ date : Date, format: "2024-01-15" ...
#  $ sym  : chr  "AAPL" "AAPL" "MSFT" ...
#  $ price: num  185.5 185.6 375.2 ...
#  $ size : num  100 200 150 ...

# Symbol columns come back as character vectors
class(df$sym)      # "character"

# Timestamp handling
ts_data <- execute(h, "select time from trade where date=2024.01.15, i<5")
class(ts_data$time)  # "POSIXct" "POSIXt"
format(ts_data$time[1], "%H:%M:%OS6")  # "09:30:00.123456"

close_connection(h)
```

## The Moment: Doing Real Analysis

Here's the pattern that makes the R-Q combination click — pull structured data from Q, do something R is uniquely good at, and optionally push results back.

```r
library(rkdb)
library(dplyr)
library(ggplot2)
library(lubridate)

h <- open_connection("localhost", 5001)
on.exit(close_connection(h))

# Pull a month of OHLC data from Q
ohlc <- execute(h, "
  select
    date,
    open: first price,
    high: max price,
    low: min price,
    close: last price,
    volume: sum size
  from trade
  where
    date within 2024.01.01 2024.01.31,
    sym=`AAPL
  by date
")

# That Q query runs in milliseconds on a properly splayed table.
# Now R takes over.

# Compute technical indicators in R
ohlc <- ohlc %>%
  arrange(date) %>%
  mutate(
    # Simple moving averages
    sma_5  = zoo::rollmean(close, 5,  fill = NA, align = "right"),
    sma_20 = zoo::rollmean(close, 20, fill = NA, align = "right"),

    # Daily return
    ret = (close / lag(close)) - 1,

    # Bollinger bands (20-day)
    bb_mid  = sma_20,
    bb_sd   = zoo::rollapply(close, 20, sd, fill = NA, align = "right"),
    bb_upper = bb_mid + 2 * bb_sd,
    bb_lower = bb_mid - 2 * bb_sd,

    # RSI (14-day)
    gain = pmax(ret, 0),
    loss = pmax(-ret, 0),
    avg_gain = zoo::rollmean(gain, 14, fill = NA, align = "right"),
    avg_loss = zoo::rollmean(loss, 14, fill = NA, align = "right"),
    rs = avg_gain / avg_loss,
    rsi = 100 - (100 / (1 + rs))
  )

print(tail(ohlc %>% select(date, close, sma_5, sma_20, rsi), 5))

# Visualize
ggplot(ohlc %>% filter(!is.na(sma_20)), aes(x = date)) +
  geom_ribbon(aes(ymin = bb_lower, ymax = bb_upper), fill = "steelblue", alpha = 0.2) +
  geom_line(aes(y = close), color = "black") +
  geom_line(aes(y = sma_20), color = "red", linetype = "dashed") +
  labs(title = "AAPL: Price with Bollinger Bands (Jan 2024)",
       y = "Price", x = NULL) +
  theme_minimal()
```

## Pushing Results Back to Q

Results computed in R can be inserted back into Q:

```r
library(rkdb)

h <- open_connection("localhost", 5001)
on.exit(close_connection(h))

# Compute something in R
set.seed(42)
simulated_paths <- data.frame(
  t = 0:252,
  path1 = cumsum(c(0, rnorm(252, 0.0003, 0.015))),
  path2 = cumsum(c(0, rnorm(252, 0.0003, 0.015))),
  path3 = cumsum(c(0, rnorm(252, 0.0003, 0.015)))
)

# Push back to Q
# Note: execute() sends data; the second argument to `set` is the name
execute(h, "set", "sim_paths", simulated_paths)

# Verify
n <- execute(h, "count sim_paths")
cat("Rows in Q:", n, "\n")  # 253
```

The type mapping on the way *in* can be fiddly. Data frames map to Q tables; R numerics map to Q floats; R integers map to Q ints. Factors will cause you grief — convert them to character first.

## Statistical Patterns That Work Well

### Cross-Sectional Factor Analysis

```r
library(rkdb)
library(dplyr)

h <- open_connection("localhost", 5001)
on.exit(close_connection(h))

# Pull end-of-day prices for a universe of symbols
eod <- execute(h, "
  select
    date,
    sym,
    close: last price,
    volume: sum size
  from trade
  where
    date within 2023.01.01 2024.01.01,
    sym in `AAPL`MSFT`GOOG`AMZN`META`NVDA`TSLA`JPM`BAC`GS
  by date, sym
")

# Compute factor exposures
eod <- eod %>%
  group_by(sym) %>%
  arrange(date) %>%
  mutate(
    ret_1d   = close / lag(close) - 1,
    ret_5d   = close / lag(close, 5) - 1,
    ret_21d  = close / lag(close, 21) - 1,
    vol_21d  = zoo::rollapply(ret_1d, 21, sd, fill = NA, align = "right"),
    vol_log  = log(volume),
    momentum = ret_21d / vol_21d  # risk-adjusted momentum
  ) %>%
  ungroup()

# Cross-sectional z-score each factor
eod <- eod %>%
  group_by(date) %>%
  mutate(
    z_momentum = scale(momentum)[,1],
    z_vol_log  = scale(vol_log)[,1]
  ) %>%
  ungroup() %>%
  filter(!is.na(z_momentum))

# IC (Information Coefficient) of momentum factor
ic_by_date <- eod %>%
  group_by(date) %>%
  summarise(
    ic = cor(z_momentum, lead(ret_5d), use = "complete.obs", method = "spearman"),
    .groups = "drop"
  ) %>%
  filter(!is.na(ic))

cat("Mean IC:", round(mean(ic_by_date$ic, na.rm = TRUE), 4), "\n")
cat("IC IR:", round(mean(ic_by_date$ic, na.rm = TRUE) /
                    sd(ic_by_date$ic, na.rm = TRUE), 4), "\n")
```

### Cointegration Analysis

Q's tick data is excellent for cointegration testing — pairs trading, spread analysis, that sort of thing:

```r
library(rkdb)
library(urca)  # for Johansen cointegration test

h <- open_connection("localhost", 5001)
on.exit(close_connection(h))

# Pull synchronized mid prices for two instruments
prices <- execute(h, "
  aj[`time`sym;
    select time, sym:`SPY, mid:0.5*(bid+ask) from quotes where date=2024.01.15, sym=`SPY;
    select time, sym:`IVV, mid:0.5*(bid+ask) from quotes where date=2024.01.15, sym=`IVV
  ]
")

spy <- prices$mid[prices$sym == "SPY"]
ivv <- prices$mid[prices$sym == "IVV"]

# Engle-Granger cointegration test
coint_reg <- lm(spy ~ ivv)
cat("Cointegration regression R²:", summary(coint_reg)$r.squared, "\n")

# ADF test on residuals
adf_result <- ur.df(residuals(coint_reg), type = "drift", lags = 5)
cat("ADF test statistic:", adf_result@teststat[1], "\n")
cat("5% critical value:", adf_result@cval[1, 2], "\n")

# Spread (basis)
spread <- spy - coef(coint_reg)[2] * ivv - coef(coint_reg)[1]
cat("Spread mean:", round(mean(spread), 6), "\n")
cat("Spread SD:", round(sd(spread), 6), "\n")
cat("Current z-score:", round(tail(spread, 1) / sd(spread), 4), "\n")
```

## Parameterized Queries

String interpolation for Q queries is fragile. Use the list-form execute when you need parameters:

```r
# Fragile: SQL injection-ish problems, quoting issues
sym <- "AAPL"
bad_query <- paste0("select from trade where sym=`", sym)
execute(h, bad_query)  # works but ugly

# Better: use execute with arguments
# rkdb supports passing arguments after the function call
result <- execute(h, "{[s;d] select from trade where sym=s, date=d}",
                  as.name("AAPL"),        # Q symbol
                  as.Date("2024-01-15"))  # Q date

# The as.name() trick converts R character to Q symbol
# as.Date() converts R Date to Q date
```

## Scheduling and Automation

For regular R → Q data pulls (end-of-day analytics, overnight batch jobs):

```r
library(rkdb)

run_eod_analytics <- function(date_str) {
  h <- open_connection("localhost", 5001)
  on.exit(close_connection(h))

  tryCatch({
    # Pull data
    data <- execute(h, paste0(
      "select from eod_summary where date=", date_str
    ))

    if (nrow(data) == 0) {
      cat("No data for", date_str, "\n")
      return(invisible(NULL))
    }

    # Run analytics
    results <- compute_factor_scores(data)

    # Write back
    execute(h, "set", "r_factor_scores", results)
    execute(h, paste0("r_factor_scores: update date:", date_str,
                      " from r_factor_scores"))

    # Append to persistent table
    execute(h, "`factor_scores upsert r_factor_scores")

    cat("EOD analytics complete for", date_str, "\n")
  }, error = function(e) {
    cat("Error in EOD analytics:", conditionMessage(e), "\n")
  })
}

# Run for today
run_eod_analytics(format(Sys.Date(), "%Y.%m.%d"))
```

## Performance Notes

Q will outperform R on any query over large datasets. R's `data.frame` operations get slow past a few million rows; Q's columnar storage and query engine handle billions of rows comfortably. The right division of labor is:

- **Filter and aggregate in Q** (date ranges, symbol selection, OHLC, VWAP)
- **Analyze in R** (regressions, factor models, rolling stats, visualization)
- **Write results back to Q** (persist computed signals for the trading system)

Don't pull raw tick data into R and filter it there. Pull the Q-computed aggregates. Your laptop will thank you.

A query like this runs in milliseconds in Q; the equivalent in R against a data.frame with 100M rows does not:

```q
/ Q: ~10ms on 100M row trade table
select vwap: size wavg price by sym from trade where date=2024.01.15
```

```r
# R equivalent: slow
# trade_df %>% group_by(sym) %>% summarise(vwap = weighted.mean(price, size))
# Much better: let Q do it, just receive the result in R
```
