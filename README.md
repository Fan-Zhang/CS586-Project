# CS586 Introduction to Databases Fall 2016 Graduate Project
## Fan Zhang

### ER Diagram

![ER Diagram](https://github.com/Fan-Zhang/CS586-Project/blob/master/ER-Diagram.png?raw=true)


### CREATE TABLE Statements

    create table household (
        household_id        integer primary key,
        age_desc            text,
        marital_status_code text,
        income_desc         text,
        homeowner_desc      text,
        hh_comp_desc        text,
        household_size_desc text,
        kid_category_desc   text
    );

    create table product (
        product_id           integer primary key,
        manufacturer         text,
        department           text,
        brand                text,
        commodity_desc       text,
        sub_commodity_desc   text,
        curr_size_of_product text
    );

    create table store (
        store_id integer primary key,
        state    text
    );

    create table sells (
        product_id integer references product (product_id),
        store_id   integer references store (store_id),
        primary key (product_id, store_id)
    );

    create table transaction (
        transaction_id bigint  primary key,
        household_id   integer references household (household_id),
        store_id       integer references store (store_id),
        day            integer,
        trans_time     integer,
        week_no        integer
    );

    create table includes (
        transaction_id bigint  references transaction (transaction_id),
        product_id     integer references product (product_id),
        quantity       integer,
        sales_value    float,
        primary key (transaction_id, product_id)
    );

    create table shops_at (
        household_id integer references household (household_id),
        store_id     integer references store (store_id),
        primary key (household_id, store_id)
    );

### How the database was populated

The dataset used can be found at
[dunnhumby](http://www.dunnhumby.com/sourcefiles), under the "The Complete
Journey" section.  The zip file contains 8 CSV files.  This project used four
of them: causal_data.csv, hh_demographic.csv, product,csv and
transaction_data.csv.  Here's the process to prepare the data to load into
tables, done with a Linux shell and common Linux tools:

1. Make sure simple split by comma works (observe that there's no output):

        $ perl -ne 'chomp; @F=split/,/; print(@F) if scalar(@F) != 8'  hh_demographic.csv
        $ perl -ne 'chomp; @F=split/,/; print(@F) if scalar(@F) != 7'  product.csv
        $ perl -ne 'chomp; @F=split/,/; print(@F) if scalar(@F) != 12' transaction_data.csv
        $ perl -ne 'chomp; @F=split/,/; print(@F) if scalar(@F) != 5'  causal_data.csv

2. Make sure there's no pipe in any file (so we can use that as the delimitor
   in the next step):

        $ fgrep '|' *csv

3. Extract relevant columns, skip headers, and deduplicate:

        $ tail -n+2 hh_demographic.csv   |
            awk -F',' '{print $8"|"$1"|"$2"|"$3"|"$4"|"$5"|"$6"|"$7}'  \
            > household.psv
        $ tail -n+2 product.csv          |
            awk -F',' '{print $1"|"$2"|"$3"|"$4"|"$5"|"$6"|"$7}'       \
            > product.psv
        $ tail -n+2 transaction_data.csv |
            awk -F',' '{print $7}' | sort -u                           \
            > store.psv
        $ ./assign-store-state.pl < store.psv > store_state.psv
        $ tail -n+2 causal_data.csv      |
            awk -F',' '{print $1"|"$2"}'                               \
            > sells.psv
        $ tail -n+2 transaction_data.csv |
            awk -F',' '{print $2"|"$1"|"$7"|"$3"|"$9"|"$10}' | sort -u \
            > transaction.psv
        $ tail -n+2 transaction_data.csv |
            awk -F',' '{print $2"|"$4"|"$5"|"$6}'                      \
            > includes.psv
        $ tail -n+2 transaction_data.csv |
            awk -F',' '{print $1"|"$7}' | sort -u                      \
            > shops_at.psv

4. `assign-store-state.pl` is a small Perl script that randomly assigns one of
   the three states to stores:

        #!/usr/bin/env perl

        use strict;
        use warnings;

        my @states = ('CA', 'OR', 'WA');

        while(<>) {
            chomp;
            my $state = $states[int(rand(3))];
            print("$_|$state\n");
        }

5. Load into tables:

        psql> \copy household from household.psv     delimiter '|'
        psql> \copy product from product.psv         delimiter '|'
        psql> \copy store from store_state.psv       delimiter '|'
        psql> \copy sells from sells.psv             delimiter '|'
        psql> \copy transaction from transaction.psv delimiter '|'

* transaction.psv has some `household_id`s that are not in the `household`
  table.  To fix it:

        psql> create temp table tmp_transaction as (select * from transaction limit 0);
        psql> \copy tmp_transaction from transaction.psv delimiter '|'
        psql> insert into transaction (select * from tmp_transaction
                where household_id in (select household_id from household));

* This means we did not insert certain transactions.  We have to circumvent
  this problem for subsequent files:

        psql> create temp table tmp_transaction_product as (select * from includes limit 0);
        psql> \copy tmp_transaction_product from includes.psv delimiter '|'
        psql> insert into includes (select * from tmp_transaction_product
                where transaction_id in (select transaction_id from transaction));

        psql> create temp table tmp_shops_at as (select * from shops_at limit 0);
        psql> \copy tmp_shops_at from shops_at.psv delimiter '|'
        psql> insert into shops_at (select * from tmp_shops_at
                where household_id in (select household_id from household));

        psql> drop table tmp_transaction;
        psql> drop table tmp_transaction_product;
        psql> drop table tmp_shops_at;

        psql> commit;

* Many rows have been removed for this project because of the size of the
  dataset.  For example before trimming, the `sells` table has 36,786,524 rows,
  which is quite unwieldy when joing with others.

### 20 Questions, the queries, and the answers

1. List all information about families who are in the age group 45-54.

        SELECT * FROM household WHERE AGE_DESC='45-54';

         household_id | age_desc | marital_status_code | income_desc | homeowner_desc  |   hh_comp_desc   | household_size_desc | kid_category_desc
        --------------+----------+---------------------+-------------+-----------------+------------------+---------------------+-------------------
                    7 | 45-54    | Married             | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                   18 | 45-54    | Married             | 100-124K    | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                   22 | 45-54    | Married             | 75-99K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                   46 | 45-54    | Married             | 150-174K    | Homeowner       | 2 Adults Kids    | 5+                  | 3+
                   67 | 45-54    | Married             | Under 15K   | Homeowner       | 2 Adults Kids    | 5+                  | 3+
                  101 | 45-54    | Married             | Under 15K   | Homeowner       | 2 Adults Kids    | 4                   | 2
                  133 | 45-54    | Married             | Under 15K   | Homeowner       | Single Male      | 2                   | None/Unknown
                  192 | 45-54    | Married             | 35-49K      | Homeowner       | Single Male      | 2                   | None/Unknown
                  193 | 45-54    | Married             | 75-99K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  201 | 45-54    | Married             | 15-24K      | Homeowner       | 2 Adults Kids    | 5+                  | 3+
                  218 | 45-54    | Married             | 15-24K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  220 | 45-54    | Married             | 35-49K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  242 | 45-54    | Married             | 75-99K      | Homeowner       | 2 Adults Kids    | 5+                  | 3+
                  250 | 45-54    | Married             | 75-99K      | Homeowner       | 1 Adult Kids     | 4                   | 2
                  317 | 45-54    | Married             | 75-99K      | Homeowner       | Single Male      | 2                   | None/Unknown
                  321 | 45-54    | Married             | 100-124K    | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  334 | 45-54    | Married             | Under 15K   | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  350 | 45-54    | Married             | 50-74K      | Homeowner       | 1 Adult Kids     | 4                   | 2
                  353 | 45-54    | Married             | 150-174K    | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  377 | 45-54    | Married             | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  417 | 45-54    | Married             | 35-49K      | Unknown         | Single Female    | 2                   | None/Unknown
                  443 | 45-54    | Married             | 25-34K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  491 | 45-54    | Married             | 75-99K      | Unknown         | Single Female    | 2                   | None/Unknown
                  518 | 45-54    | Married             | 75-99K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  582 | 45-54    | Married             | 125-149K    | Homeowner       | Unknown          | 3                   | 1
                  596 | 45-54    | Married             | 15-24K      | Homeowner       | 1 Adult Kids     | 3                   | 1
                  623 | 45-54    | Married             | 35-49K      | Homeowner       | 2 Adults Kids    | 3                   | 1
                  699 | 45-54    | Married             | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  716 | 45-54    | Married             | 75-99K      | Homeowner       | 2 Adults Kids    | 5+                  | 3+
                  718 | 45-54    | Married             | 25-34K      | Homeowner       | 2 Adults Kids    | 5+                  | 3+
                  733 | 45-54    | Married             | 75-99K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  740 | 45-54    | Married             | 175-199K    | Homeowner       | Single Female    | 2                   | None/Unknown
                  753 | 45-54    | Married             | 75-99K      | Homeowner       | 2 Adults Kids    | 4                   | 2
                  766 | 45-54    | Married             | 150-174K    | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  784 | 45-54    | Married             | 100-124K    | Homeowner       | Single Female    | 2                   | None/Unknown
                  825 | 45-54    | Married             | 175-199K    | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  844 | 45-54    | Married             | 200-249K    | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  908 | 45-54    | Married             | 100-124K    | Probable Owner  | 2 Adults Kids    | 3                   | 1
                  920 | 45-54    | Married             | 35-49K      | Homeowner       | 2 Adults Kids    | 5+                  | 3+
                  981 | 45-54    | Married             | 25-34K      | Homeowner       | Single Female    | 2                   | None/Unknown
                  983 | 45-54    | Married             | 35-49K      | Homeowner       | 2 Adults Kids    | 3                   | 1
                  992 | 45-54    | Married             | 35-49K      | Homeowner       | 2 Adults Kids    | 3                   | 1
                 1014 | 45-54    | Married             | 15-24K      | Unknown         | 2 Adults Kids    | 4                   | 2
                 1018 | 45-54    | Married             | 35-49K      | Homeowner       | 2 Adults Kids    | 5+                  | 3+
                 1020 | 45-54    | Married             | 25-34K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1074 | 45-54    | Married             | 35-49K      | Homeowner       | 2 Adults Kids    | 4                   | 2
                 1091 | 45-54    | Married             | 75-99K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1094 | 45-54    | Married             | 75-99K      | Homeowner       | 2 Adults Kids    | 5+                  | 3+
                 1113 | 45-54    | Married             | 50-74K      | Homeowner       | 2 Adults Kids    | 3                   | 1
                 1142 | 45-54    | Married             | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1154 | 45-54    | Married             | 35-49K      | Homeowner       | Unknown          | 2                   | None/Unknown
                 1159 | 45-54    | Married             | 15-24K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1166 | 45-54    | Married             | 125-149K    | Homeowner       | 2 Adults Kids    | 3                   | 1
                 1186 | 45-54    | Married             | 50-74K      | Homeowner       | 1 Adult Kids     | 5+                  | 3+
                 1218 | 45-54    | Married             | 35-49K      | Homeowner       | 2 Adults Kids    | 4                   | 2
                 1261 | 45-54    | Married             | 25-34K      | Homeowner       | Single Female    | 2                   | None/Unknown
                 1300 | 45-54    | Married             | 50-74K      | Homeowner       | 2 Adults Kids    | 4                   | 2
                 1391 | 45-54    | Married             | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1394 | 45-54    | Married             | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1412 | 45-54    | Married             | 50-74K      | Homeowner       | 2 Adults Kids    | 3                   | 1
                 1451 | 45-54    | Married             | 125-149K    | Homeowner       | 1 Adult Kids     | 5+                  | 3+
                 1452 | 45-54    | Married             | 25-34K      | Unknown         | 2 Adults Kids    | 3                   | 1
                 1453 | 45-54    | Married             | 125-149K    | Homeowner       | 2 Adults Kids    | 3                   | 1
                 1509 | 45-54    | Married             | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1517 | 45-54    | Married             | 50-74K      | Homeowner       | 2 Adults Kids    | 4                   | 2
                 1524 | 45-54    | Married             | 35-49K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1545 | 45-54    | Married             | 35-49K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1556 | 45-54    | Married             | 35-49K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1563 | 45-54    | Married             | 75-99K      | Homeowner       | Unknown          | 3                   | 1
                 1609 | 45-54    | Married             | 125-149K    | Homeowner       | 2 Adults Kids    | 5+                  | 3+
                 1633 | 45-54    | Married             | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1650 | 45-54    | Married             | 75-99K      | Homeowner       | 2 Adults Kids    | 3                   | 1
                 1677 | 45-54    | Married             | 75-99K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1696 | 45-54    | Married             | 50-74K      | Homeowner       | 2 Adults Kids    | 3                   | 1
                 1745 | 45-54    | Married             | Under 15K   | Unknown         | Single Male      | 2                   | None/Unknown
                 1762 | 45-54    | Married             | 125-149K    | Homeowner       | 2 Adults Kids    | 5+                  | 3+
                 1764 | 45-54    | Married             | 75-99K      | Homeowner       | 2 Adults Kids    | 4                   | 2
                 1775 | 45-54    | Married             | 75-99K      | Homeowner       | 2 Adults Kids    | 5+                  | 3+
                 1820 | 45-54    | Married             | 150-174K    | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1823 | 45-54    | Married             | 75-99K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1844 | 45-54    | Married             | 250K+       | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1848 | 45-54    | Married             | 75-99K      | Homeowner       | Unknown          | 3                   | 1
                 1896 | 45-54    | Married             | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1937 | 45-54    | Married             | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2007 | 45-54    | Married             | 50-74K      | Homeowner       | 2 Adults Kids    | 3                   | 1
                 2012 | 45-54    | Married             | 25-34K      | Renter          | Single Male      | 2                   | None/Unknown
                 2017 | 45-54    | Married             | 75-99K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2053 | 45-54    | Married             | 15-24K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2082 | 45-54    | Married             | Under 15K   | Homeowner       | 2 Adults Kids    | 3                   | 1
                 2086 | 45-54    | Married             | 25-34K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2094 | 45-54    | Married             | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2112 | 45-54    | Married             | 35-49K      | Homeowner       | 2 Adults Kids    | 5+                  | 3+
                 2154 | 45-54    | Married             | 35-49K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2156 | 45-54    | Married             | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2179 | 45-54    | Married             | 35-49K      | Homeowner       | 2 Adults Kids    | 3                   | 1
                 2182 | 45-54    | Married             | 75-99K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2203 | 45-54    | Married             | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2243 | 45-54    | Married             | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2252 | 45-54    | Married             | 50-74K      | Homeowner       | 2 Adults Kids    | 4                   | 2
                 2264 | 45-54    | Married             | 250K+       | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2284 | 45-54    | Married             | 250K+       | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2292 | 45-54    | Married             | 35-49K      | Renter          | 2 Adults Kids    | 5+                  | 3+
                 2307 | 45-54    | Married             | 75-99K      | Homeowner       | 2 Adults Kids    | 3                   | 1
                 2312 | 45-54    | Married             | 250K+       | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2314 | 45-54    | Married             | 125-149K    | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2351 | 45-54    | Married             | 75-99K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2359 | 45-54    | Married             | 25-34K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2364 | 45-54    | Married             | 25-34K      | Renter          | 2 Adults No Kids | 2                   | None/Unknown
                 2407 | 45-54    | Married             | 100-124K    | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2421 | 45-54    | Married             | 150-174K    | Homeowner       | 2 Adults Kids    | 5+                  | 3+
                 2448 | 45-54    | Married             | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2496 | 45-54    | Married             | 75-99K      | Homeowner       | Unknown          | 3                   | 1
                   16 | 45-54    | Single              | 50-74K      | Homeowner       | Single Female    | 1                   | None/Unknown
                   52 | 45-54    | Single              | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                   57 | 45-54    | Single              | 35-49K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  118 | 45-54    | Single              | 35-49K      | Homeowner       | Single Female    | 1                   | None/Unknown
                  123 | 45-54    | Single              | 35-49K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  289 | 45-54    | Single              | 100-124K    | Homeowner       | Single Male      | 1                   | None/Unknown
                  390 | 45-54    | Single              | 25-34K      | Renter          | Single Female    | 1                   | None/Unknown
                  441 | 45-54    | Single              | 50-74K      | Homeowner       | Single Female    | 1                   | None/Unknown
                  706 | 45-54    | Single              | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  748 | 45-54    | Single              | Under 15K   | Homeowner       | 2 Adults Kids    | 3                   | 1
                  831 | 45-54    | Single              | 50-74K      | Homeowner       | 1 Adult Kids     | 3                   | 2
                  914 | 45-54    | Single              | 15-24K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  929 | 45-54    | Single              | 25-34K      | Renter          | 1 Adult Kids     | 2                   | 1
                 1131 | 45-54    | Single              | 50-74K      | Homeowner       | 2 Adults Kids    | 3                   | 1
                 1234 | 45-54    | Single              | 35-49K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1397 | 45-54    | Single              | 35-49K      | Renter          | Single Female    | 1                   | None/Unknown
                 1429 | 45-54    | Single              | 35-49K      | Renter          | Single Female    | 1                   | None/Unknown
                 1585 | 45-54    | Single              | 150-174K    | Homeowner       | Single Female    | 1                   | None/Unknown
                 1791 | 45-54    | Single              | 100-124K    | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1841 | 45-54    | Single              | 35-49K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2087 | 45-54    | Single              | 100-124K    | Homeowner       | Unknown          | 1                   | None/Unknown
                 2294 | 45-54    | Single              | 25-34K      | Renter          | 2 Adults Kids    | 4                   | 2
                 2305 | 45-54    | Single              | 50-74K      | Homeowner       | Single Female    | 1                   | None/Unknown
                 2453 | 45-54    | Single              | 50-74K      | Homeowner       | Single Female    | 1                   | None/Unknown
                 2455 | 45-54    | Single              | 25-34K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2483 | 45-54    | Single              | 75-99K      | Homeowner       | Single Male      | 1                   | None/Unknown
                 2488 | 45-54    | Single              | 35-49K      | Homeowner       | Single Female    | 1                   | None/Unknown
                   27 | 45-54    | Unknown             | 25-34K      | Probable Renter | Single Female    | 1                   | None/Unknown
                   40 | 45-54    | Unknown             | Under 15K   | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                   80 | 45-54    | Unknown             | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                   97 | 45-54    | Unknown             | 75-99K      | Unknown         | Single Female    | 1                   | None/Unknown
                  119 | 45-54    | Unknown             | 125-149K    | Unknown         | Single Male      | 1                   | None/Unknown
                  121 | 45-54    | Unknown             | 25-34K      | Unknown         | 2 Adults Kids    | 3                   | 1
                  127 | 45-54    | Unknown             | 25-34K      | Unknown         | Single Female    | 1                   | None/Unknown
                  131 | 45-54    | Unknown             | 50-74K      | Unknown         | 2 Adults No Kids | 2                   | None/Unknown
                  136 | 45-54    | Unknown             | 50-74K      | Unknown         | Unknown          | 1                   | None/Unknown
                  149 | 45-54    | Unknown             | 35-49K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  158 | 45-54    | Unknown             | 50-74K      | Homeowner       | Unknown          | 1                   | None/Unknown
                  159 | 45-54    | Unknown             | Under 15K   | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  164 | 45-54    | Unknown             | 35-49K      | Unknown         | 2 Adults No Kids | 2                   | None/Unknown
                  184 | 45-54    | Unknown             | 25-34K      | Unknown         | Single Male      | 1                   | None/Unknown
                  208 | 45-54    | Unknown             | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  209 | 45-54    | Unknown             | 25-34K      | Homeowner       | 2 Adults Kids    | 3                   | 1
                  239 | 45-54    | Unknown             | 50-74K      | Renter          | Unknown          | 1                   | None/Unknown
                  253 | 45-54    | Unknown             | Under 15K   | Unknown         | Single Female    | 1                   | None/Unknown
                  263 | 45-54    | Unknown             | 35-49K      | Unknown         | 2 Adults No Kids | 2                   | None/Unknown
                  264 | 45-54    | Unknown             | 75-99K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  282 | 45-54    | Unknown             | 35-49K      | Unknown         | Single Female    | 1                   | None/Unknown
                  302 | 45-54    | Unknown             | 35-49K      | Homeowner       | 2 Adults Kids    | 3                   | 1
                  314 | 45-54    | Unknown             | 75-99K      | Homeowner       | 2 Adults Kids    | 3                   | 1
                  319 | 45-54    | Unknown             | 35-49K      | Homeowner       | 1 Adult Kids     | 4                   | 3+
                  324 | 45-54    | Unknown             | 50-74K      | Unknown         | Single Female    | 1                   | None/Unknown
                  351 | 45-54    | Unknown             | 50-74K      | Unknown         | Unknown          | 1                   | None/Unknown
                  361 | 45-54    | Unknown             | 50-74K      | Unknown         | Unknown          | 1                   | None/Unknown
                  383 | 45-54    | Unknown             | 50-74K      | Homeowner       | Unknown          | 1                   | None/Unknown
                  413 | 45-54    | Unknown             | 100-124K    | Unknown         | Unknown          | 1                   | None/Unknown
                  426 | 45-54    | Unknown             | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  469 | 45-54    | Unknown             | 75-99K      | Probable Owner  | Single Female    | 1                   | None/Unknown
                  473 | 45-54    | Unknown             | 50-74K      | Homeowner       | Unknown          | 1                   | None/Unknown
                  483 | 45-54    | Unknown             | 35-49K      | Unknown         | Single Female    | 1                   | None/Unknown
                  492 | 45-54    | Unknown             | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  513 | 45-54    | Unknown             | 75-99K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  526 | 45-54    | Unknown             | 50-74K      | Unknown         | Unknown          | 1                   | None/Unknown
                  527 | 45-54    | Unknown             | 50-74K      | Homeowner       | Single Female    | 1                   | None/Unknown
                  534 | 45-54    | Unknown             | 35-49K      | Unknown         | Single Male      | 1                   | None/Unknown
                  546 | 45-54    | Unknown             | 50-74K      | Unknown         | 2 Adults No Kids | 2                   | None/Unknown
                  553 | 45-54    | Unknown             | 75-99K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  606 | 45-54    | Unknown             | 35-49K      | Probable Renter | Unknown          | 1                   | None/Unknown
                  607 | 45-54    | Unknown             | 125-149K    | Unknown         | Unknown          | 1                   | None/Unknown
                  609 | 45-54    | Unknown             | 50-74K      | Unknown         | Unknown          | 1                   | None/Unknown
                  621 | 45-54    | Unknown             | 35-49K      | Unknown         | Single Male      | 1                   | None/Unknown
                  624 | 45-54    | Unknown             | 50-74K      | Unknown         | Unknown          | 1                   | None/Unknown
                  661 | 45-54    | Unknown             | 50-74K      | Homeowner       | Unknown          | 1                   | None/Unknown
                  680 | 45-54    | Unknown             | 50-74K      | Homeowner       | Unknown          | 1                   | None/Unknown
                  693 | 45-54    | Unknown             | 15-24K      | Unknown         | 2 Adults No Kids | 2                   | None/Unknown
                  725 | 45-54    | Unknown             | 25-34K      | Unknown         | Single Female    | 1                   | None/Unknown
                  731 | 45-54    | Unknown             | 35-49K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  771 | 45-54    | Unknown             | 50-74K      | Homeowner       | Unknown          | 1                   | None/Unknown
                  790 | 45-54    | Unknown             | 50-74K      | Homeowner       | Unknown          | 1                   | None/Unknown
                  808 | 45-54    | Unknown             | 75-99K      | Homeowner       | 2 Adults Kids    | 3                   | 1
                  863 | 45-54    | Unknown             | 15-24K      | Unknown         | Unknown          | 1                   | None/Unknown
                  866 | 45-54    | Unknown             | 15-24K      | Unknown         | 1 Adult Kids     | 2                   | 1
                  867 | 45-54    | Unknown             | 75-99K      | Homeowner       | Unknown          | 1                   | None/Unknown
                  882 | 45-54    | Unknown             | 50-74K      | Unknown         | 2 Adults No Kids | 2                   | None/Unknown
                  895 | 45-54    | Unknown             | 150-174K    | Unknown         | Single Male      | 1                   | None/Unknown
                  915 | 45-54    | Unknown             | 15-24K      | Unknown         | Single Female    | 1                   | None/Unknown
                  922 | 45-54    | Unknown             | 35-49K      | Unknown         | Unknown          | 1                   | None/Unknown
                  934 | 45-54    | Unknown             | Under 15K   | Unknown         | Single Female    | 1                   | None/Unknown
                  939 | 45-54    | Unknown             | 75-99K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  956 | 45-54    | Unknown             | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                  982 | 45-54    | Unknown             | 35-49K      | Unknown         | 2 Adults Kids    | 4                   | 2
                  997 | 45-54    | Unknown             | 75-99K      | Homeowner       | Single Male      | 1                   | None/Unknown
                 1001 | 45-54    | Unknown             | 50-74K      | Homeowner       | Unknown          | 1                   | None/Unknown
                 1015 | 45-54    | Unknown             | 50-74K      | Homeowner       | Unknown          | 1                   | None/Unknown
                 1082 | 45-54    | Unknown             | 35-49K      | Unknown         | Single Male      | 1                   | None/Unknown
                 1135 | 45-54    | Unknown             | 50-74K      | Unknown         | Unknown          | 2                   | 1
                 1151 | 45-54    | Unknown             | Under 15K   | Probable Renter | Single Male      | 1                   | None/Unknown
                 1158 | 45-54    | Unknown             | 15-24K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1178 | 45-54    | Unknown             | 15-24K      | Unknown         | Single Female    | 1                   | None/Unknown
                 1188 | 45-54    | Unknown             | 35-49K      | Renter          | Unknown          | 1                   | None/Unknown
                 1222 | 45-54    | Unknown             | 15-24K      | Homeowner       | Single Female    | 1                   | None/Unknown
                 1228 | 45-54    | Unknown             | 100-124K    | Unknown         | Single Female    | 1                   | None/Unknown
                 1240 | 45-54    | Unknown             | Under 15K   | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1248 | 45-54    | Unknown             | 50-74K      | Unknown         | 1 Adult Kids     | 2                   | 1
                 1256 | 45-54    | Unknown             | 50-74K      | Unknown         | 1 Adult Kids     | 2                   | 1
                 1257 | 45-54    | Unknown             | 50-74K      | Homeowner       | 2 Adults Kids    | 3                   | 1
                 1258 | 45-54    | Unknown             | 75-99K      | Homeowner       | Unknown          | 1                   | None/Unknown
                 1264 | 45-54    | Unknown             | 15-24K      | Unknown         | Single Male      | 1                   | None/Unknown
                 1270 | 45-54    | Unknown             | 15-24K      | Unknown         | Single Female    | 1                   | None/Unknown
                 1306 | 45-54    | Unknown             | Under 15K   | Unknown         | Unknown          | 1                   | None/Unknown
                 1326 | 45-54    | Unknown             | Under 15K   | Homeowner       | Single Male      | 1                   | None/Unknown
                 1333 | 45-54    | Unknown             | 125-149K    | Homeowner       | 2 Adults Kids    | 4                   | 2
                 1337 | 45-54    | Unknown             | 50-74K      | Homeowner       | 2 Adults Kids    | 4                   | 2
                 1382 | 45-54    | Unknown             | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1423 | 45-54    | Unknown             | 15-24K      | Unknown         | Single Male      | 1                   | None/Unknown
                 1438 | 45-54    | Unknown             | Under 15K   | Unknown         | Single Female    | 1                   | None/Unknown
                 1470 | 45-54    | Unknown             | 35-49K      | Unknown         | Single Female    | 1                   | None/Unknown
                 1479 | 45-54    | Unknown             | 15-24K      | Probable Renter | Unknown          | 1                   | None/Unknown
                 1505 | 45-54    | Unknown             | 25-34K      | Unknown         | Single Female    | 1                   | None/Unknown
                 1529 | 45-54    | Unknown             | 50-74K      | Unknown         | Single Male      | 1                   | None/Unknown
                 1536 | 45-54    | Unknown             | 50-74K      | Unknown         | Unknown          | 1                   | None/Unknown
                 1540 | 45-54    | Unknown             | 15-24K      | Unknown         | 2 Adults Kids    | 4                   | 2
                 1549 | 45-54    | Unknown             | 15-24K      | Homeowner       | Unknown          | 1                   | None/Unknown
                 1567 | 45-54    | Unknown             | 15-24K      | Renter          | Single Male      | 1                   | None/Unknown
                 1578 | 45-54    | Unknown             | 15-24K      | Unknown         | 2 Adults No Kids | 2                   | None/Unknown
                 1595 | 45-54    | Unknown             | 50-74K      | Homeowner       | Single Male      | 1                   | None/Unknown
                 1678 | 45-54    | Unknown             | 75-99K      | Unknown         | Single Female    | 1                   | None/Unknown
                 1686 | 45-54    | Unknown             | 15-24K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1689 | 45-54    | Unknown             | 50-74K      | Unknown         | Unknown          | 1                   | None/Unknown
                 1708 | 45-54    | Unknown             | 25-34K      | Unknown         | Single Female    | 1                   | None/Unknown
                 1720 | 45-54    | Unknown             | 50-74K      | Unknown         | Unknown          | 1                   | None/Unknown
                 1722 | 45-54    | Unknown             | 15-24K      | Unknown         | 2 Adults No Kids | 2                   | None/Unknown
                 1748 | 45-54    | Unknown             | 35-49K      | Unknown         | Single Male      | 1                   | None/Unknown
                 1753 | 45-54    | Unknown             | 35-49K      | Homeowner       | Unknown          | 1                   | None/Unknown
                 1765 | 45-54    | Unknown             | 35-49K      | Unknown         | Single Male      | 1                   | None/Unknown
                 1795 | 45-54    | Unknown             | 25-34K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1802 | 45-54    | Unknown             | 150-174K    | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 1812 | 45-54    | Unknown             | 50-74K      | Unknown         | Single Female    | 1                   | None/Unknown
                 1815 | 45-54    | Unknown             | 25-34K      | Unknown         | 2 Adults No Kids | 2                   | None/Unknown
                 1834 | 45-54    | Unknown             | 125-149K    | Homeowner       | Unknown          | 5+                  | 3+
                 1845 | 45-54    | Unknown             | 175-199K    | Unknown         | 2 Adults Kids    | 3                   | 1
                 1847 | 45-54    | Unknown             | 175-199K    | Unknown         | Single Female    | 1                   | None/Unknown
                 1864 | 45-54    | Unknown             | 125-149K    | Homeowner       | 1 Adult Kids     | 5+                  | 3+
                 1865 | 45-54    | Unknown             | 50-74K      | Homeowner       | Unknown          | 1                   | None/Unknown
                 1901 | 45-54    | Unknown             | 35-49K      | Homeowner       | 2 Adults Kids    | 3                   | 1
                 1931 | 45-54    | Unknown             | 50-74K      | Homeowner       | Unknown          | 1                   | None/Unknown
                 1949 | 45-54    | Unknown             | Under 15K   | Unknown         | Single Female    | 1                   | None/Unknown
                 1953 | 45-54    | Unknown             | 50-74K      | Homeowner       | Unknown          | 1                   | None/Unknown
                 1979 | 45-54    | Unknown             | 150-174K    | Unknown         | 2 Adults Kids    | 3                   | 1
                 2004 | 45-54    | Unknown             | 50-74K      | Unknown         | Unknown          | 1                   | None/Unknown
                 2006 | 45-54    | Unknown             | Under 15K   | Unknown         | Single Male      | 1                   | None/Unknown
                 2011 | 45-54    | Unknown             | 75-99K      | Homeowner       | Single Male      | 1                   | None/Unknown
                 2018 | 45-54    | Unknown             | 50-74K      | Unknown         | Unknown          | 1                   | None/Unknown
                 2023 | 45-54    | Unknown             | 35-49K      | Homeowner       | 2 Adults Kids    | 3                   | 1
                 2070 | 45-54    | Unknown             | 50-74K      | Unknown         | Unknown          | 1                   | None/Unknown
                 2111 | 45-54    | Unknown             | 75-99K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2123 | 45-54    | Unknown             | 75-99K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2162 | 45-54    | Unknown             | 75-99K      | Homeowner       | Unknown          | 1                   | None/Unknown
                 2200 | 45-54    | Unknown             | Under 15K   | Unknown         | Single Male      | 1                   | None/Unknown
                 2235 | 45-54    | Unknown             | 175-199K    | Homeowner       | 2 Adults Kids    | 3                   | 1
                 2237 | 45-54    | Unknown             | 50-74K      | Homeowner       | Single Female    | 1                   | None/Unknown
                 2260 | 45-54    | Unknown             | 25-34K      | Homeowner       | Single Female    | 1                   | None/Unknown
                 2300 | 45-54    | Unknown             | 50-74K      | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2322 | 45-54    | Unknown             | 175-199K    | Homeowner       | Single Male      | 1                   | None/Unknown
                 2328 | 45-54    | Unknown             | 250K+       | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2330 | 45-54    | Unknown             | 50-74K      | Homeowner       | Unknown          | 1                   | None/Unknown
                 2341 | 45-54    | Unknown             | 50-74K      | Homeowner       | Unknown          | 1                   | None/Unknown
                 2347 | 45-54    | Unknown             | 35-49K      | Unknown         | 2 Adults No Kids | 2                   | None/Unknown
                 2367 | 45-54    | Unknown             | Under 15K   | Unknown         | 2 Adults No Kids | 2                   | None/Unknown
                 2374 | 45-54    | Unknown             | 150-174K    | Homeowner       | 2 Adults No Kids | 2                   | None/Unknown
                 2376 | 45-54    | Unknown             | 35-49K      | Unknown         | Single Female    | 1                   | None/Unknown
                 2380 | 45-54    | Unknown             | 50-74K      | Homeowner       | Unknown          | 1                   | None/Unknown
                 2420 | 45-54    | Unknown             | 15-24K      | Unknown         | 2 Adults Kids    | 3                   | 1
                 2435 | 45-54    | Unknown             | 50-74K      | Homeowner       | 1 Adult Kids     | 5+                  | 3+
                 2445 | 45-54    | Unknown             | 35-49K      | Unknown         | Unknown          | 1                   | None/Unknown
                 2497 | 45-54    | Unknown             | 35-49K      | Unknown         | Single Male      | 1                   | None/Unknown
                (288 rows)

2. List the age group, home owner description and composition of families who
   make less than 24K USD a year and have more than 3 family members.

        SELECT DISTINCT AGE_DESC, HOMEOWNER_DESC, HH_COMP_DESC
        FROM household
        WHERE (INCOME_DESC='Under 15K' OR INCOME_DESC='15-24K') AND (HOUSEHOLD_SIZE_DESC='4' OR HOUSEHOLD_SIZE_DESC='5+');

         age_desc | homeowner_desc | hh_comp_desc
        ----------+----------------+---------------
         19-24    | Renter         | 1 Adult Kids
         19-24    | Renter         | 2 Adults Kids
         25-34    | Homeowner      | 2 Adults Kids
         25-34    | Renter         | 2 Adults Kids
         25-34    | Unknown        | 2 Adults Kids
         35-44    | Homeowner      | 2 Adults Kids
         35-44    | Renter         | 1 Adult Kids
         35-44    | Renter         | 2 Adults Kids
         35-44    | Unknown        | 2 Adults Kids
         45-54    | Homeowner      | 2 Adults Kids
         45-54    | Unknown        | 2 Adults Kids
        (11 rows)

3. How many manufacturers are there for each product category?

        SELECT COMMODITY_DESC, COUNT(DISTINCT MANUFACTURER) FROM product GROUP BY COMMODITY_DESC;

                 commodity_desc         | count
        --------------------------------+-------
                                        |     1
         ADULT INCONTINENCE             |     4
         AIR CARE                       |    12
         ANALGESICS                     |    35
         ANTACIDS                       |    23
         APPAREL                        |    36
         APPLES                         |    13
         AUDIO/VIDEO PRODUCTS           |    52
         AUTOMOTIVE PRODUCTS            |    53
         BABYFOOD                       |     2
         BABY FOODS                     |     8
         BABY HBC                       |    32
         BACON                          |    30
         BAG SNACKS                     |    51
         BAKED BREAD/BUNS/ROLLS         |    48
         BAKED SWEET GOODS              |    25
         BAKERY PARTY TRAYS             |    11
         BAKING                         |    15
         BAKING MIXES                   |    34
         BAKING NEEDS                   |    56
         BATH                           |    54
         BATH TISSUES                   |     8
         BATTERIES                      |    11
         BEANS - CANNED GLASS & MW      |    19
         BEEF                           |   375
         BEERS/ALES                     |    88
         BERRIES                        |    15
         BEVERAGE                       |    13
         BIRD SEED                      |     5
         BLEACH                         |    11
         BOOKSTORE                      |   184
         BOTTLE DEPOSITS                |     1
         BOUQUET (NON ROSE)             |     1
         BREAD                          |    36
         BREAKFAST SAUSAGE/SANDWICHES   |    22
         BREAKFAST SWEETS               |    29
         BROCCOLI/CAULIFLOWER           |    10
         BROOMS AND MOPS                |    46
         BULK FOODS                     |     2
         BUTTER                         |     6
         CAKES                          |    29
         CANDLES/ACCESSORIES            |    39
         CANDY - CHECKLANE              |    30
         CANDY - PACKAGED               |   114
         CANNED JUICES                  |    39
         CANNED MILK                    |    12
         CARROTS                        |     7
         CAT FOOD                       |    16
         CAT LITTER                     |    17
         CEREAL/BREAKFAST               |    22
         CHARCOAL AND LIGHTER FLUID     |    20
         CHEESE                         |    40
         CHEESES                        |   268
         CHICKEN                        |   118
         CHICKEN/POULTRY                |    60
         CHIPS&SNACKS                   |    24
         CHRISTMAS  SEASONAL            |   101
         CIGARETTES                     |    11
         CIGARS                         |     7
         CITRUS                         |     6
         COCOA MIXES                    |    12
         COFFEE                         |    39
         COFFEE FILTERS                 |    23
         COFFEE SHOP                    |     5
         COFFEE SHOP SWEET GOODS&RETAIL |     2
         COLD AND FLU                   |    62
         COLD CEREAL                    |    12
         CONDIMENTS                     |    33
         CONDIMENTS/SAUCES              |   124
         CONTINUITIES                   |    16
         CONVENIENT BRKFST/WHLSM SNACKS |    18
         COOKIES                        |    28
         COOKIES/CONES                  |    52
         COOKWARE & BAKEWARE            |    44
         CORN                           |     6
         (CORP USE ONLY)                |     3
         COSMETIC ACCESSORIES           |    16
         COUPON                         |     3
         COUPON/MISC ITEMS              |    18
         COUPONS/STORE & MFG            |    14
         CRACKERS/MISC BKD FD           |    42
         DELI MEATS                     |   266
         DELI SPECIALTIES (RETAIL PK)   |    18
         DELI SUPPLIES                  |     4
         DEODORANTS                     |    29
         DIAPERS & DISPOSABLES          |     4
         DIETARY AID PRODUCTS           |    37
         DINNER MXS:DRY                 |    20
         DINNER SAUSAGE                 |    45
         DISHWASH DETERGENTS            |     9
         DISPOSIBLE FOILWARE            |     1
         DOG FOODS                      |    28
         DOLLAR VALUE PRODUCTS          |     6
         DOMESTIC GOODS                 |    68
         DOMESTIC WINE                  |    99
         DRIED FRUIT                    |    29
         DRY BN/VEG/POTATO/RICE         |    30
         DRY MIX DESSERTS               |    16
         DRY NOODLES/PASTA              |    26
         DRY SAUCES/GRAVY               |    24
         DRY TEA/COFFEE/COCO MIX        |    11
         EASTER                         |    45
         EASTER LILY                    |     1
         EGGS                           |    12
         ELECTRICAL SUPPPLIES           |    10
         ETHNIC PERSONAL CARE           |    36
         EXOTIC GAME/FOWL               |    12
         EYE AND EAR CARE PRODUCTS      |    26
         FACIAL TISS/DNR NAPKIN         |     8
         FALL AND WINTER SEASONAL       |    18
         FAMILY PLANNING                |    15
         FD WRAPS/BAGS/TRSH BG          |    10
         FEMININE HYGIENE               |    19
         FILM AND CAMERA PRODUCTS       |    27
         FIREWORKS                      |     2
         FIRST AID PRODUCTS             |    46
         FITNESS&DIET                   |    27
         FLORAL-ACCESSORIES             |    11
         FLORAL BALLOONS                |    13
         FLORAL-FLOWERING PLANTS        |    54
         FLORAL-FOLIAGE PLANTS          |    37
         FLORAL-FRESH CUT               |    52
         FLORAL- HARD GOODS             |    32
         FLOUR & MEALS                  |    16
         FLUID MILK PRODUCTS            |    31
         FOOT CARE PRODUCTS             |    19
         FRAGRANCES                     |    24
         FROZEN                         |    38
         FROZEN - BOXED(GROCERY)        |    24
         FROZEN BREAD/DOUGH             |    16
         FROZEN CHICKEN                 |     4
         FROZEN MEAT                    |    38
         FROZEN PACKAGE MEAT            |     1
         FROZEN PIE/DESSERTS            |    21
         FROZEN PIZZA                   |    40
         FRUIT - SHELF STABLE           |    16
         FRZN BREAKFAST FOODS           |    18
         FRZN FRUITS                    |     3
         FRZN ICE                       |     9
         FRZN JCE CONC/DRNKS            |     7
         FRZN MEAT/MEAT DINNERS         |    43
         FRZN NOVELTIES/WTR ICE         |    21
         FRZN POTATOES                  |     7
         FRZN SEAFOOD                   |     7
         FRZN VEGETABLE/VEG DSH         |     8
         FUEL                           |     1
         GARDEN CENTER                  |    18
         GIFT & FRUIT BASKETS           |     7
         GLASSES/VISION AIDS            |     8
         GLASSWARE & DINNERWARE         |    32
         GRAPES                         |     6
         GREETING CARDS/WRAP/PARTY SPLY |    27
         HAIR CARE ACCESSORIES          |    18
         HAIR CARE PRODUCTS             |    82
         HALLOWEEN                      |    72
         HAND/BODY/FACIAL PRODUCTS      |    60
         HARDWARE SUPPLIES              |    49
         HEAT/SERVE                     |    30
         HERBS                          |    13
         HISPANIC                       |   170
         HOME FREEZING & CANNING SUPPLY |     9
         HOME FURNISHINGS               |    23
         HOME HEALTH CARE               |     7
         HOSIERY/SOCKS                  |    24
         HOT CEREAL                     |    13
         HOT DOGS                       |    24
         HOUSEHOLD CLEANG NEEDS         |    45
         ICE CREAM/MILK/SHERBTS         |    19
         IMPORTED WINE                  |    69
         INFANT CARE PRODUCTS           |    31
         INFANT FORMULA                 |    10
         INSECTICIDES                   |    23
         IN-STORE PHOTOFINISHING        |    28
         IRONING AND CHEMICALS          |    25
         ISOTONIC DRINKS                |     5
         J-HOOKS                        |   126
         JUICE                          |    23
         KITCHEN GADGETS                |    58
         LAMB                           |    26
         LAUNDRY ADDITIVES              |    14
         LAUNDRY DETERGENTS             |    13
         LAWN AND GARDEN SHOP           |    46
         LAXATIVES                      |    24
         LIQUOR                         |    66
         LONG DISTANCE CALLING CARDS    |     2
         LUNCHMEAT                      |    36
         MAGAZINE                       |    47
         MAKEUP AND TREATMENT           |   101
         MARGARINES                     |    12
         MEAT - MISC                    |    44
         MEAT - SHELF STABLE            |    31
         MEAT SUPPLIES                  |     1
         MELONS                         |     6
         MILK BY-PRODUCTS               |    20
         MISC. DAIRY                    |    29
         MISCELLANEOUS                  |    15
         MISCELLANEOUS(CORP USE ONLY)   |     1
         MISCELLANEOUS CROUTONS         |     1
         MISCELLANEOUS HBC              |     3
         MISC WINE                      |    38
         MOLASSES/SYRUP/PANCAKE MIXS    |    27
         MUSHROOMS                      |     8
         NATURAL HBC                    |    11
         NATURAL VITAMINS               |     5
         NDAIRY/TEAS/JUICE/SOD          |     2
         NEW AGE                        |    12
         NEWSPAPER                      |    21
         NO COMMODITY DESCRIPTION       |   302
         NON-DAIRY BEVERAGES            |    11
         NON EDIBLE PRODUCTS            |     3
         NUTS                           |    26
         OLIVES                         |    14
         ONIONS                         |     5
         ORAL HYGIENE PRODUCTS          |    51
         ORGANICS FRUIT & VEGETABLES    |    44
         OVERNIGHT PHOTOFINISHING       |    10
         PACKAGED NATURAL SNACKS        |     8
         PAPER HOUSEWARES               |    11
         PAPER TOWELS                   |     7
         PARTY TRAYS                    |    12
         PASTA SAUCE                    |    32
         PEARS                          |     2
         PEPPERS-ALL                    |     9
         PERSONAL CARE APPLIANCES       |    11
         PET CARE SUPPLIES              |    56
         PHARMACY                       |     8
         PICKLE/RELISH/PKLD VEG         |    31
         PIES                           |    20
         PKG.SEAFOOD MISC               |     1
         PLASTIC HOUSEWARES             |    32
         PNT BTR/JELLY/JAMS             |    28
         POPCORN                        |    13
         PORK                           |   243
         PORTABLE ELECTRIC APPLIANCES   |    16
         POTATOES                       |    15
         PREPAID WIRELESS&ACCESSORIES   |     9
         PREPARED FOOD                  |    70
         PREPARED/PKGD FOODS            |    37
         PROCESSED                      |    49
         PROD SUPPLIES                  |     2
         PROPANE                        |     1
         PWDR/CRYSTL DRNK MX            |     9
         QUICK SERVICE                  |     3
         REFRGRATD DOUGH PRODUCTS       |    15
         REFRGRATD JUICES/DRNKS         |    18
         REFRIGERATED                   |    38
         RESTRICTED DIET                |     9
         RICE CAKES                     |     5
         ROLLS                          |    24
         ROSES                          |    17
         RW FRESH PROCESSED MEAT        |     5
         SALAD BAR                      |    16
         SALAD MIX                      |    11
         SALADS/DIPS                    |   131
         SALD DRSNG/SNDWCH SPRD         |    36
         SANDWICHES                     |    10
         SEAFOOD-FRESH                  |   161
         SEAFOOD - FROZEN               |   100
         SEAFOOD - MISC                 |    31
         SEAFOOD - SHELF STABLE         |    19
         SEASONAL                       |     8
         SERVICE BEVERAGE               |     4
         SEWING                         |     9
         SHAVING CARE PRODUCTS          |    35
         SHOE CARE                      |     4
         SHORTENING/OIL                 |    33
         SINUS AND ALLERGY              |    30
         SMOKED MEATS                   |   124
         SMOKING CESSATIONS             |     5
         SNACK NUTS                     |    22
         SNACKS                         |    27
         SNKS/CKYS/CRKR/CNDY            |    38
         SOAP - LIQUID & BAR            |    31
         SOFT DRINKS                    |    37
         SOUP                           |    41
         SPICES & EXTRACTS              |    58
         SPORTS MEMORABLILIA            |    26
         SPRING/SUMMER SEASONAL         |   143
         SQUASH                         |     1
         STATIONERY & SCHOOL SUPPLIES   |   154
         STONE FRUIT                    |     3
         SUGARS/SWEETNERS               |    15
         SUNTAN                         |    15
         SUSHI                          |     5
         SWEET GOODS & SNACKS           |    26
         SYRUPS/TOPPINGS                |    10
         TEAS                           |    19
         TICKETS                        |     1
         TOBACCO OTHER                  |    23
         TOMATOES                       |    12
         TOYS                           |     2
         TOYS AND GAMES                 |   166
         TROPICAL FRUIT                 |    11
         TURKEY                         |   131
         UNKNOWN                        |    20
         VALENTINE                      |    44
         VALUE ADDED FRUIT              |    39
         VALUE ADDED VEGETABLES         |    28
         VEAL                           |    14
         VEGETABLES - ALL OTHERS        |    18
         VEGETABLES SALAD               |     6
         VEGETABLES - SHELF STABLE      |    45
         VITAMINS                       |    65
         WAREHOUSE SNACKS               |    25
         WATCHES/CALCULATORS/LOBBY      |    16
         WATER                          |     9
         WATER - CARBONATED/FLVRD DRINK |    46
         YOGURT                         |     9
        (308 rows)

4. List the family ID, age group, composition and size of families who have
   bought corn dogs.

        SELECT DISTINCT household_id, AGE_DESC, HH_COMP_DESC, HOUSEHOLD_SIZE_DESC
        FROM household NATURAL JOIN transaction NATURAL JOIN includes
        WHERE PRODUCT_ID IN
        (SELECT PRODUCT_ID FROM product WHERE SUB_COMMODITY_DESC='CORN DOGS');

         household_id | age_desc |   hh_comp_desc   | household_size_desc
        --------------+----------+------------------+---------------------
                  732 | 55-64    | 2 Adults Kids    | 4
                  546 | 45-54    | 2 Adults No Kids | 2
                 1337 | 45-54    | 2 Adults Kids    | 4
                  392 | 35-44    | 2 Adults Kids    | 3
                 1167 | 35-44    | 2 Adults Kids    | 4
                 1159 | 45-54    | 2 Adults No Kids | 2
                 2328 | 45-54    | 2 Adults No Kids | 2
                  164 | 45-54    | 2 Adults No Kids | 2
                 1759 | 35-44    | 2 Adults Kids    | 5+
                 2085 | 35-44    | 2 Adults No Kids | 2
                 2294 | 45-54    | 2 Adults Kids    | 4
                 2199 | 19-24    | 2 Adults Kids    | 3
                  193 | 45-54    | 2 Adults No Kids | 2
                  979 | 35-44    | 1 Adult Kids     | 2
                 1834 | 45-54    | Unknown          | 5+
                  660 | 55-64    | Unknown          | 2
                   39 | 35-44    | 2 Adults No Kids | 2
                 1166 | 45-54    | 2 Adults Kids    | 3
                 2432 | 19-24    | Single Female    | 1
                 1901 | 45-54    | 2 Adults Kids    | 3
                 1595 | 45-54    | Single Male      | 1
                 2318 | 19-24    | Single Female    | 1
                 1633 | 45-54    | 2 Adults No Kids | 2
                  886 | 35-44    | 2 Adults No Kids | 2
                  583 | 35-44    | Single Female    | 1
                  788 | 25-34    | 1 Adult Kids     | 2
                 1802 | 45-54    | 2 Adults No Kids | 2
                 1425 | 19-24    | 2 Adults Kids    | 3
                 1748 | 45-54    | Single Male      | 1
                 1873 | 35-44    | Single Male      | 1
                 1062 | 65+      | 2 Adults No Kids | 2
                 1042 | 25-34    | 2 Adults No Kids | 2
                  367 | 55-64    | 2 Adults No Kids | 2
                  325 | 35-44    | 2 Adults Kids    | 4
                 1240 | 45-54    | 2 Adults No Kids | 2
                  105 | 25-34    | 1 Adult Kids     | 3
                  432 | 19-24    | Single Female    | 1
                 2341 | 45-54    | Unknown          | 1
                  857 | 65+      | 1 Adult Kids     | 3
                 1707 | 35-44    | Single Female    | 1
                  232 | 35-44    | 2 Adults No Kids | 2
                  113 | 35-44    | 2 Adults Kids    | 4
                 2264 | 45-54    | 2 Adults No Kids | 2
                 2200 | 45-54    | Single Male      | 1
                  675 | 35-44    | 2 Adults No Kids | 2
                  731 | 45-54    | 2 Adults No Kids | 2
                 2295 | 35-44    | Single Male      | 1
                  768 | 35-44    | Single Female    | 2
                 1653 | 35-44    | Single Female    | 1
                  300 | 55-64    | Single Female    | 2
                 1765 | 45-54    | Single Male      | 1
                 1917 | 65+      | Single Female    | 1
                 2012 | 45-54    | Single Male      | 2
                 2467 | 35-44    | 2 Adults Kids    | 3
                  418 | 35-44    | 2 Adults Kids    | 3
                 2097 | 35-44    | 2 Adults Kids    | 5+
                 1634 | 25-34    | 2 Adults Kids    | 5+
                 1795 | 45-54    | 2 Adults No Kids | 2
                  361 | 45-54    | Unknown          | 1
                 2198 | 35-44    | Single Female    | 1
                 1285 | 25-34    | 1 Adult Kids     | 3
                 1845 | 45-54    | 2 Adults Kids    | 3
                  718 | 45-54    | 2 Adults Kids    | 5+
                  518 | 45-54    | 2 Adults No Kids | 2
                  248 | 35-44    | 2 Adults Kids    | 4
                 1995 | 25-34    | 1 Adult Kids     | 3
                 2331 | 35-44    | 2 Adults Kids    | 5+
                  354 | 25-34    | 2 Adults Kids    | 5+
                 1462 | 25-34    | 2 Adults No Kids | 2
                 1229 | 55-64    | 2 Adults No Kids | 2
        (70 rows)

5. List the lowest sales price of cider.

        SELECT MIN(SALES_VALUE/QUANTITY)
        FROM includes
        WHERE PRODUCT_ID IN
        (SELECT PRODUCT_ID FROM product WHERE SUB_COMMODITY_DESC='CIDER');

         min
        ------
         2.49
        (1 row)

6. List the income group(s) where no families in this group have ever bought
   private brand cat food.

        (SELECT INCOME_DESC FROM household)
        EXCEPT
        (SELECT INCOME_DESC
        FROM household NATURAL JOIN transaction NATURAL JOIN includes
        WHERE PRODUCT_ID IN
        (SELECT PRODUCT_ID FROM product WHERE BRAND='Private' AND COMMODITY_DESC='CAT FOOD'));

         income_desc
        -------------
         150-174K
         175-199K
         200-249K
        (3 rows)

    * Old question: List the income of families that have bought private brand
      dog foods.

    * Reason: There's no commodity category call 'DOG FOOD' in product table so
      I used 'CAT FOOD' and made a little change to the original query.

7. List household id, composition and number of kids for families that have
   bought both batteries and baseballs.

        (SELECT household_id, hh_comp_desc, kid_category_desc
        FROM transaction NATURAL JOIN includes NATURAL JOIN product NATURAL JOIN household
        WHERE SUB_COMMODITY_DESC = 'BATTERIES')
        INTERSECT
        (SELECT household_id, hh_comp_desc, kid_category_desc
        FROM transaction NATURAL JOIN includes NATURAL JOIN product NATURAL JOIN household
        WHERE SUB_COMMODITY_DESC = 'BASEBALL');

         household_id |   hh_comp_desc   | kid_category_desc
        --------------+------------------+-------------------
                 1804 | 2 Adults No Kids | None/Unknown
        (1 row)

8. Which state has the most stores?

        WITH x AS
        (SELECT state, COUNT(*) AS cnt
        FROM store
        GROUP BY state)

        SELECT * FROM x
        WHERE cnt = (SELECT MAX(cnt) FROM x);

         state | cnt
        -------+-----
         WA    | 205
        (1 row)

9. List stores that sell 1 gallon chocolate milk.

        SELECT DISTINCT store_id
        FROM product NATURAL JOIN sells
        WHERE SUB_COMMODITY_DESC='CHOCOLATE MILK' AND CURR_SIZE_OF_PRODUCT='1 GALLON';

         store_id
        ----------
              286
              288
              289
              292
              293
              295
              296
              297
              298
              299
              300
              304
              306
              309
              310
              311
              313
              315
              316
              317
              318
              319
              320
              321
              322
              323
              324
              327
              329
              330
              333
              334
              335
              337
              338
              339
              340
              341
              343
              345
              346
              352
              354
              355
              356
              358
              359
              360
              361
              362
              363
              364
              365
              366
              367
              368
              369
              370
              372
              374
              375
              379
              380
              381
              382
              384
              388
              389
              391
              396
              400
              401
              402
              403
              404
              406
              408
              410
              412
              414
              420
              421
              422
              424
              427
              429
              432
              433
              436
              438
              439
              441
              442
              443
              445
              446
              447
              448
              450
            31401
            31582
            31642
            31742
            31762
            31782
            31862
            32004
        (107 rows)

10. What's the size of families that buy the most 32 OZ yogurt?

        WITH x AS
        (SELECT HOUSEHOLD_SIZE_DESC, SUM(quantity) AS sum
        FROM household NATURAL JOIN transaction NATURAL JOIN includes NATURAL JOIN product
        WHERE SUB_COMMODITY_DESC='YOGURT' AND CURR_SIZE_OF_PRODUCT='32 OZ'
        GROUP BY HOUSEHOLD_SIZE_DESC)

        SELECT household_size_desc FROM x WHERE sum =
        (SELECT MAX(sum) FROM x);

         household_size_desc
        ---------------------
         4
        (1 row)

11. List id(s) of stores in Oregon that have the most age 65+ patrons.

        WITH x AS
        (SELECT store_id, COUNT(*) AS cnt
        FROM household NATURAL JOIN shops_at NATURAL JOIN store
        WHERE  AGE_DESC = '65+' AND state = 'OR'
        GROUP BY store_id)

        SELECT store_id FROM x WHERE cnt =
        (SELECT MAX(cnt) FROM x);

         store_id
        ----------
              319
        (1 row)

12. Find the age group that spent the most on laundry detergent.

        WITH x AS
        (SELECT AGE_DESC, SUM(SALES_VALUE) AS sum
        FROM product NATURAL JOIN includes NATURAL JOIN transaction NATURAL JOIN household
        WHERE COMMODITY_DESC='LAUNDRY DETERGENTS'
        GROUP BY AGE_DESC)

        SELECT AGE_DESC FROM x WHERE sum =
        (SELECT MAX(sum) FROM x);

         age_desc
        ----------
         45-54
        (1 row)

13. Find the number of stores in Oregon that have sold anything relating
    garden.

        SELECT COUNT(DISTINCT store_id)
        FROM product NATURAL JOIN includes NATURAL JOIN transaction NATURAL JOIN store
        WHERE SUB_COMMODITY_DESC LIKE '%GARDEN%' AND state = 'OR';

         count
        -------
            30
        (1 row)


    * Old question: Find the number of stores in Oregon that sell anything
      including the key word “Christmas”.

    * Reason: Changed the search keyword to "GARDEN" because I found that
      "Christmas" is not the best choice for this query: although there's
      products containing the keywords "CHRISTMAS" or "XMAS" in product table,
      there's no actual transaction happened during the period data are
      collected which returns an empty query result.

14. Which income group has the most families with more than 3 kids.

        WITH x AS
        (SELECT INCOME_DESC, COUNT(*) AS cnt
        FROM household
        WHERE KID_CATEGORY_DESC='3+'
        GROUP BY INCOME_DESC)

        SELECT * FROM x WHERE cnt =
        (SELECT MAX(cnt) FROM x);

         income_desc | cnt
        -------------+-----
         35-49K      |  16
        (1 row)

15. The distribution of expense on medicine in Oregon among age groups, in
    descending order.

        SELECT age_desc, SUM(sales_value)
        FROM household NATURAL JOIN transaction NATURAL JOIN includes NATURAL JOIN product NATURAL JOIN store
        WHERE department = 'DRUG GM' AND state = 'OR'
        GROUP BY age_desc
        ORDER BY SUM(sales_value) DESC;

         age_desc |       sum
        ----------+------------------
         45-54    | 5881.14999999991
         35-44    | 4984.68999999996
         25-34    | 3042.96999999999
         65+      |          1846.66
         19-24    |          1629.48
         55-64    |          1424.16
        (6 rows)


    * Old question: Families of which age group bought the most expensive
      televisions in Oregon.

    * Reason: The original query is similar to query #5 asking the lowest sales
      price of cider and I wanted to maintain a variety of my query sets so
      that I get more information from the source data.

16. How many families have bought the same product from different stores.

        SELECT COUNT(*)
        FROM (SELECT DISTINCT household_id
        FROM transaction NATURAL JOIN includes
        GROUP BY household_id, product_id
        HAVING COUNT(store_id) > 1
        ) AS temp;

         count
        -------
           741
        (1 row)

17. What products have store #368 sold in week 20 between 11:00am to 1:00pm.

        SELECT DISTINCT product_id, COMMODITY_DESC, SUB_COMMODITY_DESC
        FROM sells NATURAL JOIN includes NATURAL JOIN transaction NATURAL JOIN product
        WHERE store_id = 368 AND week_no = 20 AND trans_time > 1100 AND trans_time < 1400;

         product_id |        commodity_desc        |       sub_commodity_desc
        ------------+------------------------------+--------------------------------
             820612 | SOFT DRINKS                  | SOFT DRINKS 20PK&24PK CAN CARB
             828114 | FRZN MEAT/MEAT DINNERS       | FRZN SS PREMIUM ENTREES/DNRS/N
             835080 | FRZN MEAT/MEAT DINNERS       | FRZN SS PREMIUM ENTREES/DNRS/N
             839162 | HEAT/SERVE                   | ENTREES
             839849 | SOFT DRINKS                  | SFT DRNK 2 LITER BTL CARB INCL
             847982 | SALAD MIX                    | REGULAR GARDEN
             849559 | WAREHOUSE SNACKS             | SNACK MIX
             858424 | BAKING MIXES                 | LAYER CAKE MIX
             860776 | VEGETABLES - ALL OTHERS      | CUCUMBERS
             862349 | FLUID MILK PRODUCTS          | FLUID MILK WHITE ONLY
             864774 | CONDIMENTS/SAUCES            | SALAD MUSTARD
             868319 | WAREHOUSE SNACKS             | CANISTER POTATO/TORT CHIPS
             871652 | PICKLE/RELISH/PKLD VEG       | PICKLD VEG PEPPERS ETC
             873203 | CHEESE                       | SHREDDED CHEESE
             879045 | COLD CEREAL                  | ALL FAMILY CEREAL
             882857 | MAKEUP AND TREATMENT         | IMPLEMENTS SETS
             888438 | BAKED BREAD/BUNS/ROLLS       | ENGLISH MUFFINS/WAFFLES
             893810 | MELONS                       | CANTALOUPE WHOLE
             898958 | STONE FRUIT                  | PLUMS
             904995 | APPLES                       | APPLES GALA (BULK&BAG)
             905539 | DINNER SAUSAGE               | SMOKED/COOKED
             908531 | FLUID MILK PRODUCTS          | CHOCOLATE MILK
             921744 | BAG SNACKS                   | TORTILLA/NACHO CHIPS
             923670 | BREAKFAST SAUSAGE/SANDWICHES | PATTIES - RAW
             923746 | EGGS                         | EGGS - LARGE
             923789 | COLD CEREAL                  | ADULT CEREAL
             928749 | ICE CREAM/MILK/SHERBTS       | PREMIUM
             932503 | CHEESE                       | SHREDDED CHEESE
             941295 | DELI MEATS                   | MEAT:HAM BULK
             948232 | BREAKFAST SAUSAGE/SANDWICHES | SANDWICHES/BISCUITS/GRAVY
             956200 | FRZN MEAT/MEAT DINNERS       | FRZN SS PREMIUM ENTREES/DNRS/N
             959074 | HAIR CARE PRODUCTS           | HAIR CONDITIONERS AND RINSES
             961554 | CARROTS                      | CARROTS MINI PEELED
             962970 | BAKED BREAD/BUNS/ROLLS       | ENGLISH MUFFINS/WAFFLES
             964857 | INFANT CARE PRODUCTS         | FEEDING ACCESSORIES BOTTLES
             968215 | TROPICAL FRUIT               | KIWI FRUIT
             972515 | FRZN MEAT/MEAT DINNERS       | FRZN SS PREMIUM ENTREES/DNRS/N
             995242 | FLUID MILK PRODUCTS          | FLUID MILK WHITE ONLY
            1003188 | CANNED JUICES                | APPLE JUICE & CIDER (OVER 50%
            1013321 | VEGETABLES - SHELF STABLE    | BEANS GREEN: FS/WHL/CUT
            1016800 | SOFT DRINKS                  | SOFT DRINKS 12/18&15PK CAN CAR
            1038443 | SOFT DRINKS                  | SOFT DRINKS 12/18&15PK CAN CAR
            1039441 | ICE CREAM/MILK/SHERBTS       | PREMIUM
            1054281 | ICE CREAM/MILK/SHERBTS       | PREMIUM
            1058021 | BAKED SWEET GOODS            | SWEET GOODS - FULL SIZE
            1065038 | SALD DRSNG/SNDWCH SPRD       | POURABLE SALAD DRESSINGS
            1066725 | FRUIT - SHELF STABLE         | FRUIT BOWL AND CUPS
            1068719 | SUGARS/SWEETNERS             | SUGAR
            1071939 | CHEESE                       | CREAM CHEESE
            1079484 | WAREHOUSE SNACKS             | SNACK MIX
            1080841 | REFRGRATD DOUGH PRODUCTS     | REFRIGERATED SPECILATY ROLLS
            1081177 | TOMATOES                     | TOMATOES VINE RIPE BULK
            1082185 | TROPICAL FRUIT               | BANANAS
            1088628 | PASTA SAUCE                  | MAINSTREAM
            1093730 | BEANS - CANNED GLASS & MW    | PREPARED BEANS - BAKED W/PORK
            1098066 | BAKED BREAD/BUNS/ROLLS       | HOT DOG BUNS
            1099089 | MEAT - MISC                  | BREAST - BONELESS(IQF)
            1099446 | WAREHOUSE SNACKS             | SNACK MIX
            1124201 | ICE CREAM/MILK/SHERBTS       | QUARTS
            1131550 | FRZN BREAKFAST FOODS         | WAFFLES/PANCAKES/FRENCH TOAST
            1139336 | CRACKERS/MISC BKD FD         | SNACK CRACKERS
            1139827 | CHEESE                       | SHREDDED CHEESE
            5569230 | SOFT DRINKS                  | SOFT DRINKS 12/18&15PK CAN CAR
            5584027 | YOGURT                       | YOGURT NOT MULTI-PACKS
            5587837 | YOGURT                       | YOGURT NOT MULTI-PACKS
            5589985 | SWEET GOODS & SNACKS         | SW GDS: BROWNIE/BAR COOKIE
            7431989 | CANDY - CHECKLANE            | CHEWING GUM
            8352928 | DIETARY AID PRODUCTS         | DIET CNTRL LIQS NUTRITIONAL
            9526268 | ORAL HYGIENE PRODUCTS        | TOOTHPASTE
            9527066 | BAG SNACKS                   | POTATO CHIPS
            9527160 | BAG SNACKS                   | POTATO CHIPS
            9553131 | HAIR CARE PRODUCTS           | HAIR CONDITIONERS AND RINSES
            9553343 | HAIR CARE PRODUCTS           | SHAMPOO
           10204822 | CANDY - PACKAGED             | CANDY BARS (MULTI PACK)
           10457591 | REFRGRATD DOUGH PRODUCTS     | REFRIGERATED SPECILATY ROLLS
        (75 rows)

    * Old question: Stores in which state sell the most charcoals.

    * Reason: Replaced the old one because it was similar to query #10 asking
      the families that buy the most 32 oz yogurt.  The new query covers the
      'trans_time' and 'week_no' attributes of transaction table which gives a
      better coverage of the schema.

18. List information of families who have never bought anything from drug
    department(but have bought from other departments).

        (SELECT household_id, AGE_DESC, MARITAL_STATUS_CODE, INCOME_DESC, HOMEOWNER_DESC, HH_COMP_DESC, HOUSEHOLD_SIZE_DESC, KID_CATEGORY_DESC
        FROM household NATURAL JOIN transaction NATURAL JOIN includes NATURAL JOIN product)
        EXCEPT
        (SELECT household_id, AGE_DESC, MARITAL_STATUS_CODE, INCOME_DESC, HOMEOWNER_DESC, HH_COMP_DESC, HOUSEHOLD_SIZE_DESC, KID_CATEGORY_DESC
        FROM household NATURAL JOIN transaction NATURAL JOIN includes NATURAL JOIN product
        WHERE department='DRUG GM');

         household_id | age_desc | marital_status_code | income_desc | homeowner_desc |   hh_comp_desc   | household_size_desc | kid_category_desc
        --------------+----------+---------------------+-------------+----------------+------------------+---------------------+-------------------
                 2181 | 55-64    | Married             | 100-124K    | Homeowner      | 2 Adults Kids    | 3                   | 1
                 1962 | 65+      | Unknown             | 25-34K      | Unknown        | 2 Adults No Kids | 2                   | None/Unknown
                 1894 | 35-44    | Married             | 100-124K    | Homeowner      | 2 Adults No Kids | 2                   | None/Unknown
                 2173 | 25-34    | Unknown             | 35-49K      | Unknown        | 2 Adults No Kids | 2                   | None/Unknown
                 1186 | 45-54    | Married             | 50-74K      | Homeowner      | 1 Adult Kids     | 5+                  | 3+
                 1412 | 45-54    | Married             | 50-74K      | Homeowner      | 2 Adults Kids    | 3                   | 1
                  770 | 25-34    | Married             | 50-74K      | Homeowner      | 2 Adults Kids    | 5+                  | 3+
                 1454 | 65+      | Married             | 75-99K      | Homeowner      | 2 Adults No Kids | 2                   | None/Unknown
                  309 | 25-34    | Married             | 75-99K      | Homeowner      | Single Male      | 2                   | None/Unknown
                 1321 | 19-24    | Unknown             | 35-49K      | Unknown        | 2 Adults No Kids | 2                   | None/Unknown
                 1926 | 25-34    | Unknown             | 35-49K      | Homeowner      | 2 Adults No Kids | 2                   | None/Unknown
                 1394 | 45-54    | Married             | 50-74K      | Homeowner      | 2 Adults No Kids | 2                   | None/Unknown
                   46 | 45-54    | Married             | 150-174K    | Homeowner      | 2 Adults Kids    | 5+                  | 3+
                 1689 | 45-54    | Unknown             | 50-74K      | Unknown        | Unknown          | 1                   | None/Unknown
                 1566 | 35-44    | Unknown             | 75-99K      | Homeowner      | 2 Adults No Kids | 2                   | None/Unknown
                 1650 | 45-54    | Married             | 75-99K      | Homeowner      | 2 Adults Kids    | 3                   | 1
                  540 | 25-34    | Single              | 50-74K      | Homeowner      | Single Female    | 1                   | None/Unknown
                 1082 | 45-54    | Unknown             | 35-49K      | Unknown        | Single Male      | 1                   | None/Unknown
                 1709 | 25-34    | Married             | 35-49K      | Unknown        | 2 Adults Kids    | 3                   | 1
                  857 | 65+      | Married             | 35-49K      | Homeowner      | 1 Adult Kids     | 3                   | 1
                  346 | 65+      | Married             | 50-74K      | Homeowner      | 2 Adults No Kids | 2                   | None/Unknown
                 2119 | 25-34    | Unknown             | 50-74K      | Homeowner      | Single Male      | 1                   | None/Unknown
                  675 | 35-44    | Unknown             | 50-74K      | Unknown        | 2 Adults No Kids | 2                   | None/Unknown
                  127 | 45-54    | Unknown             | 25-34K      | Unknown        | Single Female    | 1                   | None/Unknown
                 1135 | 45-54    | Unknown             | 50-74K      | Unknown        | Unknown          | 2                   | 1
                 2154 | 45-54    | Married             | 35-49K      | Homeowner      | 2 Adults No Kids | 2                   | None/Unknown
                 2393 | 35-44    | Unknown             | 75-99K      | Homeowner      | Single Male      | 1                   | None/Unknown
                 1759 | 35-44    | Unknown             | 35-49K      | Unknown        | 2 Adults Kids    | 5+                  | 3+
                 2172 | 25-34    | Married             | 100-124K    | Homeowner      | 2 Adults No Kids | 2                   | None/Unknown
        (29 rows)

    * Old question: List homeowner description and size of households that have
      been patronizing more than three stores.

    * Reason: The old query was not selective enough and in turn, didn't give
      me much information as there's hundreds of families have bought from more
      than 3 stores and they distribute in every age, income, and household
      composition group.  I replaced it with this new query which is more
      selective and meaningful.

19. List the household information, transaction time and commodity description
    of families who have ONLY bought products from DELI department.

        SELECT DISTINCT household_id, AGE_DESC, MARITAL_STATUS_CODE, INCOME_DESC, HOMEOWNER_DESC, HH_COMP_DESC, HOUSEHOLD_SIZE_DESC, KID_CATEGORY_DESC, TRANS_TIME, COMMODITY_DESC, SUB_COMMODITY_DESC
        FROM household NATURAL JOIN transaction NATURAL JOIN includes NATURAL JOIN product
        WHERE department='DELI' AND household_id IN
        (SELECT household_id
        FROM household NATURAL JOIN transaction NATURAL JOIN includes NATURAL JOIN product
        GROUP BY HOUSEHOLD_ID
        HAVING COUNT(DISTINCT DEPARTMENT) = 1);

         household_id | age_desc | marital_status_code | income_desc | homeowner_desc |   hh_comp_desc   | household_size_desc | kid_category_desc | trans_time | commodity_desc  |    sub_commodity_desc
        --------------+----------+---------------------+-------------+----------------+------------------+---------------------+-------------------+------------+-----------------+---------------------------
                  346 | 65+      | Married             | 50-74K      | Homeowner      | 2 Adults No Kids | 2                   | None/Unknown      |       1145 | CHICKEN/POULTRY | CHIX:FRD 8PC/CUT UP (HOT)
                  346 | 65+      | Married             | 50-74K      | Homeowner      | 2 Adults No Kids | 2                   | None/Unknown      |       1149 | CHICKEN/POULTRY | CHIX:FRD 8PC/CUT UP (HOT)
                  346 | 65+      | Married             | 50-74K      | Homeowner      | 2 Adults No Kids | 2                   | None/Unknown      |       1155 | CHICKEN/POULTRY | CHIX:FRD 8PC/CUT UP (HOT)
        (3 rows)

    * Old question: List all the stores that sell adult cereal.

    * Reason: Replaced a simple query with a query that covers the 'department'
      attribute in product table and reveals more about the source data.

20. How likely is it for a household with kid to buy kid's cereal, compare to
    average?

        with kids_cereal_product as
        (select product_id from product
        where sub_commodity_desc like '%KID%CEREAL%'),

        kids_household as
        (select household_id from household
        where hh_comp_desc in ('1 Adult Kids', '2 Adults Kids')),

        kids_household_kids_cereal_index as
        (select
                (
                    select count(*) from transaction natural join includes
                    where household_id in (select * from kids_household) and product_id in (select * from kids_cereal_product)
                ) / (select count(*) :: float from kids_household) x
        ),

        all_household_kids_cereal_index as
        (select
                (
                    select count(*) from transaction natural join includes
                    where product_id in (select * from kids_cereal_product)
                ) / (select count(*) :: float from household) x
        )

        select (select * from kids_household_kids_cereal_index) / (select * from all_household_kids_cereal_index);

             ?column?
        ------------------
         1.43181333901556
        (1 row)


    * Old question: What's the major part of income distribution for families
      who are homeowners.

    * Reason: I wanted to challenge a complicated query that makes more sense
      statistically and simulate statistic use of database.  If I want to
      deliver ads to potential customers, this query would help me to identify
      my target market.

### Listing of the Contents of All Tables

Please click [this link](https://github.com/Fan-Zhang/CS586-Project/tree/master/table-contents) to view files on GitHub.

