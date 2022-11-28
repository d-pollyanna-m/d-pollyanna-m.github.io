---
header: 
  teaser: "/assets/cyclistic/figs/fig3.png"

title: "RPersonalFinance"

tags:
  - R
  - ggplot
  - dplyr
categories:
  - portfolio
---

R, dplyr, ggplot. This set of custom R functions is designed to automatically process an arbitrary number of bank accounts and visualize their future balances and transactions, based on a set of expected bills or transfers.

<!--more-->

If you are interested in the development side, I welcome you to dig through the code in the [GitHub repository here.](https://github.com/kdmartin518/RPersonalFinance)

## Premise

This program helps the user manage their personal finances across many accounts by taking a set of simple budget data and predicting account balances into the future.

## Format

Data is stored in three different CSVs, each for a different purpose:

*bills.csv* keeps track of regular expenses for each bank account. *transfers.csv* keeps track of transfers occurring between bank accounts.

These are recorded by the day of month or day of week along with the total monthly amount.

*special.csv* keeps track of one-time transactions that occur on a specific date. This is useful to input a starting balance for a given month, or noting an annually-recurring bill.

# Getting started

Let's start by importing the data and seeing what it looks like.



{% highlight r %}
transaction_sheet <- import_data('data/bills.csv','data/transfers.csv')
special_sheet <- import_special('data/special.csv')
{% endhighlight %}

You're welcome to investigate these data frames yourself to see what's inside. They are more or less the same as the CSVs, which is to say the budget lines for a bunch of bank accounts. We can review one account at a time by filtering:


{% highlight r %}
filter(transaction_sheet,bank_account=="Alice Primary")
{% endhighlight %}



{% highlight text %}
## [1] bank_account   accountor      day            name           monthly_amount
## <0 rows> (or 0-length row.names)
{% endhighlight %}

{% highlight r %}
filter(transaction_sheet,bank_account=="Bob Primary")
{% endhighlight %}



{% highlight text %}
## [1] bank_account   accountor      day            name           monthly_amount
## <0 rows> (or 0-length row.names)
{% endhighlight %}

Note that Bob's budgets are scheduled by days of the week. Alice's in comparison are scheduled by day of month. This will be important in a moment.

# Balance Sheets

Now that we've imported our budgets, let's turn them into a a set of balance sheets. A balance sheet looks like your bank statement. Here you'll see the sum of transactions on each day of the month with a running tally at the end of the day on the side. 

## create_balance_sheet()

Given a transaction sheet like the one we created earlier, create_balance_sheet() will create a set of balance sheets. It will automatically create one sheet for each bank account listed in the transaction sheet. It will create a balance sheet for as large a date range as your specify.

*Note: dates are given in Lubridate format.*

{% highlight r %}
from_date <- mdy('11-01-2022')
to_date <- mdy('12-1-2022')

balance_sheet <- create_balance_sheet(transaction_sheet,from_date,to_date)
glimpse(balance_sheet)
{% endhighlight %}



{% highlight text %}
## Rows: 217
## Columns: 4
## $ date         <date> 2022-11-01, 2022-11-02, 2022-11-03, 2022-11-04, 2022-11-05, 2022-11-06, 2022-11-07, 2022-11-08, 2…
## $ amount       <dbl> -150, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 682, 0, -15…
## $ bank_account <chr> "Alysia Credit", "Alysia Credit", "Alysia Credit", "Alysia Credit", "Alysia Credit", "Alysia Credi…
## $ balance      <dbl> -150, -150, -150, -150, -150, -150, -150, -150, -150, -150, -150, -150, -150, -150, -150, -150, -1…
{% endhighlight %}
There are 251 rows. That's 8 accounts * 31 days each. Let's filter out to the accounts we care about right now.


{% highlight r %}
alice <- filter(balance_sheet,bank_account=="Alice Primary")
alice
{% endhighlight %}



{% highlight text %}
## # A tibble: 0 × 4
## # … with 4 variables: date <date>, amount <dbl>, bank_account <chr>, balance <dbl>
{% endhighlight %}


{% highlight r %}
bob <- filter(balance_sheet,bank_account=="Bob Primary")
bob
{% endhighlight %}



{% highlight text %}
## # A tibble: 0 × 4
## # … with 4 variables: date <date>, amount <dbl>, bank_account <chr>, balance <dbl>
{% endhighlight %}

## Weekday transactions

Take a close look at Bob's account. Notice how there's a few repeating transactions, such as the -70 on 11/11, 11/18, 11/25, and so on. Notice also that 70 = 280/4, and 280 is the monthly value of Bob's Credit Card Payment which recurs on Fridays.

If a bill is listed as occurring on a named weekday, it will be automatically split up onto all of that weekday in the month, at 1/4 of the monthly value. 

# Predicting overdrafts

## predict_max_overdraft(balance_sheet)

This function is very useful. It will find the date and value of the balance furthest below 0 for each account in the sheet provided. See for yourself:


{% highlight r %}
joint <- filter(balance_sheet,bank_account=="Joint Primary")
predict_max_overdraft(joint)
{% endhighlight %}



{% highlight text %}
##         date  balance  bank_account
## 1 2022-12-01 -1728.55 Joint Primary
{% endhighlight %}

Keep in mind this doesn't tell us when the first day the account will be overdrawn: But it does tell us that we need to move $517 in total to avoid a negative balance.

# Visualizing

## draw_balance_sheet()

Finally, let's visualize the data.

By default this function displays transactions and overall balance in a chart, separated into different color groups by bank account. It's a combination chart, with a line part representing the running balance and bars representing total transactions on a given day. 

It will display as many accounts for as wide a range of dates as you supply it, which can get messy fast.


{% highlight r %}
draw_balance_sheet(balance_sheet)
{% endhighlight %}

![plot of chunk unnamed-chunk-7](/assets/Rfig/unnamed-chunk-7-1.svg)

So we might want to filter it down a little. 


{% highlight r %}
joint <- filter(balance_sheet,
                bank_account=='Joint Primary',
                between(date,mdy('11-01-22'),mdy('11-30-22'))
                )
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
##   day   name monthly_amount
## 1   1  Fetch         -61.19
## 2   4  Rent        -1220.95
## 3   5 Avista         -75.26
{% endhighlight %}

Yep, so it looks like we are a little short for rent this month. But we know by exactly how much, so as long as we move $518 into Joint Primary by 11/4, we should be able to cover it!
