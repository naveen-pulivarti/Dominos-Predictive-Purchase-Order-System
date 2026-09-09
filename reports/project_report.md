# Domino's - Predictive Purchase Order System
## Project Report

**Author:** Sai Naveen Pulivarthi
**Domain:** Food Service Industry
**Metric:** Mean Absolute Percentage Error (MAPE)

---

## 1. Executive Summary

Domino's needs to order ingredients ahead of demand. Order too little and you get stockouts and
lost sales; order too much and fresh stock spoils. Both cost money.

This project builds a pipeline that takes a year of raw transaction data and produces a
purchase order for the coming week, measured in kilograms per ingredient.

**Headline results:**

| | |
|---|---|
| Best model | **SARIMA(1,0,2)(1,1,1,13)** |
| Forecast accuracy | **91.96%** (MAPE 0.0804) |
| Forecast week | 28 Dec 2015 - 3 Jan 2016 |
| Predicted demand | 1,085 pizzas across 91 pizza-size variants |
| Purchase order | 64 ingredients, 198.79 kg total |
| Largest line item | Chicken, 19.2 kg |

All 70 missing values across the two source files (66 in sales, 4 in ingredients) were
reconstructed rather than dropped, so all 48,620 sales records were retained.

---

## 2. Data

Two source files, downloaded from the project's original spreadsheets:

| Dataset | Rows | Columns |
|---|---|---|
| `pizza_sales.csv` | 48,620 | 12 |
| `Pizza_ingredients.csv` | 518 | 4 |

**Scale of the business, from the data:**

- 49,574 pizzas sold across 21,350 orders
- $817,860 total revenue
- 358 trading days in 2015 (the store was closed on 7 days)
- 32 distinct pizzas, sold in 91 name-and-size variants

---

## 3. Methodology

### 3.1 Data Cleaning

Both files ship with deliberate gaps. The approach was to **recover** each value from
relationships already present in the data, never to drop the row - every dropped row is a real
pizza sale, and deleting it biases demand downward.

| Column | Missing | Recovery method |
|---|---|---|
| `total_price` | 7 | Recomputed as `unit_price x quantity` |
| `pizza_category` | 23 | Lookup from `pizza_name_id` |
| `pizza_ingredients` | 13 | Lookup from `pizza_name` |
| `pizza_name` | 7 | Lookup from the ingredient string |
| `pizza_name_id` | 16 | Rebuilt from the `(pizza_name, pizza_size)` pair |
| `Items_Qty_In_Grams` | 4 | Mean gram-weight of that pizza's other ingredients |

**Outcome: 48,620 / 48,620 rows retained. Zero duplicates found.**

One detail mattered more than it first appears. `pizza_name_id` encodes *both* the recipe and
the size - `bbq_ckn_l` is a **large** Barbecue Chicken. It therefore cannot be recovered from
the pizza name alone; the `(name, size)` pair is required to identify it. Filling it from the
name only would have silently assigned the wrong size to 16 rows, and size is exactly what
drives ingredient quantities.

### 3.2 Date Parsing

`order_date` arrives in **two different formats in the same column**: `d/m/Y` for days 1-12 and
`d-m-Y` for days 13-31. This is a spreadsheet export artefact - the rows where day and month
are both <= 12 were reformatted, the ambiguous-free ones were not.

Both are day-first. A parser that tries each format in turn recovers **all 48,620 dates with
zero failures**, giving a clean series from 1 Jan to 31 Dec 2015.

Reading the column as month-first would have silently mangled every date in the first twelve
days of each month - the kind of error that produces a plausible-looking but wrong forecast.

### 3.3 Feature Engineering

- Calendar features: day of week, month, ISO week, year
- US public holiday flag
- Weekend flag, treated as the promotional period per the brief

### 3.4 Handling the Truncated Weeks

The purchase order covers one week, so sales were aggregated to weekly totals. This exposed a
boundary problem worth stating explicitly:

The data covers 1 Jan - 31 Dec 2015, and **neither end lands on a week boundary**. The first
bucket (starting Mon 29 Dec 2014) contains only 4 in-range days and totals 591 units. The last
(starting Mon 28 Dec 2015) also contains 4 days and totals 442. Against a full-week mean of
**952**, both look like severe demand collapses that never happened.

Both were dropped, leaving **51 complete weeks**.

