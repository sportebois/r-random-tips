R for dataviz with Redshift
================

Interact with Redshift using SQL
--------------------------------

Whenever in doubt, don't forget Redshift has its own flavor of SQL, and you can always check your functions in [Amazon Redshift SQL](http://docs.aws.amazon.com/redshift/latest/dg/c_redshift-sql.html) documentation.

Software used:

-   [Amazon Redshift](https://aws.amazon.com/redshift/)
-   [RStudio](https://www.rstudio.com/)

### Connecting to Redshift

First step is to setup some parameters, to prepare your connection to the database.

``` r
Sys.setenv(REDSHIFT_USER = "user", REDSHIFT_PASSWD = "password") # use your real credentials, if not already set in the environment

redshiftUser <- Sys.getenv("REDSHIFT_USER")
redshiftPassword <- Sys.getenv("REDSHIFT_PASSWD")
redshiftPort <- 5439 # default port
redshiftHost <- "clusterName.abc01234.us-east-1.redshift.amazonaws.com" # get this string from your AWS console (or ask a colleague)
dbName <- "myDb"
dbTable <- "dbTable"
```

Then you will need osme libraries to get the job done. You can either install these using the `install.packages(..)` command or using the **Packages** tab of RStudio.

``` r
library(GetoptLong)  # Use for qq and qqcat method, usful for easy templating
library(RPostgreSQL) # The PostgreSQL driver to connect to Redshfit
library(readr)       # To load sql template files
library(magrittr)    # To get nice pipes
library(plyr)        # Data transformation
library(dplyr)       # More data wrangling
library(tidyr)       # Data transformation continuer
library(lubridate)   # To work with dates
library(ggplot2)     # We want plots
library(scales)      # Lots of useful scaling funcs, life pretty_breaks
library(viridis)     # The best graph color palette
```

And now you can copy/paste the following helper to handle the boilerplate:

``` r
#' Load a sql template, parse it with the variables set in the current environment 
#' and execute this in the current Redhshift configuration set
#' @param sqlTemplatePath string relative filepath of the sql template file
#' @param command string, 'get' when you want to fetch data, 'send' for simple executions
#' @param debugSQL logical Dump the SQL string (after variables injection in the template) if TRUE
#' @param execSql logical To let you do dry-run if FALSE
#' @output data frame if \code{command} is "get"
executeSqlTemplate <- function(sqlTemplatePath, command = "get", debugSQL = FALSE, execSql = TRUE) {
    template <- readr::read_file(sqlTemplatePath)
    executeSql(template, command, debugSQL = debugSQL, execSql = execSql)
}


#' Parse a SQL string with the variables set in the current environment 
#' and execute this in the current Redhshift configuration set
#' @param sql string the SQL query (with placeholder for variable injection if needed)
#' @param command string, 'get' when you want to fetch data, 'send' for simple executions
#' @param debugSQL logical Dump the SQL string (after variables injection in the template) if TRUE
#' @param execSql logical To let you do dry-run if FALSE
#' @output data frame if \code{command} is "get"
executeSql <- function(sql, command = "get", debugSQL = FALSE, execSql = TRUE) {
    sqlQuery <- qq(sql)
    if (debugSQL) qqcat(sqlQuery)
    
    qqcat(strsplit(sqlQuery, "\n")[[1]][1])
    
    results <- if (execSql && command == "get") {
        dbGetQuery(redshift, sqlQuery)
    } else if (execSql && command == "send") {
        dbSendQuery(redshift, sqlQuery)
    } else {
        NA
    }
    results
}

#' Initalize the Redshift connection, call it before sending queries to set a 
#' current connection in the local environment
#' @output DBI connection
getRedshiftConn <- function() {
    driver <- dbDriver("PostgreSQL")
    conn <- dbConnect(driver, 
                      host = redshiftHost,
                      port = redshiftPort,
                      dbname = dbName,
                      user = redshiftUser,
                      password = redshiftPassword)
    conn
}

#' Utility to ask th eDBI driver to correctly escape your string, usefull to avoid the single-double-quotes mess
#' @param str string the sql string to escape
#' @param conn DBI connection the current connection
#' @output string
escapeSql <- function(str, conn) {
    ifelse(is.na(str), 
           NA, 
           dbQuoteString(conn, str))
}


#' Close your DBI connection
#' @param conn DBI connection the current connection
closeRedshiftConn <- function(conn) {
    dbDisconnect(conn) # close connection
}
```

### Sending simple SQL queries

Ok, now you're set, you can start sending queries!

``` r
redshift <- getRedshiftConn()
dbgSql <- "SELECT year, COUNT(*) as items,
COALESCE(sum(CASE WHEN pr THEN 1 ELSE 0 END),0) as pr,
COALESCE(sum(CASE WHEN oa THEN 1 ELSE 0 END),0) as oa,
FROM myTable
WHERE publication_year BETWEEN 2005 AND 2016
GROUP BY year ORDER BY year ASC;"
testSQL <- executeSql(dbgSql)
```

### Using templates

The `qq` function let us use simple templating. Variables are looked for in the current environment, and ar identified with the `@{variableName}` syntax.

``` r
redshift <- getRedshiftConn()
yearMin <- 2005
yearMax <- 2016
dbgSql <- "SELECT year, COUNT(*) as items,
COALESCE(sum(CASE WHEN pr THEN 1 ELSE 0 END),0) as pr,
COALESCE(sum(CASE WHEN oa THEN 1 ELSE 0 END),0) as oa,
FROM @{dbTable}
WHERE publication_year BETWEEN @{yearMin} AND @{yearMax}
GROUP BY year ORDER BY year ASC;"
testSample <- executeSql(dbgSql)
```

And this SQL template can be saved to a file, and loaded on-demand like this:

``` r
redshift <- getRedshiftConn()
yearMin <- 2005
yearMax <- 2016
testTemplate <- executeSqlTemplate("path/to/template.sql")
```

Using dplyR DSL rather than SQL
-------------------------------

If you don't have some SQL to start with, and you prefer a more abstract DSL, the dplyR package provides you a nice way to interact with your data.

To do so, your data source is no longer the PostreSQL'S DBI driver, but dplyR's own `src_postgres` and `tbl` functions to get reference to the DB and its tables, as illustrated below.

``` r
redshift <- src_postgres(dbName,
                        host = redshiftHost, port = redshiftPort,
                        user = redshiftUser,  password = redshiftPassword)
# create table reference
tblRef <- tbl(redshift, dbTable)
```

DplyR's `glimpse` data overview is also very handy to have a quick overview of the table you're connecting to:

``` r
glimpse(tblRef)
```

``` r
Observations: 96,158,229
Variables: 27
$ id                      (dbl) 3.753801e+12, 3.753801e+12, 3.753801e+12, 3.753801e+12, 7.241315e+12, 7.241315e+12, 7.241315e+12, 7.24...
$ sha1                    (chr) "47837d60b3bb8e8a2c36e83de7b502b4a7204ecf", "478972d4defcfa2e41440b8cc6841989da79a11b", "4791cc15da71a...
$ indexed                 (lgl) FALSE, FALSE, FALSE, FALSE, FALSE, FALSE, FALSE, FALSE, TRUE, FALSE, FALSE, FALSE, TRUE, TRUE, FALSE, ...
$ publication_year        (int) NA, 2010, 2008, 2008, NA, 2001, 2007, 1975, 2010, 2006, 1976, 1986, 1992, 2013, 1973, 2013, 1958, 1866...
$ language                (chr) NA, "jpn", "eng", "eng", NA, "eng", NA, "eng#dut", "ger", "eng", NA, "eng", "eng#por", "eng", "eng", N...
...
```

Ok, you have a working connection to the database. Now, let start to play with the DSL. DplyR - and it's companion magrittR - provide a pipe operator `%>%` (magrittr also gives you a Tee operator `%T>%`, an assignation-and-pipe `%<>%`, and a pipe without data `%$%`... but start with standard pipes and then you'll be eager to use more!) The basic idea is that rather than combining methods calls in parenthesis pyramidal-like structures, the pipe will fill the first argument of the right-side of the pipe operator with the output of the left side of the pipe. For example, `mutate(filter(myData, someField > 10), newField = someField + 2)` becomes `myData %>% filter(someField > 10) %>% mutate(newField = someField + 2)`.

DplyR and tidyR give you a lot of function to filter, arrange, mutate, filter, select, and summarize your data, and a quick look at the 2 pages from the [Data wrangling cheat sheet](http://www.rstudio.com/wp-content/uploads/2015/02/data-wrangling-cheatsheet.pdf) is probably for best and fastest way to get yoru hands on it.

Let's try it. In the example below, we're requesting againt a 90M observations x 27 variables table. Keep in mind that the DSL is smart enough to build the requests and return you only the results, ie you don't have to load all that data in your local memory.... unless you're asking for it!

``` r
yearRepartition <- tblRef %>% 
    filter(publication_year >= 2005 & publication_year < 2016) %>% 
    group_by(publication_year) %>% 
    summarise(items_cnt = n()) %>% 
    arrange(publication_year)
knitr::kable(yearRepartition)
```

|  publication\_year|  items\_cnt|
|------------------:|-----------:|
|               2005|     2340594|
|               2006|     2566776|
|               2007|     2644248|
|               2008|     3101536|
|               2009|     3567953|
|               2010|     3664628|
|               2011|     3538337|
|               2012|     3416758|
|               2013|     2961184|
|               2014|     2647337|
|               2015|     1954115|

Here the summary is simple, just a count, but you have access to all your favorite and baked-in tools in R: santard deviation, variance, ...

``` r
dslSample2 <- tblRef %>% 
    filter(publication_year >= 2005 & publication_year < 2016) %>% 
    group_by(domain, publication_year) %>% 
    mutate(indexed_val = if (indexed) 1 else 0) %>%   # Convert logical to int, ifelse(,,) cannot be run on DB
    summarize(items_cnt = n(), indexed_var = var(indexed_val), publication_year = publication_year) %>% 
    mutate(items_vmin = items_cnt * (1 - indexed_var), 
           items_vmax = items_cnt * (1 + indexed_var)) %>% 
    arrange(domain, publication_year)

knitr::kable(dslSample2)
```

| domain                     |  publication\_year|  items\_cnt|  indexed\_var|  publication\_year|  items\_vmin|  items\_vmax|
|:---------------------------|------------------:|-----------:|-------------:|------------------:|------------:|------------:|
|                            |               2005|      441067|     0.0596484|               2005|    414758.04|    467375.96|
|                            |               2006|      507208|     0.0688693|               2006|    472276.92|    542139.08|
|                            |               2007|      528619|     0.0777789|               2007|    487503.60|    569734.40|
|                            |               2008|      677091|     0.0841341|               2008|    620124.56|    734057.44|
|                            |               2009|      873083|     0.0905327|               2009|    794040.47|    952125.53|
|                            |               2010|     1159260|     0.0812300|               2010|   1065093.35|   1253426.65|
|                            |               2011|     1007702|     0.0931519|               2011|    913832.68|   1101571.32|
|                            |               2012|      859820|     0.1091923|               2012|    765934.25|    953705.75|
|                            |               2013|      781115|     0.1418689|               2013|    670299.04|    891930.96|
|                            |               2014|      614841|     0.1698104|               2014|    510434.58|    719247.42|
|                            |               2015|      409470|     0.1515754|               2015|    347404.41|    471535.59|
| Applied Sciences           |               2005|      291723|     0.1338878|               2005|    252664.84|    330781.16|
| Applied Sciences           |               2006|      323750|     0.1409118|               2006|    278129.81|    369370.19|
| Applied Sciences           |               2007|      341452|     0.1539352|               2007|    288890.53|    394013.47|
| Applied Sciences           |               2008|      388058|     0.1714030|               2008|    321543.71|    454572.29|
| Applied Sciences           |               2009|      415391|     0.1577224|               2009|    349874.52|    480907.48|
| Applied Sciences           |               2010|      433788|     0.1597504|               2010|    364490.20|    503085.80|
| Applied Sciences           |               2011|      477608|     0.1451613|               2011|    408277.81|    546938.19|
| Applied Sciences           |               2012|      486657|     0.1431873|               2012|    416973.90|    556340.10|
| Applied Sciences           |               2013|      415829|     0.1582944|               2013|    350005.61|    481652.39|
| Applied Sciences           |               2014|      396474|     0.1642207|               2014|    331364.76|    461583.24|
| Applied Sciences           |               2015|      329060|     0.1344322|               2015|    284823.73|    373296.27|
| Arts & Humanities          |               2005|       72438|     0.0518149|               2005|     68684.64|     76191.36|
| Arts & Humanities          |               2006|       76349|     0.0517402|               2006|     72398.69|     80299.31|
| Arts & Humanities          |               2007|       80370|     0.0560151|               2007|     75868.07|     84871.93|
| Arts & Humanities          |               2008|      105568|     0.0498432|               2008|    100306.15|    110829.85|
| Arts & Humanities          |               2009|      148889|     0.0391377|               2009|    143061.83|    154716.17|
| Arts & Humanities          |               2010|      104445|     0.0527802|               2010|     98932.38|    109957.62|
| Arts & Humanities          |               2011|      105707|     0.0509311|               2011|    100323.23|    111090.77|
| Arts & Humanities          |               2012|      116401|     0.0444967|               2012|    111221.54|    121580.46|
| Arts & Humanities          |               2013|       62339|     0.0661380|               2013|     58216.02|     66461.98|
| Arts & Humanities          |               2014|       41576|     0.0902215|               2014|     37824.95|     45327.05|
| Arts & Humanities          |               2015|       26941|     0.0833400|               2015|     24695.74|     29186.26|
| Economic & Social Sciences |               2005|      116459|     0.1138645|               2005|    103198.45|    129719.55|
| Economic & Social Sciences |               2006|      122531|     0.1244334|               2006|    107284.05|    137777.95|
| Economic & Social Sciences |               2007|      135494|     0.1285109|               2007|    118081.54|    152906.46|
| Economic & Social Sciences |               2008|      151795|     0.1236797|               2008|    133021.04|    170568.96|
| Economic & Social Sciences |               2009|      225053|     0.0933937|               2009|    204034.47|    246071.53|
| Economic & Social Sciences |               2010|      160517|     0.1359094|               2010|    138701.22|    182332.78|
| Economic & Social Sciences |               2011|      154631|     0.1283700|               2011|    134781.01|    174480.99|
| Economic & Social Sciences |               2012|      155028|     0.1220170|               2012|    136111.96|    173944.04|
| Economic & Social Sciences |               2013|      124027|     0.1291264|               2013|    108011.84|    140042.16|
| Economic & Social Sciences |               2014|      102761|     0.1393385|               2014|     88442.43|    117079.57|
| Economic & Social Sciences |               2015|       81754|     0.1122218|               2015|     72579.42|     90928.58|
| General                    |               2005|       26160|     0.1779548|               2005|     21504.70|     30815.30|
| General                    |               2006|       35309|     0.1568713|               2006|     29770.03|     40847.97|
| General                    |               2007|       27751|     0.2064086|               2007|     22022.95|     33479.05|
| General                    |               2008|       33938|     0.2121868|               2008|     26736.80|     41139.20|
| General                    |               2009|       35258|     0.2321384|               2009|     27073.26|     43442.74|
| General                    |               2010|       38444|     0.2406681|               2010|     29191.75|     47696.25|
| General                    |               2011|       45154|     0.2499985|               2011|     33865.57|     56442.43|
| General                    |               2012|       60249|     0.2448187|               2012|     45498.92|     74999.08|
| General                    |               2013|       68185|     0.2260609|               2013|     52771.04|     83598.96|
| General                    |               2014|       69535|     0.2233953|               2014|     54001.21|     85068.79|
| General                    |               2015|       56274|     0.2262508|               2015|     43541.96|     69006.04|
| Health Sciences            |               2005|      903415|     0.1384257|               2005|    778359.10|   1028470.90|
| Health Sciences            |               2006|      960524|     0.1424208|               2006|    823725.36|   1097322.64|
| Health Sciences            |               2007|      977974|     0.1504772|               2007|    830811.22|   1125136.78|
| Health Sciences            |               2008|     1152223|     0.1598524|               2008|    968037.39|   1336408.61|
| Health Sciences            |               2009|     1216583|     0.1540699|               2009|   1029144.21|   1404021.79|
| Health Sciences            |               2010|     1165611|     0.1672850|               2010|    970621.75|   1360600.25|
| Health Sciences            |               2011|     1135482|     0.1796957|               2011|    931440.81|   1339523.19|
| Health Sciences            |               2012|     1141833|     0.1735117|               2012|    943711.58|   1339954.42|
| Health Sciences            |               2013|      957476|     0.1869339|               2013|    778491.29|   1136460.71|
| Health Sciences            |               2014|      892001|     0.1830543|               2014|    728716.38|   1055285.62|
| Health Sciences            |               2015|      650880|     0.1493592|               2015|    553665.10|    748094.90|
| Natural Sciences           |               2005|      489332|     0.1308237|               2005|    425315.78|    553348.22|
| Natural Sciences           |               2006|      541105|     0.1474560|               2006|    461315.80|    620894.20|
| Natural Sciences           |               2007|      552588|     0.1500811|               2007|    469654.96|    635521.04|
| Natural Sciences           |               2008|      592863|     0.1698480|               2008|    492166.40|    693559.60|
| Natural Sciences           |               2009|      653696|     0.1679758|               2009|    543890.91|    763501.09|
| Natural Sciences           |               2010|      602563|     0.1805006|               2010|    493800.04|    711325.96|
| Natural Sciences           |               2011|      612053|     0.1804901|               2011|    501583.51|    722522.49|
| Natural Sciences           |               2012|      596770|     0.1684478|               2012|    496245.42|    697294.58|
| Natural Sciences           |               2013|      552213|     0.1775660|               2013|    454158.73|    650267.27|
| Natural Sciences           |               2014|      530149|     0.1689966|               2014|    440555.61|    619742.39|
| Natural Sciences           |               2015|      399736|     0.1359402|               2015|    345395.81|    454076.19|

Data transformation and visualization
-------------------------------------

``` r
dfTable <- as.data.frame(yearRepartition) # we have to fetch the data to get a data frame rather than a tbl_postgrestbl_sqltbl
ggplot(dfTable) + coord_cartesian(ylim = c(0, max(dfTable$items_cnt))) +
    scale_x_continuous(breaks = pretty_breaks()) + scale_y_continuous(labels = comma) +
    theme_bw() +
    geom_line(aes(publication_year, items_cnt))
```

![](dataviz-from-redshift_files/figure-markdown_github/Sample%20GGplot-1.png)<!-- -->

``` r
dsl2Df <- as.data.frame(dslSample2)
ggplot(dsl2Df, aes(publication_year, items_cnt)) +
    geom_smooth(level = 0.95) + 
    geom_line(aes(color = domain)) +
    geom_errorbar(aes(x = publication_year, 
                      ymin = items_vmin, ymax = items_vmax,
                      color = domain, width = 0.2)) +
    scale_fill_viridis() + scale_y_log10(labels = comma) + # + scale_y_continuous(labels = comma)
    scale_x_continuous(breaks = pretty_breaks()) + 
    facet_wrap(~domain , ncol = 2 ) + theme_bw()
```

![](dataviz-from-redshift_files/figure-markdown_github/Multiple%20aesthetics%20and%20facetting-1.png)<!-- -->

TODO: add example with histogram, plots, boxplots, fiting model, ....

Other tools
-----------

base plots

package `manipulate` to bring basic interactivity easily in your plot parameters

Other graph libraries (highCharts, rCharts, ...)

Plotly from R

Reference
---------

### DplyR tutorials and resources

-   [dplyR repo and quick intro](https://github.com/hadley/dplyR)
-   [RStudio: Data wrangling cheat sheet](http://www.rstudio.com/wp-content/uploads/2015/02/data-wrangling-cheatsheet.pdf) (Really great!, covers plyR and tidyr)
-   [Datascience+: data manipulation with dplyR](http://datascienceplus.com/data-manipulation-with-dplyR/)
-   [dplyR tutorial](http://genomicsclass.github.io/book/pages/dplyR_tutorial.html)
-   [Datacamp tutorial: Data Manipulation in R with dplyR](https://www.datacamp.com/courses/dplyR-data-manipulation-r-tutorial)

### ggplot2 resources

-   [RStudio: ggplot2 cheat sheet](http://www.rstudio.com/wp-content/uploads/2015/12/ggplot2-cheatsheet-2.0.pdf) (Really great!)
-   [ggplot2 official documentation](http://docs.ggplot2.org/current/)
-   [How to make any plot in ggplot2?](http://r-statistics.co/ggplot2-Tutorial-With-R.html)
