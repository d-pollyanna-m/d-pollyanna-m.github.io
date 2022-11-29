---
header: 
  teaser: "/assets/cyclistic/figs/fig3.png"

title: "RPersonalFinance"

toc: true
toc_sticky: true

tags:
  - R
  - ggplot
  - dplyr
categories:
  - portfolio
---

R, dplyr, ggplot: This program helps the user manage their personal finances across many accounts by taking a set of simple budget data and predicting account balances into the future. This set of custom R functions is designed to automatically process an arbitrary number of bank accounts and visualize their future balances and transactions, based on a set of expected bills or transfers.

<!--more-->


# Getting Started

## Libraries and Data Files

Make sure to include the following libraries. You can explore them at the [GitHub repo](https://github.com/kdmartin518/RPersonalFinance).


{% highlight r %}
library(tidyverse)
library(tidyr)
library(lubridate) 
library(ggplot2)

source('source/constants.R')
source('source/importer_functions.R')
source('source/helper_functions.R')
source('source/user_functions.R')
{% endhighlight %}

Data is stored in three different CSV files under [/source](https://github.com/kdmartin518/RPersonalFinance/tree/master/data). Each is useful for different things.

> -   *bills.csv* keeps track of regular expenses for each bank account.
>
> -   *transfers.csv* keeps track of transfers occurring between bank accounts.
>
>     -   *These are recorded by the day of month or day of week along with the total monthly amount.*
>
> -   *special.csv* keeps track of one-time transactions that occur on a specific date.
>
>     -   *This is useful to input a starting balance for a given month, or noting an annually-recurring bill.*

# Creating the budget

My friends Alice and Bob wrote down their combined finances in a set of spreadsheeets. They wrote their names next to their bank accounts and a list of bills and transfers that they wanted to plan for.

Alice converted the spreadsheets into CSV files. Let's import them.


{% highlight r %}
transaction_sheet <- import_data('data/bills.csv','data/transfers.csv')
special_sheet <- import_special('data/special.csv')
{% endhighlight %}

The budgets are in R in data frames. Now we can do whatever we want to analyse them.

We can review one bank account account at a time by filtering:


{% highlight r %}
filter(transaction_sheet,bank_account=="Alice Primary")
{% endhighlight %}



{% highlight text %}
##     bank_account accountor day                name monthly_amount
## 1  Alice Primary     Alice   5       Credit Card A            -77
## 2  Alice Primary     Alice  11       Credit Card B            -30
## 3  Alice Primary     Alice  20       Credit Card C            -30
## 4  Alice Primary     Alice  22            Car Loan           -280
## 5  Alice Primary     Alice  29    Credit Card Bill           -682
## 6  Alice Primary     Alice  22  Joint Contribution           -650
## 7  Alice Primary     Alice   7  Joint Contribution           -650
## 8  Alice Primary     Alice   1 Coffee and Shopping           -500
## 9  Alice Primary     Alice  15            Paycheck           1600
## 10 Alice Primary     Alice  30            Paycheck           1600
{% endhighlight %}


{% highlight r %}
filter(transaction_sheet,bank_account=="Bob Primary")
{% endhighlight %}



{% highlight text %}
##   bank_account accountor     day               name monthly_amount
## 1  Bob Primary       Bob  Friday   Credit Card Bill           -280
## 2  Bob Primary       Bob Tuesday Joint Contribution           -900
## 3  Bob Primary       Bob  Monday           Paycheck           1200
{% endhighlight %}

Note that Bob's budgets are scheduled by days of the week. Alice's in comparison are scheduled by day of month. This will be important in a moment.

# Making a balance sheet

Now that we've imported our budgets, let's turn them into a a set of balance sheets. A balance sheet looks like your bank statement. Here you'll see the sum of transactions on each day of the month with a running tally at the end of the day on the side.

## create_balance_sheet()

Given a transaction sheet like the one we created earlier, create_balance_sheet() will create a set of balance sheets. It will automatically create one sheet for each bank account listed in the transaction sheet. It will create a balance sheet for as large a date range as your specify.


{% highlight r %}
from_date <- mdy('11-01-2022')
to_date <- mdy('12-1-2022')
balance_sheet <- create_balance_sheet(transaction_sheet,from_date,to_date)
{% endhighlight %}

As it is, *balance_sheet* contains a couple hundred rows: A month of days from 11/1 ro 12/1 for each bank account in the transaction sheet. Let's filter it out to a couple of individual accounts.


{% highlight r %}
alice <- filter(balance_sheet,bank_account=="Alice Primary")
bob <- filter(balance_sheet,bank_account=="Bob Primary")

head(alice,10)
{% endhighlight %}



{% highlight text %}
## # A tibble: 10 × 4
## # Groups:   bank_account [1]
##    date       amount bank_account  balance
##    <date>      <dbl> <chr>           <dbl>
##  1 2022-11-01    929 Alice Primary     929
##  2 2022-11-02      0 Alice Primary     929
##  3 2022-11-03      0 Alice Primary     929
##  4 2022-11-04      0 Alice Primary     929
##  5 2022-11-05    -77 Alice Primary     852
##  6 2022-11-06      0 Alice Primary     852
##  7 2022-11-07   -650 Alice Primary     202
##  8 2022-11-08      0 Alice Primary     202
##  9 2022-11-09      0 Alice Primary     202
## 10 2022-11-10      0 Alice Primary     202
{% endhighlight %}

Here, each row represents the sum of all transactions on a given day, for Alice's account.

## Weekday transactions

Earlier I pointed out that Bob's account has his transactions listed as occurring on days of the week. Here's what his balance sheet looks like:


{% highlight r %}
bob
{% endhighlight %}



{% highlight text %}
## # A tibble: 31 × 4
## # Groups:   bank_account [1]
##    date       amount bank_account balance
##    <date>      <dbl> <chr>          <dbl>
##  1 2022-11-01   -107 Bob Primary     -107
##  2 2022-11-02      0 Bob Primary     -107
##  3 2022-11-03      0 Bob Primary     -107
##  4 2022-11-04    -70 Bob Primary     -177
##  5 2022-11-05      0 Bob Primary     -177
##  6 2022-11-06      0 Bob Primary     -177
##  7 2022-11-07    300 Bob Primary      123
##  8 2022-11-08   -225 Bob Primary     -102
##  9 2022-11-09      0 Bob Primary     -102
## 10 2022-11-10      0 Bob Primary     -102
## # … with 21 more rows
{% endhighlight %}

Take a close look at Bob's account. Notice how there's a few repeating transactions, such as the -70 on 11/11, 11/18, 11/25, and so on. Notice also that 70 = 280/4, and 280 is the monthly value of Bob's Credit Card Payment which recurs on Fridays.

If a bill is listed as occurring on a named weekday, it will be automatically split up onto all of that weekday in the month, at 1/4 of the monthly value.

# Predicting overdrafts

Bob is worried that there won't be enough in the joint bank account to cover a bill, but he isn't sure how much will be in the acccount by the day it is due.

## predict_max_overdraft(balance_sheet)

This would be Bob's fortune. It will find the future date of the most negative balance for each bank account in the sheet provided. See for yourself:


{% highlight r %}
joint <- filter(balance_sheet,bank_account=="Joint Primary")
predict_max_overdraft(joint)
{% endhighlight %}



{% highlight text %}
##         date balance  bank_account
## 1 2022-11-06    -292 Joint Primary
{% endhighlight %}

Keep in mind this doesn't tell us when the first day the account will be overdrawn: But it does tell us that we need to move \$517 in total to avoid a negative balance.

# Visualizing

Alice and bob would like to see a visualization of their future finances. It would help them to feel more confident that they can see it for themselves at a glance. Let's make an easy way for them to see trends in their balance and expect upcoming bills.

## draw_balance_sheet()

Enter this function. It will automatically display a combination chart for each bank account in the provided balance sheet.
This will look messy if we import the whole thing all at once. Let's filter.




{% highlight r %}
joint <- filter(balance_sheet,
                bank_account=='Joint Primary'|bank_account=='Alice Primary',
                between(date,
                  mdy('11-01-22'),
                  mdy('11-30-22'))
                )
source('source/user_functions.R')
draw_balance_sheet(joint)
{% endhighlight %}

![plot of chunk unnamed-chunk-8](/assets/Rfig/unnamed-chunk-8-1.svg)

By the way, we can use this to analyze that overdraft we found out about earlier. Let's zoom in to just that week.


{% highlight r %}
joint <- filter(balance_sheet,
                bank_account=='Joint Primary',
                between(date,mdy('11-01-22'),mdy('11-8-22'))
                )
draw_balance_sheet(joint)
{% endhighlight %}

![plot of chunk unnamed-chunk-9](/assets/Rfig/unnamed-chunk-9-1.svg)

So here we see what our account looks like around the date of the most negative balance. Looks like there's a large deduction that day. We can get more details about that time range this way:


{% highlight r %}
overdraft <- transaction_sheet %>% 
              filter(
                     bank_account=="Joint Primary",
                     between(day,1,7)
                    ) %>%
              transmute(day,name,monthly_amount)
overdraft
{% endhighlight %}



{% highlight text %}
##   day               name monthly_amount
## 1   1              Fetch         -61.19
## 2   4              Rent        -1720.95
## 3   6   Electric Company        -100.00
## 4   1       Pet Supplies        -208.00
## 5   7 Joint Contribution         650.00
{% endhighlight %}

Yep, so it looks like we are a little short for rent this month. But we know by exactly how much, so as long as we move \$518 into Joint Primary by 11/4, we should be able to cover it!
