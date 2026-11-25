# DAX Measures and Calculated Columns

> **Project:** Olist E-Commerce Analytics | Power BI
> **Source:** User-provided `DAX.txt` export/transcription.

This reference documents the DAX supplied for the five-page portfolio dashboard. Formulas are preserved as provided except for two explicitly noted missing closing parentheses. They have **not** been executed or independently validated against the PBIX model.

## Contents

- [Sales Performance](#sales-performance)
- [Delivery & Logistics](#delivery--logistics)
- [Customer Insights](#customer-insights)
- [Product Performance](#product-performance)
- [Date & Category Helpers](#date--category-helpers)

## Source corrections and scope

- **Average Delivery Delay (Days):** the supplied text omitted the final closing parenthesis for `AVERAGEX`; one `)` was appended in this documentation.
- **Category Revenue Contribution %:** the supplied text omitted the final closing parenthesis for `DIVIDE`; one `)` was appended in this documentation.
- The source ends with an unfinished `24.` label and no associated DAX definition; it is not presented as a measure.
- `Order Purchase Date` and `Category_Display` are **calculated columns**, not measures.
- Date and category filter behavior depends on relationships, slicers, and filter context in the PBIX.

## Sales Performance

### Total Sales

Sum of recorded item prices; excludes freight.

```dax
Total Sales =
SUM(Order_Items[price])
```

### Total Items Sold

Count of rows in the order-items table (items/line records).

```dax
Total Items Sold =
COUNTROWS(Order_Items)
```

### Total Freight

Sum of recorded freight charges on order items.

```dax
Total Freight =
SUM(Order_Items[freight_value])
```

### Average Order Value

Merchandise revenue per distinct order with recorded order items; not divided by all orders.

```dax
Average Order Value =
DIVIDE(
    [Total Sales],
    DISTINCTCOUNT(Order_Items[order_id]),
    0
)
```

### Average Freight Cost per Order

Total freight divided by distinct orders in the order-items table.

```dax
Average Freight Cost per Order =
DIVIDE(
    SUM(Order_Items[freight_value]),
    DISTINCTCOUNT(Order_Items[order_id])
)
```

### Total Orders

Distinct orders; when a product-category column is directly filtered, applies the matching order-item IDs to the orders table using TREATAS.

```dax
Total Orders =
VAR CategoryFiltered =
    ISFILTERED(Products[Category_Display])
        || ISFILTERED(Products[Category_English_Final])
        || ISFILTERED(Products[Product_Category_Clean])

VAR SelectedOrders =
    VALUES(Order_Items[order_id])

RETURN
    IF(
        CategoryFiltered,
        CALCULATE(
            DISTINCTCOUNT(orders[order_id]),
            KEEPFILTERS(
                TREATAS(
                    SelectedOrders,
                    orders[order_id]
                )
            )
        ),
        DISTINCTCOUNT(orders[order_id])
    )
```

### Orders With Items

Distinct order IDs present in the order-items table.

```dax
Orders With Items =
DISTINCTCOUNT(Order_Items[order_id])
```

## Delivery & Logistics

### Delivered Orders

Count of orders with status delivered, using the Total Orders measure.

```dax
Delivered Orders =
CALCULATE(
    [Total Orders],
    orders[order_status] = "delivered"
)
```

### Delivery Rate %

Delivered orders divided by total orders.

```dax
Delivery Rate % =
DIVIDE(
    CALCULATE(
        [Total Orders],
        orders[order_status] = "delivered"
    ),
    [Total Orders],
    0
)
```

### On-Time Delivery Rate %

Among delivered orders with actual and estimated delivery dates, fraction delivered on or before the estimated calendar date.

```dax
On-Time Delivery Rate % =
VAR SelectedOrders =
    VALUES(Order_Items[order_id])

VAR DeliveredOrders =
    CALCULATE(
        COUNTROWS(orders),
        KEEPFILTERS(
            TREATAS(
                SelectedOrders,
                orders[order_id]
            )
        ),
        orders[order_status] = "delivered",
        NOT ISBLANK(
            orders[order_delivered_customer_date]
        ),
        NOT ISBLANK(
            orders[order_estimated_delivery_date]
        )
    )

VAR OnTimeOrders =
    CALCULATE(
        COUNTROWS(orders),
        KEEPFILTERS(
            TREATAS(
                SelectedOrders,
                orders[order_id]
            )
        ),
        orders[order_status] = "delivered",
        FILTER(
            orders,
            NOT ISBLANK(
                orders[order_delivered_customer_date]
            )
            &&
            NOT ISBLANK(
                orders[order_estimated_delivery_date]
            )
            &&
            INT(orders[order_delivered_customer_date])
                <= INT(orders[order_estimated_delivery_date])
        )
    )

RETURN
    DIVIDE(
        OnTimeOrders,
        DeliveredOrders
    )
```

### Late Delivered Orders

Count of delivered orders with both delivery dates whose actual calendar delivery date is after the estimated date.

```dax
Late Delivered Orders =
VAR SelectedOrders =
    VALUES(Order_Items[order_id])
RETURN
    CALCULATE(
        DISTINCTCOUNT(orders[order_id]),
        KEEPFILTERS(
            TREATAS(
                SelectedOrders,
                orders[order_id]
            )
        ),
        orders[order_status] = "delivered",
        FILTER(
            orders,
            NOT ISBLANK(orders[order_delivered_customer_date])
                && NOT ISBLANK(orders[order_estimated_delivery_date])
                && INT(orders[order_delivered_customer_date])
                    > INT(orders[order_estimated_delivery_date])
        )
    )
```

### Late Delivery Rate %

Complement of on-time delivery rate, preserving blanks.

```dax
Late Delivery Rate % =
VAR OnTimeRate =
    [On-Time Delivery Rate %]
RETURN
    IF(
        ISBLANK(OnTimeRate),
        BLANK(),
        1 - OnTimeRate
    )
```

### Average Delivery Time (Days)

Average DATEDIFF in days between purchase timestamp and actual delivery for delivered orders with both timestamps.

```dax
Average Delivery Time (Days) =
VAR SelectedOrders =
    VALUES(Order_Items[order_id])
RETURN
    CALCULATE(
        AVERAGEX(
            FILTER(
                orders,
                orders[order_status] = "delivered"
                    && NOT ISBLANK(orders[order_purchase_timestamp])
                    && NOT ISBLANK(orders[order_delivered_customer_date])
            ),
            DATEDIFF(
                orders[order_purchase_timestamp],
                orders[order_delivered_customer_date],
                DAY
            )
        ),
        KEEPFILTERS(
            TREATAS(
                SelectedOrders,
                orders[order_id]
            )
        )
    )
```

### Average Delivery Delay (Days)

Average days late among late delivered orders only, comparing actual versus estimated calendar dates.

```dax
Average Delivery Delay (Days) =
VAR SelectedOrders =
    VALUES(Order_Items[order_id])

VAR LateOrders =
    CALCULATETABLE(
        FILTER(
            orders,
            orders[order_status] = "delivered"
                && NOT ISBLANK(
                    orders[order_delivered_customer_date]
                )
                && NOT ISBLANK(
                    orders[order_estimated_delivery_date]
                )
                && INT(
                    orders[order_delivered_customer_date]
                )
                > INT(
                    orders[order_estimated_delivery_date]
                )
        ),
        KEEPFILTERS(
            TREATAS(
                SelectedOrders,
                orders[order_id]
            )
        )
    )

RETURN
    AVERAGEX(
        LateOrders,
        DATEDIFF(
            INT(orders[order_estimated_delivery_date]),
            INT(orders[order_delivered_customer_date]),
            DAY
        )
)
```

## Customer Insights

### Total Unique Customers

Distinct unique customer IDs, with category-aware order filtering via TREATAS.

```dax
Total Unique Customers =
VAR CategoryFiltered =
    ISFILTERED(Products[Category_Display])
        || ISFILTERED(Products[Category_English_Final])
        || ISFILTERED(Products[Product_Category_Clean])

VAR SelectedOrders =
    VALUES(Order_Items[order_id])

RETURN
    IF(
        CategoryFiltered,
        CALCULATE(
            DISTINCTCOUNT(Customers[customer_unique_id]),
            KEEPFILTERS(
                TREATAS(
                    SelectedOrders,
                    orders[order_id]
                )
            )
        ),
        DISTINCTCOUNT(Customers[customer_unique_id])
    )
```

### Repeat Customers

Number of active unique customers with more than one distinct order in the current filter context.

```dax
Repeat Customers =
VAR CategoryFiltered =
    ISFILTERED(Products[Category_Display])
        || ISFILTERED(Products[Category_English_Final])
        || ISFILTERED(Products[Product_Category_Clean])

VAR SelectedOrders =
    VALUES(Order_Items[order_id])

VAR CustomerOrders =
    ADDCOLUMNS(
        VALUES(Customers[customer_unique_id]),
        "@OrderCount",
            IF(
                CategoryFiltered,
                CALCULATE(
                    DISTINCTCOUNT(orders[order_id]),
                    KEEPFILTERS(
                        TREATAS(
                            SelectedOrders,
                            orders[order_id]
                        )
                    )
                ),
                CALCULATE(
                    DISTINCTCOUNT(orders[order_id])
                )
            )
    )

RETURN
    COUNTROWS(
        FILTER(
            CustomerOrders,
            [@OrderCount] > 1
        )
    )
```

### Repeat Customer Rate %

Repeat customers divided by active unique customers in the current filter context; this is not a cohort retention rate.

```dax
Repeat Customer Rate % =
VAR CategoryFiltered =
    ISFILTERED(Products[Category_Display])
        || ISFILTERED(Products[Category_English_Final])
        || ISFILTERED(Products[Product_Category_Clean])

VAR SelectedOrders =
    VALUES(Order_Items[order_id])

VAR CustomerOrders =
    ADDCOLUMNS(
        VALUES(Customers[customer_unique_id]),
        "@OrderCount",
            IF(
                CategoryFiltered,
                CALCULATE(
                    DISTINCTCOUNT(orders[order_id]),
                    KEEPFILTERS(
                        TREATAS(
                            SelectedOrders,
                            orders[order_id]
                        )
                    )
                ),
                CALCULATE(
                    DISTINCTCOUNT(orders[order_id])
                )
            )
    )

VAR RepeatCustomers =
    COUNTROWS(
        FILTER(
            CustomerOrders,
            [@OrderCount] > 1
        )
    )

VAR ActiveCustomers =
    COUNTROWS(
        FILTER(
            CustomerOrders,
            [@OrderCount] > 0
        )
    )

RETURN
    DIVIDE(
        RepeatCustomers,
        ActiveCustomers,
        0
    )
```

### Average Orders per Customer

Total Orders divided by Total Unique Customers.

```dax
Average Orders per Customer =
DIVIDE(
    [Total Orders],
    [Total Unique Customers],
    0
)
```

## Product Performance

### Total Distinct Products

Distinct product IDs found in the order-items table.

```dax
Total Distinct Products =
DISTINCTCOUNT(Order_Items[product_id])
```

### Average Item Price

Average recorded item price across order-item rows.

```dax
Average Item Price =
DIVIDE(
    SUM(Order_Items[price]),
    COUNTROWS(Order_Items),
    0
)
```

### Category Revenue Contribution %

Selected category merchandise revenue divided by revenue with product-table filters removed; other filter context may remain.

```dax
Category Revenue Contribution % =
DIVIDE(
    [Total Sales],
    CALCULATE(
        [Total Sales],
        REMOVEFILTERS(Products)
    ),
    0
)
```

## Date & Category Helpers

### Latest Order Date

Maximum value of the Order Purchase Date column.

```dax
Latest Order Date =
MAX(orders[Order Purchase Date])
```

### Order Purchase Date

Calculated date-only column derived from the order purchase timestamp.

```dax
Order Purchase Date =
DATE(
    YEAR(orders[order_purchase_timestamp]),
    MONTH(orders[order_purchase_timestamp]),
    DAY(orders[order_purchase_timestamp])
)
```

### Category_Display

Calculated category label replacing underscores with spaces, capitalizing the first character, and labeling blanks Unknown.

```dax
Category_Display =
VAR CategoryName =
    Products[Category_English_Final]
VAR CleanName =
    SUBSTITUTE(CategoryName, "_", " ")
RETURN
    IF(
        ISBLANK(CategoryName),
        "Unknown",
        UPPER(LEFT(CleanName, 1)) &
        MID(CleanName, 2, LEN(CleanName))
    )
```

## Implementation notes

- **Category-aware measures:** `Total Orders`, `Total Unique Customers`, `Repeat Customers`, and `Repeat Customer Rate %` explicitly test direct filters on three product-category columns. `TREATAS` transfers selected order IDs to the orders table in the category-filtered branch.
- **On-time comparison:** `INT` removes the time-of-day component so orders delivered on the estimated date count as on time.
- **Repeat purchasing:** the rate is within the active reporting/filter context, not lifetime retention or a cohort-based retention metric.
- **AOV denominator:** distinct orders in `Order_Items`, which can differ from `Total Orders`.
- **Percent formatting:** apply percentage formatting to percentage measures in Power BI; the underlying DAX generally returns a fraction.

## Reproducibility

To verify these definitions, open the project PBIX in Power BI Desktop and compare each measure/column against the model formula bar. The supplied text alone does not establish model relationships, Power Query transformations, or all formatting settings.