The test used is *date-range coverage*, not the number of days that recorded a sale. That
distinction matters: several complete weeks contain a day with genuinely zero sales - the store
was shut on Christmas Day - and that is real demand information which must be kept. Counting
trading days would have wrongly discarded those weeks too.

### 3.5 Train / Test Split

The split is **chronological, never shuffled** - 80% train (40 weeks), 20% test (11 weeks). A
random split would let the model see future weeks while predicting past ones, inflating the
score with information it would never have in production.

---

## 4. Exploratory Findings

### 4.1 Demand is remarkably flat across the menu

| Pizza | Units |
|---|---|
| The Classic Deluxe Pizza | 2,453 |
| The Barbecue Chicken Pizza | 2,432 |
| The Hawaiian Pizza | 2,422 |
| The Pepperoni Pizza | 2,418 |
| The Thai Chicken Pizza | 2,371 |

The top five sit within 3.5% of each other. There is no runaway bestseller to build inventory
around - which means ingredient planning has to cover the whole menu rather than optimising for
a handful of stars. The clear outlier is at the bottom: **The Brie Carre Pizza at 490 units**,
almost exactly a fifth of the leaders.

### 4.2 Category and size mix

| Category | Units | | Size | Units |
|---|---|---|---|---|
| Classic | 14,888 | | L | 18,956 |
| Supreme | 11,987 | | M | 15,635 |
| Veggie | 11,649 | | S | 14,403 |
| Chicken | 11,050 | | XL | 552 |
| | | | XXL | 28 |

Large is the single most popular size, and the L/M/S split is fairly even. XL and XXL are
negligible. This size distribution is precisely why the forecast is built per
`pizza_name_id` rather than per pizza name.

### 4.3 The weekend assumption does not hold

The project brief suggests treating weekends as the promotional period. **The data contradicts
this.**

| Day | Avg daily units |
|---|---|
| Friday | 165 |
| Saturday | 144 |
| Thursday | 144 |
| Monday | 135 |
| Wednesday | 134 |
| Tuesday | 133 |
| **Sunday** | **116** |

Weekends average **130 units/day against 142 on weekdays**. Friday is the clear peak, and
**Sunday is the weakest day of the week** - the opposite of what a weekend-promotion assumption
predicts.

The weekend flag was retained for transparency, since the brief calls for it, but it should be
read as a *weak negative* signal rather than a promotional uplift. If a promotional calendar
were ever attached to this data, Friday is where it would show.

Holidays behave differently again: **169 units/day against 138 on ordinary days**, a genuine
uplift of around 22%.

### 4.4 Seasonality is mild

July is the strongest month (4,392 units) and October the weakest (3,883) - a spread of about
13%. There is no dramatic annual cycle, which is consistent with the models finding most of
their signal in week-to-week movement rather than long-range season.

---

## 5. Model Comparison

Five models, identical weekly data, identical held-out weeks, identical metric.

| Model | MAPE | Accuracy | Rank |
|---|---|---|---|
| **SARIMA** | **0.0804** | **91.96%** | 1 (Best) |
| ARIMA | 0.0853 | 91.47% | 2 |
| Regression | 0.0914 | 90.86% | 3 |
| Prophet | 0.0962 | 90.38% | 4 |
| LSTM | 0.0981 | 90.19% | 5 (Worst) |

**Why SARIMA won.** It was the only model given an explicit seasonal block, and the grid search
selected a 13-week seasonal period - roughly a quarter. With trend and seasonality separated,
it tracked the held-out weeks more closely than anything else.

**Why LSTM came last.** 40 training weeks is far too little data for a recurrent network. LSTMs
need thousands of sequences to justify their capacity; here the classical models encode the
right structural assumptions directly and win comfortably. This is a useful negative result -
model sophistication is not the same as model suitability.

**Why the regression baseline is respectable.** A plain linear model on calendar features lands
within 1.1 percentage points of SARIMA. Given the mild seasonality found in the EDA, most of
the predictable signal is simple. Any additional complexity has to justify itself against this
baseline.

The spread across all five is under 1.8 percentage points, which is itself informative: the
series is well-behaved and no approach fails badly.

