# DATA QUALITY
The most important step in data analysis.
This helps in discovering data quality issues.
After loading the data in powerBI here are some of the issues found:
- The Order ID was not an ID. There were five prefix styles `(LC1000, LCL-1001, ORD1004, ord1020, CAR1086), 19 blanks or N/A, and three numbers used twice`.
- Dates were in six formats. `Some were numbers (46066), 
some 26-Mar-25, some Aug 29, 2025, and four were 2026-13-04, which is year-day-month. Worst were the 47 slash dates like 03/04/2026, which are valid as both 3 April and 4 March`.
- Categories exploded. `Toyota appeared as Toyota, TOYOTA, Toyta, Totoya, Toyota Kenya. In total, Car Make had 62 spellings for 10 real car makes. Sales reps had 80 "names" for 10 people, including Dan1el Kimani, where a digit 1 had replaced the letter i.`
- Discounts came in over 50 different spellings. `7%, 0.07, 7, 7 percent, ten percent, No Discount, #ERROR.`
- Numbers were formatted as text as shown below.
- Five columns held money: price, unit cost, delivery fee, logistics cost, and recorded revenue. They mixed `KSh, KES, plain numbers, $, USD, EUR, ZAR, R, a ?, and an M suffix (9.14M)`.
- Customer names, Region, Branch, City were not in proper case and inconsistent spellings like `COUNTY GOVERMENT, Msa, Nkr, Faith ACHIENG`.
- Numerical values that held monetary values had negative values like `-22,738,440` and customer rating of `-1`.
- Customer Age as text.

