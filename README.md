# csv-groupby
## `gb`   A Command that does a SQL like "group by" on delimited files OR arbitrary lines of text

gb is a command that takes delimited data (like csv files) or lines of text (like a log file) and emulates a SQL select-group-by on that data.  A utility partially inspired by [xsv](https://github.com/BurntSushi/xsv) and the desire to stop having to write the same perl one-liners to analyze massive log files.

- Do group-bys with sums, counts, and count distincts
- Can process [CSV](https://crates.io/crates/csv) files OR text using [regular](https://www.pcre.org/current/doc/html/pcre2syntax.html) [expressions](https://crates.io/crates/pcre2)
- Process files or stdin as a data source:
  - csv files
  - text/log where parsed fields come from regular expression sub groups
  - files can be decompressed (like .zst, .gz, .xz, etc) data files on the fly
  - recursive [--walk](https://github.com/BurntSushi/ripgrep/tree/master/ignore) directory trees and filter for only the files you want
- Filenames (-p) parsed with regular expressions used in filtering can also have these sub groups used as fields.
- Delimited output or prettytable format output
- Fast - processing at 500MB/s is not uncommon on multicore machines

It does this very fast by "slicing" blocks of data on line boundary points and forwarding those line-even blocks to multiple parser threads.


## HOW-TO:

You identify fields as column numbers.  These will either be part of the "key" or group-by, an aggregate (avg or sum), or count distinct.  You may use none to many fields for each kind of field.


### Summaries on a csv file:

Using some airline flight data taken from:
http://stat-computing.org/dataexpo/2009/2008.csv.bz2

>*Note that this data is truncated a bit here and reformated from this csv to make it readable.*
```
  1     2        3           4         5         6          7          8           9

Year  Month  DayofMonth  DayOfWeek  DepTime  CRSDepTime  ArrTime  CRSArrTime  UniqueCarrier  ...
2008  1      3           4          2003     1955        2211     2225        WN             ...
2008  1      3           4          754      735         1002     1000        WN             ...
2008  1      3           4          628      620         804      750         WN             ...
2008  1      3           4          926      930         1054     1100        WN             ...
2008  1      3           4          1829     1755        1959     1925        WN             ...
2008  1      3           4          1940     1915        2121     2110        WN             ...
2008  1      3           4          1937     1830        2037     1940        WN             ...
2008  1      3           4          1039     1040        1132     1150        WN             ...
2008  1      3           4          617      615         652      650         WN             ...
....
```
Running the command:
```
gb -f 2008.csv -k 2,9 -s 14 --skip_header -c | head -10
```

Here this command corresponds to the SQL:
```
select Month, UniqueCarrier, count(*), sum(AirTime) from csv group by Month, UniqueCarrier
       ^              ^                     ^
       |              |                     |
   -k  2,             9                -s   14
```

Here's a partial output:
``` 
 k:2 | k:9 | count  | s:14    | a:14
-----+-----+--------+---------+--------------------
 1   | 9E  | 22840  | 1539652 | 71.22412915760744
 1   | AA  | 52390  | 7317245 | 144.5524496246543
 1   | AQ  | 4026   | 247200  | 61.830915457728864
 1   | AS  | 12724  | 1587637 | 129.23378103378104
 1   | B6  | 16441  | 2477670 | 152.93315227455096
 1   | CO  | 25168  | 3878167 | 155.51858683883387
 1   | DL  | 38241  | 4812768 | 130.22967853663818
 1   | EV  | 23106  | 1584233 | 72.21739526826822
....
 ```

Another example that determines number of airplanes used and time spent in the air by that carrier.

```
select Carrier, count(*), sum(AirTime), count(distinct TailNum), sum(AirTime) average(AirTime) from csv 
group by Carrier
```
The following command emulates this:
```
gb -f ~/dl/2008.csv -k 9 -s 14 -u 11 -s 14 -a 14 --skip_header
```

Output:
> *Note that the output order of the columns does not correspond to the order of the field options.  It is fixed to keys, count, sums, avgs, and then uniques.*
```
 k:9 | count   | s:14      | s:14      | a:14               | u:11
-----|---------|-----------|-----------|--------------------|------
 9E  | 262109  | 18080077  | 18080077  | 71.11840692300127  | 162
 AA  | 604655  | 82989937  | 82989937  | 141.8001178963196  | 656
 AQ  | 7797    | 479581    | 479581    | 61.889405084527034 | 21
 AS  | 151045  | 19569900  | 19569900  | 131.83977040764768 | 126
 B6  | 196018  | 28849406  | 28849406  | 150.22524356777978 | 154
 CO  | 298342  | 45784515  | 45784515  | 155.86589297447088 | 378
 ...
 ```

## Summaries on arbitrary text
#### Using regular expression sub groups as data fields.
Alternatively, you can use a regular expression (see -r option) against the lines where the sub groups captured become fields and these field indices correspond with the subgroup.  This is useful for data that is not as organized like a csv such as logs files etc.



TODO/ideas:  

- Better readme
- csv comment char
- Use stdin as a filelist source in addition to stdin as a data source
- more aggregates: min, max, empty_count, number_count, zero_count
  - avg and sum done
- faster sort on mixed key fields before output
- do more work to multi-line re mode - not sure how it should really work yet
> This does function and I have used it on well formed xml, but not sure it will be a thing or not.
- flat mode - no summary but just write X fields to the output as found - a kind s/()..()..()/$1,$2,$3,.../;
- ticker interval
- ticker off
- use RE find/search for matches instead of line bound interface?