One caveat on reproducibility: the four classical models are deterministic and reproduce
exactly on a re-run. The LSTM is not - despite fixed NumPy and TensorFlow seeds, GPU/CPU thread
scheduling makes its MAPE drift by a few thousandths between runs. It stays last either way,
but the exact figure should be read as approximate in a way the others are not.

---

## 6. Purchase Order Generation

### 6.1 Method

1. The winning model is **refitted on all 51 weeks**. The train/test split's job was to *choose*
   the model; once chosen, it should learn from every week available - including the most recent
   ones, which matter most for predicting next week.
2. A separate weekly series is built for each of the **91 `pizza_name_id` variants** and each is
   forecast one week ahead. All 91 fitted successfully; none needed the fallback path.
3. Forecasts are rounded to whole pizzas and floored at zero.
4. Each pizza's forecast is multiplied through its recipe:

```
required grams  =  SUM over pizza types ( grams per pizza  x  forecast units )
```

Forecasting at variant level rather than by pizza name is what makes the totals correct. A large
pizza uses more of every ingredient than a small one, so collapsing "Barbecue Chicken" into a
single number would discard the size mix that determines the actual ingredient bill.

### 6.2 Result

**Week of 28 Dec 2015 - 3 Jan 2016: 1,085 pizzas, 64 ingredients, 198.79 kg.**

| # | Ingredient | Kg |
|---|---|---|
| 1 | Chicken | 19.20 |
| 2 | Red Onions | 19.00 |
| 3 | Capocollo | 15.95 |
| 4 | Bacon | 13.30 |
| 5 | Tomatoes | 12.56 |
| 6 | Pepperoni | 10.77 |
| 7 | Mushrooms | 8.66 |
| 8 | Spinach | 6.74 |
| 9 | Garlic | 6.62 |
| 10 | Corn | 5.85 |

The full 64-line order is in `outputs/purchase_order_next_week.csv`.

A cross-check worth recording: forecasting the *total* series directly gives a figure close to
the sum of the 91 individual forecasts. Two independent routes to a similar number is a good
sign the bottom-up build is not drifting.

---

## 7. Business Implications

**Procurement becomes a decision, not a guess.** The output is not a chart - it is a list of
64 ingredients with kilogram quantities that a manager can act on directly.

**Waste reduction is where the money is.** Pizza toppings are perishable. At roughly 92%
forecast accuracy, over-ordering shrinks substantially against ordering by intuition or by last
week's figures.

**Ingredient concentration simplifies supplier negotiation.** The top 10 ingredients account for
a large share of total weight. Those are the lines where volume commitments and supplier terms
matter most.

**Friday, not the weekend, is the operational peak.** Staffing and prep should follow Friday and
the Thursday/Saturday shoulder, not a generic weekend assumption. Sunday is the quietest day and
is being over-resourced if the current schedule treats it as a weekend peak.

---

## 8. Limitations

Stated plainly, because they bound how far these numbers should be trusted:

1. **One year of data.** Annual seasonality cannot be learned from a single cycle. The model
   captures weekly and short-range variation only.
2. **Model selection used the holdout.** Hyperparameters were chosen by test-set MAPE, so 91.96%
   is mildly optimistic as an estimate of true future error. An honest production setup would
   add a third, untouched validation period.
3. **The forecast week is atypical.** 28 Dec - 3 Jan spans New Year, and the holiday analysis
   shows holidays run ~22% above normal. With only one year of data there is no second New Year
   to learn that pattern from, so this particular week carries more uncertainty than a mid-year
   week would.
4. **No inventory context.** The order assumes zero opening stock, no wastage allowance, no
   supplier pack sizes or minimum order quantities. A production system would layer these on top.
5. **Ingredient weights are per-pizza averages.** Real kitchens vary. The order should be read as
   a planning figure, not a precise physical requirement.

---

## 9. Conclusion

The pipeline works end to end: raw transaction data in, actionable purchase order out, at
roughly 92% forecast accuracy.

The most valuable findings were not the model scores but the data problems found on the way -
the dual date format that would have silently corrupted a third of the calendar, the truncated
weeks at both ends that looked like demand collapses, and the size-encoding in `pizza_name_id`
that determines whether ingredient totals are right or quietly wrong.

The modelling itself confirmed something worth remembering: on 51 weeks of well-behaved data,
a classical seasonal model beat a neural network, and a plain linear regression came within
1.1 points of the winner.
