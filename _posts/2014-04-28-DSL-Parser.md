---
layout: post
title:  "DSL Parser"
date:   2014-04-28
categories: csv, parser 
---

# Why DSL Parser

The [DSL Parser](https://github.com/pauldeschacht/paxparser) is a tool that helps to parse tabular data: it cleans, transforms, skips, merges and splits values from one tabular format into another tabular format. 


If you have to parse a plethora of different formats such csv, Excel, tsv,.. into one common format then one option is to write a parser for every single format. But that becomes painful quite fast. This tool allows you to describe the input format and the output format. The description is done in an (internal/Clojure) DSL. It can easily be extended to process the data according your needs. A number of common build-in functions are already present and are defined using the DSL.

The input format is plain text or Excel. The output format is plain text at the moment.

The parser and DSL are written in Clojure and run on a JVM.

# Using DSL Parser

Troughout the explanation, I will use a small example. 

~~~ javascript
airport;city;pax;origin;year;month;region
BRU;Brussels;1 000.0;BRU-BE-EU;2013;1
CDG;Paris;2 000.0;PAR-FR-EU;2013;12
Total;3000.0
~~~

The output of the DSL parser must look like:

~~~ javascript
airport^BRU^BE^1000^01-Jan-2014
airport^CDG^FR^2000^01-Dec-2013
~~~

A number of transformations took place:

1. A number of lines are skipped (header line and Total line)
2. A new column with a constant value "airport" is added 
3. The column origin is splitted to extract the country code
4. The numbers "1 000.0 and 2 000.0" are converted to integers
5. The column city is completely ignored
6. The year month columns are merged and are transformed into a single date

This transformation represents some of the common use cases, but is possible to extend with far more complex transformations. The DSL `simple-dsl` below is enough to execute the required transformations.

~~~ clojure
(def simple-dsl {
  :global {:token-separator ";"
           :thousand-separator " "
           :decimal-separator "."
           :output-separator "^" }
  :skip [(line-contains? ["airport" "Total"])]
  :stop [(line-contains? ["Thanks" "Please"])]
  :tokens [{:index 0 :name "airport" }
           {:index 2 :name "pax" }
           {:index 3 :name "city-country-code" 
                     :split (split-into-cells ["city" "country" "region"] "-")}
           {:index 4 :name "year"}
           {:index 5 :name "month"} ]
  :columns [{:name "type" :value "airport"} 
            {:name "pax" :transform (convert-to-int)}
            {:name "date" :merge (merge-from ["year" "month"] "-") 
                          :transform (date-yyyy-mm)}]
  :output [{:name "type"}
           {:name "airport"}
           {:name "pax"}
           {:name "country"}
           {:name "date"}] 
})
~~~

## Data Flow 

The data follows a predefined flow. Each step in the flow will process the data: the processing can be a transformation, an enrichment or a removal. The DSL allows you to interact with some parts in that flow.

### Read lines

The processing of the data is line based. For Excel, the cells are merged using a special separator. The seemingly unnecessary merging of Excel helps to line-oriented processing - for example: does the line contains the word "Total". This can be done without specifying the Excel row/column. It avoids a common problem of addressing merged cells in Excel.

### Adding Line Number

Each line is enriched with a line number. The goal is to have a detailed error logging during processing (the extended logging is work in progress)

~~~ javascript
{:line-nb 1 :text "airport;city;passengers;origin"} 
{:line-nb 2 :text "BRU;Brussels;1 000.0;BRU-BE-EU;2014;1"}
{:line-nb 3 :text "CDG;Paris;2 000.0;PAR-FR-EU;2013;12"}
{:line-nb 4 :text "Total;3 000.0"}
~~~

### Skip Lines

The DSL allows to specify which lines to skip. There are a number of predefined functions, such as `line-contains?` and `line-empty?`. If you need to define your own function, have a look at the [next section](#write-your-own-custom-function). 
The function `line-contains?` takes a list of words. If at least one word from that list is found in the line, then the line is skipped.

~~~ clojure
(def simple-dsl
   :skip [(line-contains? ["airport" "Total"])]
)
~~~

~~~ javascript
{:skip true  :line-nb 1 :text "airport;city;pax;origin"} 
{:skip false :line-nb 2 :text "BRU;Brussels;1000.0;BRU-BE-EU;2014;01"}
{:skip false :line-nb 3 :text "CDG;Paris;2000.0;PAR-FR-EU;2013;12"}
{:skip true  :line-nb 4 :text "Total;3000.0"}
~~~

The lines with a `:skip` element set to true are removed from the processing.

~~~ javascript
{:skip false :line-nb 2 :text "BRU;Brussels;1000.0;BRU-BE-EU;2014;01"}
{:skip false :line-nb 3 :text "CDG;Paris;2000.0;PAR-FR-EU;2013;12"}
~~~

### Stop Condition

The DSL allows to specify the last line of the file that needs to be processed. This function is mostly used when working with Excel files, where footnotes are added after the actual data. The stop function has the same useage as skipping the lines.

~~~ clojure
(def simple-dsl
   :stop [(line-contains? ["Thanks" "Please"])]
)
~~~

### Tokenize Lines

Each line is splitting into separate tokens. For plain text files, the separator is specified in the DSL. For Excel files, the tokens correspond to the column.

~~~ clojure
(def simple-dsl {
  :global {:token-separator ";"
  ...
 }})
~~~

Internally, each line is enriched with a field `:split-line` that contains the list of tokens.

~~~ javascript
{:text "BRU;Brussels;1 000.0;BRU-BE-EU;2014;1" 
 :split-line ["BRU" "Brussels" "1 000.0" "BRU-BE-EU" "2014"  "1"] } 
{:text "CDG;Paris;2000.0;PAR-FR-EU;2013;12"    
 :split-line ["CDG" "Paris"    "2 000.0" "PAR-FR_EU" "2013" "12"] }
~~~

### Tokens to Cells

Using the DSL, each individual token in the `:split-line` list is transformed to a __cell__. A cell corresponds to a column of the input format. The column is given by the index. 

Note: The field `:name` is mandatory.

~~~ clojure
(def simple-dsl
  :tokens [{:index 0 :name "airport" }
           {:index 2 :name "pax" } ]
)
~~~

This short DSL only takes the first and third token from the input, the other columns are ignored.

~~~ javascript
{:text ... 
 :split-line ["BRU" "Brussels" "1 000.0" "BRU-BE-EU" "2014" "1"] 
 :cells [{:index 0 :name "airport" :value "BRU"} {:index 2 :name "pax" :value "1 000.0"} ] }
{:text ... 
 :split-line ["CDG" "Paris"    "2 000.0" "PAR-FR-EU" "2013" "12"] 
 :cells [{:index 1 :name "airport" :value "CDG"} {:index 2 :name "pax" :value "2 000.0"} ] }
~~~

##### Splitting a token into multiple cells 

This step also allows to split an input column into different new columns. The pre-defined function `split-into-cells` takes a list of names and a separator. Each name corresponds to the name of a cell.

~~~ clojure
(def simple-dsl 
  :tokens [{:index 0 :name "airport" }
           {:index 2 :name "pax" }
           {:index 3 :name "origin" :split (split-into-cells ["city" "country" "region"] "-")} ]
)
~~~

~~~ javascript
{:text ...
 :split-line ["BRU" "Brussels" "1 000.0" "BRU-BE-EU" "2014" "1"] 
 :cells [{:index 0 :name "airport" :value "BRU"} 
         {:index 2 :name "pax" :value "1 000.0"} 
         {:name "city" :value "BRU"} 
         {:name "country" :value "BE"} 
         {:name "region" :value "EU"} ] }

{:text ...
 :split-line ["CDG" "Paris"    "2 000.0" "PAR-FR-EU" "2013" "12"] 
 :cells [{:index 1 :name "airport" :value "CDG"} 
         {:index 2 :name "pax" :value "2 000.0"} 
         {:name "city" :value "PAR"} 
         {:name "country" :value "FR"} 
         {:name "region" :value "EU"} ] }
~~~

The rest of the data flow will be based on the names of the cells. Due to the splitting of columns, the numerical index `:index` is no longer meaningfull.

### Columns

The next step in the flow allows to prepare the final output columns. Some of the current values still needed to be transformed (such as converting from strings to numbers, or string reformatting). It is also possible to add new columns (with a constant value) or to remove lines based on the value of the column. 
The specifications of the columns is given in the `:columns` field of the specification.

#### Adding a column with a constant value

~~~ clojure
(def simple-dsl
  :columns [{:index 0 :name "airport" }
            {:index 2 :name "pax" }    
            {:index 3 :name "origin" :split (split-into-cells ["city" "country" "region"] "-") } 
            {:index 4 :name "year" }
            {:index 5 :name "month" }
            {:name "type" :value "airport"} ]
)
~~~

This will add a new column with name "type". The value is the string "airport". Since the new column "type" does not originate from any input column, it does not contain a numerical index.

~~~ javascript
{:text ... 
 :split-line ["BRU" "Brussels" "1 000.0" "BRU-BE-EU" "2014" "1"] 
 :cells [{:index 0 :name "airport" :value "BRU"} 
         {:index 2 :name "pax" :value "1 000.0"} 
         {:name "city" :value "BRU"} 
         {:name "country" :value "BE"} 
         {:name "region" :value "EU"} 
         {:name "year" :value "2014"} 
         {:name "month" :value "1" } ] 
 :columns [{:index 0 :name "airport" :value "BRU"} 
           {:index 2 :name "pax" :value "1 000.0"} 
           {:name "city" :value "BRU"} 
           {:name "country" :value "BE"} 
           {:name "region" :value "EU"} 
           {:name "type" :value "airport"}
           {:name "year" :value "2013"}    
           {:name "month" :value "12" } ] }

{:text ... 
 :split-line ["CDG" "Paris"    "2 000.0" "PAR-FR-EU" "2013" "12"] 
 :cells [{:index 1 :name "airport" :value "CDG"} 
         {:index 2 :name "pax" :value "2 000.0"} 
         {:name "city" :value "PAR"} 
         {:name "country" :value "FR"} 
         {:name "region" :value "EU"} ] 
 :columns [{:index 1 :name "airport" :value "CDG"} 
           {:index 2 :name "pax" :value "2 000.0"} 
           {:name "city" :value "PAR"} 
           {:name "country" :value "FR"} 
           {:name "region" :value "EU"} 
           {:name "type" :value "airport"} ] }
~~~

#### Merge columns

This feature allows to merge 2 or more columns into a new column. The merge happens __after__ adding a constant value column, but __before__ the transformation of a value.

~~~ clojure
(def simple-dsl
  :columns [{:name "date" :merge (merge-from ["year" "month"] "-") }]
)
~~~

~~~ javascript
{:text ... 
 :split-line ...
 :cells [{:index 0 :name "airport" :value "BRU"} 
         {:index 2 :name "pax" :value "1 000.0"} 
         {:name "city" :value "BRU"} 
         {:name "country" :value "BE"} 
         {:name "region" :value "EU"} 
         {:name "year" :value "2014"} 
         {:name "month" :value "1" } ] 
 :columns [{:index 0 :name "airport" :value "BRU"} 
           {:index 2 :name "pax" :value "1 000.0"} 
           {:name "city" :value "BRU"} 
           {:name "country" :value "BE"} 
           {:name "region" :value "EU"} 
           {:name "type" :value "airport"}
           {:name "year" :value "2014"}    
           {:name "month" :value "1" } 
           {:name "date" :value "2014-1"] }

{:text ... 
 :split-line ... 
 ...
 :columns [{:index 1 :name "airport" :value "CDG"} 
           {:index 2 :name "pax" :value "2 000.0"} 
           {:name "city" :value "PAR"} 
           {:name "country" :value "FR"} 
           {:name "region" :value "EU"} 
           {:name "type" :value "airport"} 
           {:name "year" :value "2013"}
           {:name "month" :value "12"}
           {:name "date" :value "2013-12"] }
~~~

#### Transforming a value

~~~ clojure
(def simple-dsl
  :columns [{:name "type" :value "airport"} 
            {:name "pax" :transform (convert-to-int)
            {:name "date" :merge (merge-from ["year" "month"] "-") :transform (date-yyyy-month) } ]
)
~~~

The transformation function `convert-to-int` receives the value of the current cell ("1 000.0" and "2 000.0") and the full DSL specification (global parameters such as thousand-separator can be accessed). The result must be the new value.
Likewise, the function date-yyyy-month receives the value of the current cell ("2014-1" and "2013-12") and calls ordinary Clojure functions to transform these values into "01-Jan-2014" and "01-Dec-2013".

~~~ javascript
{:text ...
 :split-line ["BRU" "Brussels" "1 000.0" "BRU-BE-EU" "2014" "1"] 
 :cells [{:index 0 :name "airport" :value "BRU"} 
         {:index 2 :name "pax" :value "1 000.0"} 
         {:name "city" :value "BRU"} 
         {:name "country" :value "BE"} 
         {:name "region" :value "EU"} 
         {:name "year" :value "2014"} 
         {:name "month" :value "1"} ] 
 :columns [{:name "type" :value "airport}
           {:name "airport"} 
           {:name "pax" :value 1000} 
           {:name "city :value "BRU"} 
           {:name "country" :value "BE"} 
           {:name "region" :value "EU"}
           {:name "year" :value "2014"
           {:name "month" :value "1"} 
           {:name "date" :value "1-Jan-2014" } ] }

{:text ...
 :split-line ["CDG" "Paris"    "2 000.0" "PAR-FR-EU" "2013" "12"] 
 :cells [{:index 1 :name "airport" :value "CDG"} 
         {:index 2 :name "pax" :value "2 000.0"} 
         {:name "city" :value "PAR"} 
         {:name "country" :value "FR"} 
         {:name "region" :value "EU"} 
         {:name "year" :value "2013"} 
         {:name "month" :value "12"} ] 

 :columns [{:name "type" :value "airport"}
           {:name "airport" :value "CDG"} 
           {:name "pax" :value 2000} 
           {:name "city" :value "PAR"} 
           {:name "country" :value "FR"} 
           {:name "region" :value "EU"}
           {:name "year" :value "2013"
           {:name "month" :value "12"} 
           {:name "date" :value "1-Dec-2014" } ] }
~~~

#### Repeating values

Sometimes it is needed to __repeat__ the values in a column. This is mostly the case with Excel files where a value is only set once, and the cells below are supposed to have the same value. The option `:repeat-down true` in the DSL will explicitely repeat that value. The repetition is done as long as the cell is empty.

~~~ javascript
airport;country;pax;
CDH;France;2000
ORY;      ;500
NCE;      ;250
LYS;      ;100
BRU;Belgium;1000
CRL;      ;200
~~~

~~~ clojure
(def simple-dsl {
  :global {:token-separator ";" ... }
  :tokens [{:index 0 :name "airport" }
           {:index 1 :name "country" }
           {:index 2 :name "pax" }]
  :columns [{:name "country" :repeat-down true}
            {:name "pax" :transform (convert-to-int)} ]
  :output [{:name "airport"}
           {:name "country"}
           {:name "pax"}]  })
~~~

Using the option `:repeat-down true` in the column will fill in the empty cells.

### Output 

The DSL allows you to define which fields are written to the output. 
The options for the output are 

* `:name` (mandatory) gives the name of the output column
* `:source` (optional) indicates the name of the source column. By default, the source is the same as `:name`
* `:value` (optional) allows to define a constant value
* `:skip-line` (optional) takes a function that returns a boolean. It is possible to skip an entire line based on the value of a cell. 

#### Single output 

~~~ clojure
(def simple-dsl {
  :output [{:name "type"}
           {:name "airport"}
           {:name "pax" :skip-line (negative?}
           {:name "country"}
           {:name "date"}] 

 })

~~~

#### Multiple output 

By defining several outputs, the input data will result in multiple output files. This avoid to reparse the same input with different DSL's. 

~~~ javascript
airport;pax 2013;pax 2012;pax 2011;pax 2010
CDH;    2000;    1700;    1500;    1000
ORY;    1000;    850;     700;     650
~~~

~~~ clojure
(def simple-dsl {
  :outputs [ ;;first output contains 2013 numbers
            [{:name "year" :value "2013"}
             {:name "airport"}
             {:name "pax 2013"}]
             ;;second output contains 2012 numbers
            [{:name "year" :value "2012"}
             {:name "airport"}
             {:name "pax 2012"}]
             ;;third output contains 2011 numbers
            [{:name "year" :value "2011"}
             {:name "airport"}
             {:name "pax 2011"}]
           ]
 })
~~~

The first output 

~~~
2013;CDH;2000
2013;ORY;1000
~~~

The second output

~~~
2012;CDH;1700
2012;ORY;850
~~~

etc..

## Complete example

For a more complete and complex example, you can have a look at the [DSL](https://github.com/pauldeschacht/paxparser/blob/master/src/parser/pax/dsl/destatis.clj) that parses the [public German airport data](https://www.destatis.de/DE/Publikationen/Thematisch/TransportVerkehr/Luftverkehr/Luftverkehr2080600141025.xlsx?__blob=publicationFile). 

# Specification

## `:global`

* `:token-separator` (optional,default ascii char 31)
* `:thousand-separator` (optional, no default value)
* `:decimal-separator` (optional, default ".")
* `:output-separator` (optional, default "\t")

## `:skip` & `:stop`

* `:skip` takes a list of functions. Each function must return a boolean value. If a least one function returns true, the line will be skipped
* `:stop` takes a list of functions. Each function must return a boolean value. If a least one function returns true, this line will be considered the end of the input

## `:tokens`

* `:index` (optional) is a numerical value which defines the index of the token.
* `:name` (mandatory) is the name of the cell. This name will be used as reference to the token
* `:split` (optional) takes a functions that returns a list of cells. A predefined function `split-into-cells` takes a list of names and a separator.

## `:columns`

* `:name` (mandatory)
* `:value`
* `:merge`
* `:transform`

## `:output`

* `:name` (mandatory)
* `:source` (optional, default value is `:name`) references to the name of the column
* `:value` (optional) contains a constant value
* `:skip-line` (optional) contains a function that should return if the line should not part of the output


# Extending the DSL

## Write your own custom function

A bit of Clojure knowledge will definitely help extending the DSL. 

A custom DSL function is a higher order function (ie a function that returns another function). The higher order function takes the parameters from the DSL.

~~~ clojure
(defn add-prefix [prefix]
  ...
)

(def simple-dsl {
 :columns [ {:name "airport" :transform (add-prefix "**") } ]
})

~~~

The higher order function returns a function. That (anonymous) function will be called for every single line. The arguments to the anonymous function are the DSL and the value of the cell.

~~~ clojure
(defn add-prefix [prefix]
 (fn [spec value]
   (str prefix value)))
~~~

Note: this is work in progress, the parameters of the anynomous functions are not consistent.