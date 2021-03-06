
**If you are viewing this file on CRAN, please check latest news on GitHub [here](https://github.com/Rdatatable/data.table/blob/master/NEWS.md).**

### Changes in v1.10.5  ( in development )

#### NOTICE OF INTENDED FUTURE POTENTIAL BREAKING CHANGES

1. `fread()`'s `na.strings=` argument :
    ```
    "NA"                                      # old default
    getOption("datatable.na.strings", "NA")   # this release; i.e. the same; no change yet
    getOption("datatable.na.strings", "")     # future release
    ```
This option controls how `,,` is read in character columns. It does not affect numeric columns which read `,,` as `NA` regardless. We would like `,,`=>`NA` for consistency with numeric types, and `,"",`=>empty string to be the standard default for `fwrite/fread` character columns so that `fread(fwrite(DT))==DT` without needing any change to any parameters. `fwrite` has never written `NA` as `"NA"` in case `"NA"` is a valid string in the data; e.g., 2 character id columns sometimes do. Instead, `fwrite` has always written `,,` by default for an `<NA>` in a character columns. The use of R's `getOption()` allows users to move forward now, using `options(datatable.fread.na.strings="")`, or restore old behaviour when the default's default is changed in future, using `options(datatable.fread.na.strings="NA")`.

2. `fread()` and `fwrite()`'s `logical01=` argument :
    ```
    logical01 = FALSE                         # old default
    getOption("datatable.logical01", FALSE)   # this release; i.e. the same; no change yet
    getOption("datatable.logical01", TRUE)    # future release
    ```
This option controls whether a column of all 0's and 1's is read as `integer`, or `logical` directly to avoid needing to change the type afterwards to `logical` or use `colClasses`. `0/1` is smaller and faster than `"TRUE"/"FALSE"`, which can make a significant difference to space and time the more `logical` columns there are. When the default's default changes to `TRUE` for `fread` we do not expect much impact since all arithmetic operators that are currently receiving 0's and 1's as type `integer` (think `sum()`) but instead could receive `logical`, would return exactly the same result on the 0's and 1's as `logical` type. However, code that is manipulating column types using `is.integer` or `is.logical` on `fread`'s result, could require change. It could be painful if `DT[(logical_column)]` (i.e. `DT[logical_column==TRUE]`) changed behaviour due to `logical_column` no longer being type `logical` but `integer`. But that is not the change proposed. The change is the other way around; i.e., a previously `integer` column holding only 0's and 1's would now be type `logical`. Since it's that way around, we believe the scope for breakage is limited. We think a lot of code is converting 0/1 integer columns to logical anyway, either using `colClasses=` or afterwards with an assign. For `fwrite`, the level of breakage depends on the consumer of the output file. We believe `0/1` is a better more standard default choice to move to. See notes below about improvements to `fread`'s sampling for type guessing, and automatic rereading in the rare cases of out-of-sample type surprises.

These options are meant for temporary use to aid your migration, [#2652](https://github.com/Rdatatable/data.table/pull/2652). You are not meant to set them to the old default and then not migrate your code that is dependent on the default. Either set the argument explicitly so your code is not dependent on the default, or change the code to cope with the new default. Over the next few years we will slowly start to remove these options, warning you if you are using them, and return to a simple default. See the history of NEWS and NEWS.0 for past migrations that have, generally speaking, been successfully managed in this way. For example, at the end of NOTES for this version (below in this file) is a note about the usage of `datatable.old.unique.by.key` now warning, as you were warned it would do over a year ago. When that change was introduced, the default was changed and that option provided an option to restore the old behaviour. These `fread`/`fwrite` changes are even more cautious and not even changing the default's default yet. Giving you extra warning by way of this notice to move forward. And giving you a chance to object.

#### NEW FEATURES

1. `fread()`:
    * Efficiency savings at C level including **parallelization** announced [here](https://github.com/Rdatatable/data.table/wiki/talks/BARUG_201704_ParallelFread.pdf); e.g. a 9GB 2 column integer csv input is **50s down to 12s** to cold load on a 4 core laptop with 16GB RAM and SSD. Run `echo 3 >/proc/sys/vm/drop_caches` first to measure cold load time. Subsequent load time (after file has been cached by OS on the first run) **40s down to 6s**.
    * The [fread for small data](https://github.com/Rdatatable/data.table/wiki/Convenience-features-of-fread) page has been revised.
    * Memory maps lazily; e.g. reading just the first 10 rows with `nrow=10` is **12s down to 0.01s** from cold for the 9GB file. Large files close to your RAM limit may work more reliably too. The progress meter will commence sooner and more consistently.
    * `fread` has always jumped to the middle and to the end of the file for a much improved column type guess. The sample size is increased from 100 rows at 10 jump jump points (1,000 rows) to 100 rows at 100 jumps points (10,000 row sample). In the rare case of there still being out-of-sample type exceptions, those columns are now *automatically reread* so you don't have to use `colClasses` yourself.
    * Large number of columns support; e.g. **12,000 columns** tested.
    * **Quoting rules** are more robust and flexible. See point 10 on the wiki page [here](https://github.com/Rdatatable/data.table/wiki/Convenience-features-of-fread#10-automatic-quote-escape-method-detection-including-no-escape).
    * Numeric data that has been quoted is now detected and read as numeric.
    * The ability to position `autostart` anywhere inside one of multiple tables in a single file is removed with warning. It used to search upwards from that line to find the start of the table based on a consistent number of columns. People appear to be using `skip="string"` or `skip=nrow` to find the header row exactly, which is retained and simpler. It was too difficult to retain search-upwards-autostart together with skipping/filling blank lines, filling incomplete rows and parallelization too. If there is any header info above the column names, it is still auto detected and auto skipped (particularly useful when loading a set of files where the column names start on different lines due to a varying height messy header).
    * `dec=','` is now implemented directly so there is no dependency on locale. The options `datatable.fread.dec.experiment` and `datatable.fread.dec.locale` have been removed.
    * `\\r\\r\\n` line endings are now handled such as produced by `base::download.file()` when it doubles up `\\r`. Other rare line endings (`\\r` and `\\n\\r`) are now more robust.
    * Mixed line endings are now handled; e.g. a file formed by concatenating a Unix file and a Windows file so that some lines end with `\\n` while others end with `\\r\\n`.
    * Improved automatic detection of whether the first row is column names by comparing the types of the fields on the first row against the column types ascertained by the 10,000 rows sample (or `colClasses` if provided). If a numeric column has a string value at the top, then column names are deemed present.
    * Detects GB-18030 and UTF-16 encodings and in verbose mode prints a message about BOM detection.
    * Detects and ignores trailing ^Z end-of-file control character sometimes created on MS DOS/Windows, [#1612](https://github.com/Rdatatable/data.table/issues/1612). Thanks to Gergely Daróczi for reporting and providing a file.
    * Added ability to recognize and parse hexadecimal floating point numbers, as used for example in Java. Thanks for @scottstanfield [#2316](https://github.com/Rdatatable/data.table/issues/2316) for the report.
    * Now handles floating-point NaN values in a wide variety of formats, including `NaN`, `sNaN`, `1.#QNAN`, `NaN1234`, `#NUM!` and others, [#1800](https://github.com/Rdatatable/data.table/issues/1800). Thanks to Jori Liesenborgs for highlighting and the PR.
    * If negative numbers are passed to `select=` the out-of-range error now suggests `drop=` instead, [#2423](https://github.com/Rdatatable/data.table/issues/2423). Thanks to Michael Chirico for the suggestion.
    * `sep=NULL` or `sep=""` (i.e., no column separator) can now be used to specify single column input reliably like `base::readLines`, [#1616](https://github.com/Rdatatable/data.table/issues/1616). `sep='\\n'` still works (even on Windows where line ending is actually `\\r\\n`) but `NULL` or `""` are now documented and recommended. Thanks to Dmitriy Selivanov for the pull request and many others for comments. As before, `sep=NA` is not valid; use the default `"auto"` for automatic separator detection. `sep='\\n'` is now deprecated and in future will start to warn when used.
    * Single-column input with blank lines is now valid and the blank lines are significant (representing `NA`). The blank lines are significant even at the very end, which may be surprising on first glance. The change is so that `fread(fwrite(DT))==DT` for single-column inputs containing `NA` which are written as blank. There is no change when `ncol>1`; i.e., input stops with detailed warning at the first blank line, because a blank line when `ncol>1` is invalid input due to no separators being present. Thanks to @skanskan, Michael Chirico, @franknarf1 and Pasha for the testing and discussions, [#2106](https://github.com/Rdatatable/data.table/issues/2106).
    * Too few column names are now auto filled with default column names, with warning, [#1625](https://github.com/Rdatatable/data.table/issues/1625). If there is just one missing column name it is guessed to be for the first column (row names or an index), otherwise the column names are filled at the end. Similarly, too many column names now automatically sets `fill=TRUE`, with warning.
    * `skip=` and `nrow=` are more reliable and are no longer affected by invalid lines outside the range specified. Thanks to Ziyad Saeed and Kyle Chung for reporting, [#1267](https://github.com/Rdatatable/data.table/issues/1267).
    * Ram disk (`/dev/shm`) is no longer used for the output of system command input. Although faster when it worked, it was causing too many device full errors; e.g., [#1139](https://github.com/Rdatatable/data.table/issues/1139) and [zUMIs/19](https://github.com/sdparekh/zUMIs/issues/19). Thanks to Kyle Chung for reporting. Standard `tempdir()` is now used. If you wish to use ram disk, set TEMPDIR to `/dev/shm`; see `?tempdir`.
    * Detecting whether a very long input string is a file name or data is now much faster, [#2531](https://github.com/Rdatatable/data.table/issues/2531). Many thanks to @javrucebo for the detailed report, benchmarks and suggestions.
    * A column of `TRUE/FALSE`s is ok, as well as `True/False`s and `true/false`s, but mixing styles (e.g. `TRUE/false`) is not and will be read as type `character`.
    * Many thanks to @yaakovfeldman, Guillermo Ponce, Arun Srinivasan, Hugh Parsonage, Mark Klik, Pasha Stetsenko, Mahyar K, Tom Crockett, @cnoelke, @qinjs, @etienne-s, Mark Danese, Avraham Adler, @franknarf1, @MichaelChirico, @tdhock, Luke Tierney for testing dev and reporting these regressions before release to CRAN: #2070, #2073, #2087, #2091, #2107, #2118, #2092, #1888, #2123, #2167, #2194, #2238, #2228, #1464, #2201, #2287, #2299, #2285, #2251, #2347, #2222, #2352, #2246, #2370, #2371, #2404, #2196, #2322, #2453, #2446, #2464, #2457, #1895, #2481, #2499, #2516, #2520, #2512, #2523, #2542, #2526, #2518, #2515, #1671, #2267, #2561, #2625, #2265, #2548, #2535

2. `fwrite()`:
    * empty strings are now always quoted (`,"",`) to distinguish them from `NA` which by default is still empty (`,,`) but can be changed using `na=` as before. If `na=` is provided and `quote=` is the default `'auto'` then `quote=` is set to `TRUE` so that if the `na=` value occurs in the data, it can be distinguished from `NA`. Thanks to Ethan Welty for the request [#2214](https://github.com/Rdatatable/data.table/issues/2214) and Pasha for the code change and tests, [#2215](https://github.com/Rdatatable/data.table/issues/2215).
    * `logical01` has been added and the old name `logicalAsInt` retained. Pease move to the new name when convenient for you. The old argument name (`logicalAsInt`) will slowly be deprecated over the next few years. The default is unchanged: `FALSE`, so `logical` is still written as `"TRUE"`/`"FALSE"` in full by default. We intend to change the default's default in future to `TRUE`; see the notice at the top of these release notes.

3. Added helpful message when subsetting by a logical column without wrapping it in parentheses, [#1844](https://github.com/Rdatatable/data.table/issues/1844). Thanks @dracodoc for the suggestion and @MichaelChirico for the PR.

4. `tables` gains `index` argument for supplementary metadata about `data.table`s in memory (or any optionally specified environment), part of [#1648](https://github.com/Rdatatable/data.table/issues/1648). Thanks due variously to @jangorecki, @rsaporta, @MichaelChirico for ideas and work towards PR.

5. Improved auto-detection of `character` inputs' formats to `as.ITime` to mirror the logic in `as.POSIXlt.character`, [#1383](https://github.com/Rdatatable/data.table/issues/1383) Thanks @franknarf1 for identifying a discrepancy and @MichaelChirico for investigating.

6. `setcolorder()` now accepts less than `ncol(DT)` columns to be moved to the front, [#592](https://github.com/Rdatatable/data.table/issues/592). Thanks @MichaelChirico for the PR. This also incidentally fixed [#2007](https://github.com/Rdatatable/data.table/issues/2007) whereby explicitly setting `select = NULL` in `fread` errored; thanks to @rcapell for reporting that and @dselivanov and @MichaelChirico for investigating and providing a new test.

7. Three new *Grouping Sets* functions: `rollup`, `cube` and `groupingsets`, [#1377](https://github.com/Rdatatable/data.table/issues/1377). Allows to aggregation on various grouping levels at once producing sub-totals and grand total.

8. `as.data.table()` gains new method for `array`s to return a useful data.table, [#1418](https://github.com/Rdatatable/data.table/issues/1418).

9. `print.data.table()` (all via master issue  [#1523](https://github.com/Rdatatable/data.table/issues/1523)):

    * gains `print.keys` argument, `FALSE` by default, which displays the keys and/or indices (secondary keys) of a `data.table`. Thanks @MichaelChirico for the PR, Yike Lu for the suggestion and Arun for honing that idea to its present form.

    * gains `col.names` argument, `"auto"` by default, which toggles which registers of column names to include in printed output. `"top"` forces `data.frame`-like behavior where column names are only ever included at the top of the output, as opposed to the default behavior which appends the column names below the output as well for longer (>20 rows) tables. `"none"` shuts down column name printing altogether. Thanks @MichaelChirico for the PR, Oleg Bondar for the suggestion, and Arun for guiding commentary.

    * list columns would print the first 6 items in each cell followed by a comma if there are more than 6 in that cell. Now it ends ",..." to make it clearer, part of [#1523](https://github.com/Rdatatable/data.table/issues/1523). Thanks to @franknarf1 for drawing attention to an issue raised on Stack Overflow by @TMOTTM [here](https://stackoverflow.com/q/47679701).

10. `setkeyv` accelerated if key already exists [#2331](https://github.com/Rdatatable/data.table/issues/2331). Thanks to @MarkusBonsch for the PR.

11. Keys and indexes are now partially retained up to the key column assigned to with ':=' [#2372](https://github.com/Rdatatable/data.table/issues/2372). They used to be dropped completely if any one of the columns was affected by `:=`. Tanks to @MarkusBonsch for the PR.

12. Faster `as.IDate` and `as.ITime` methods for `POSIXct` and `numeric`, [#1392](https://github.com/Rdatatable/data.table/issues/1392). Thanks to Jan Gorecki for the PR.

13. `unique(DT)` now returns `DT` early when there are no duplicates to save RAM, [#2013](https://github.com/Rdatatable/data.table/issues/2013). Thanks to Michael Chirico for the PR, and thanks to @mgahan for pointing out a reversion in `na.omit.data.table` before release, [#2660](https://github.com/Rdatatable/data.table/issues/2660#issuecomment-371027948).

14. `uniqueN()` is now faster on logical vectors. Thanks to Hugh Parsonage for [PR#2648](https://github.com/Rdatatable/data.table/pull/2648).
    ```
    N = 1e9
                                     was      now
    x = c(TRUE,FALSE,NA,rep(TRUE,N))
    uniqueN(x) == 3                 5.4s    0.00s
    x = c(TRUE,rep(FALSE,N), NA)
    uniqueN(x,na.rm=TRUE) == 2      5.4s    0.00s
    x = c(rep(TRUE,N),FALSE,NA)
    uniqueN(x) == 3                 6.7s    0.38s
    ```

15. Subsetting optimization with keys and indices is now possible for compound queries like `DT[a==1 & b==2]`, [#2472](https://github.com/Rdatatable/data.table/issues/2472).
Thanks to @MichaelChirico for reporting and to @MarkusBonsch for the implementation.

16. `melt.data.table` now offers friendlier functionality for providing `value.name` for `list` input to `measure.vars`, [#1547](https://github.com/Rdatatable/data.table/issues/1547). Thanks @MichaelChirico and @franknarf1 for the suggestion and use cases, @jangorecki and @mrdwab for implementation feedback, and @MichaelChirico for ultimate implementation.

17. `update.dev.pkg` is new function to update package from development repository, it will download package sources only when newer commit is available in repository. `data.table::update.dev.pkg()` defaults updates `data.table`, but any package can be used.

18. Item 1 in NEWS for [v1.10.2](https://github.com/Rdatatable/data.table/blob/master/NEWS.md#changes-in-v1102--on-cran-31-jan-2017) on CRAN in Jan 2017 included :
> When j is a symbol prefixed with `..` it will be looked up in calling scope and its value taken to be column names or numbers.
> When you see the `..` prefix think one-level-up, like the directory `..` in all operating systems means the parent directory.
> In future the `..` prefix could be made to work on all symbols apearing anywhere inside `DT[...]`.
The response has been positive ([this tweet](https://twitter.com/MattDowle/status/967290562725359617) and [FR#2655](https://github.com/Rdatatable/data.table/issues/2655)) and so this prefix is now expanded to all symbols appearing in `j=` as a first step; e.g. :
    ```R
    cols = "colB"
    DT[, c(..cols, "colC")]   # same as DT[, .(colB,colC)]
    DT[, -..cols]             # all columns other than colB
    ```
Thus, `with=` should no longer be needed in any cases. Please change to using the `..` prefix and in a few years we will start to formally deprecate and remove the `with=` parameter.  If this is well received, the `..` prefix could be expanded to symbols appearing in `i=` and `by=`, too.

#### BUG FIXES

1. The new quote rules handles this single field `"Our Stock Screen Delivers an Israeli Software Company (MNDO, CTCH)<\/a> SmallCapInvestor.com - Thu, May 19, 2011 10:02 AM EDT<\/cite><\/div>Yesterday in \""Google, But for Finding
 Great Stocks\"", I discussed the value of stock screeners as a powerful tool"`, [#2051](https://github.com/Rdatatable/data.table/issues/2051). Thanks to @scarrascoso for reporting. Example file added to test suite.

2. `fwrite()` creates a file with permissions that now play correctly with `Sys.umask()`, [#2049](https://github.com/Rdatatable/data.table/issues/2049). Thanks to @gnguy for reporting.

3. `fread()` no longer holds an open lock on the file when a line outside the large sample has too many fields and generates an error, [#2044](https://github.com/Rdatatable/data.table/issues/2044). Thanks to Hugh Parsonage for reporting.

4. Setting `j = {}` no longer results in an error, [#2142](https://github.com/Rdatatable/data.table/issues/2142). Thanks Michael Chirico for the pull request.

5. Segfault in `rbindlist()` when one or more items are empty, [#2019](https://github.com/Rdatatable/data.table/issues/2019). Thanks Michael Lang for the pull request. Another segfault if the result would be more than 2bn rows, thanks to @jsams's comment in [#2340](https://github.com/Rdatatable/data.table/issues/2340#issuecomment-331505494).

6. Error printing 0-length `ITime` and `NA` objects, [#2032](https://github.com/Rdatatable/data.table/issues/2032) and [#2171](https://github.com/Rdatatable/data.table/issues/2171). Thanks Michael Chirico for the pull requests and @franknarf1 for pointing out a shortcoming of the initial fix.

7. `as.IDate.POSIXct` error with `NULL` timezone, [#1973](https://github.com/Rdatatable/data.table/issues/1973). Thanks @lbilli for reporting and Michael Chirico for the pull request.

8. Printing a null `data.table` with `print` no longer visibly outputs `NULL`, [#1852](https://github.com/Rdatatable/data.table/issues/1852). Thanks @aaronmcdaid for spotting and @MichaelChirico for the PR.

9. `data.table` now works with Shiny Reactivity / Flexdashboard. The error was typically something like `col not found` in `DT[col==val]`. Thanks to Dirk Eddelbuettel leading Matt through reproducible steps and @sergeganakou and Richard White for reporting. Closes [#2001](https://github.com/Rdatatable/data.table/issues/2001) and [shiny/#1696](https://github.com/rstudio/shiny/issues/1696).

10. The `as.IDate.POSIXct` method passed `tzone` along but was not exported. So `tzone` is now taken into account by `as.IDate` too as well as `IDateTime`, [#977](https://github.com/Rdatatable/data.table/issues/977) and [#1498](https://github.com/Rdatatable/data.table/issues/1498). Tests added.

11. Named logical vector now select rows as expected from single row data.table. Thanks to @skranz for reporting. Closes [#2152](https://github.com/Rdatatable/data.table/issues/2152).

12. `fread()`'s rare `Internal error: Sampling jump point 10 is before the last jump ended` has been fixed, [#2157](https://github.com/Rdatatable/data.table/issues/2157). Thanks to Frank Erickson and Artem Klevtsov for reporting with example files which are now added to the test suite.

13. `CJ()` no longer loses attribute information, [#2029](https://github.com/Rdatatable/data.table/issues/2029). Thanks to @MarkusBonsch and @royalts for the pull request.

14. `split.data.table` respects `factor` ordering in `by` argument, [#2082](https://github.com/Rdatatable/data.table/issues/2082). Thanks to @MichaelChirico for identifying and fixing the issue.

15. `.SD` would incorrectly include symbol on lhs of `:=` when `.SDcols` is specified and `get()` appears in `j`. Thanks @renkun-ken for reporting and the PR, and @ProfFancyPants for reporing a regression introduced in the PR. Closes [#2326](https://github.com/Rdatatable/data.table/issues/2326) and [#2338](https://github.com/Rdatatable/data.table/issues/2338).

16. Integer values that are too large to fit in `int64` will now be read as strings [#2250](https://github.com/Rdatatable/data.table/issues/2250).

17. Internal-only `.shallow` now retains keys correctly, [#2336](https://github.com/Rdatatable/data.table/issues/2336). Thanks to @MarkusBonsch for reporting, fixing ([PR #2337](https://github.com/Rdatatable/data.table/pull/2337)) and adding 37 tests. This much advances the journey towards exporting `shallow()`, [#2323](https://github.com/Rdatatable/data.table/issues/2323).

18. `isoweek` calculation is correct regardless of local timezone setting (`Sys.timezone()`), [#2407](https://github.com/Rdatatable/data.table/issues/2407). Thanks to @MoebiusAV and @SimonCoulombe for reporting and @MichaelChirico for fixing.

19. Fixed `as.xts.data.table` to support all xts supported time based index clasess [#2408](https://github.com/Rdatatable/data.table/issues/2408). Thanks to @ebs238 for reporting and for the PR.

20. A memory leak when a very small number such as `0.58E-2141` is bumped to type `character` is resolved, [#918](https://github.com/Rdatatable/data.table/issues/918).

21. The edge case `setnames(data.table(), character(0))` now works rather than error, [#2452](https://github.com/Rdatatable/data.table/issues/2452).

22. Order of rows returned in non-equi joins were incorrect in certain scenarios as reported under [#1991](https://github.com/Rdatatable/data.table/issues/1991). This is now fixed. Thanks to @Henrik-P for reporting.

23. Non-equi joins work as expected when `x` in `x[i, on=...]` is a 0-row data.table. Closes [#1986](https://github.com/Rdatatable/data.table/issues/1986).

24. Non-equi joins along with `by=.EACHI` returned incorrect result in some rare cases as reported under [#2360](https://github.com/Rdatatable/data.table/issues/2360). This is fixed now. This fix also takes care of [#2275](https://github.com/Rdatatable/data.table/issues/2275). Thanks to @ebs238 for the nice minimal reproducible report, @Mihael for asking on SO and to @Frank for following up on SO and filing an issue.

25. `by=.EACHI` works now when `list` columns are being returned and some join values are missing, [#2300](https://github.com/Rdatatable/data.table/issues/2300). Thanks to @jangorecki and @franknarf1 for the reproducible examples which have been added to the test suite.

26. Indices are now retrieved by exact name, [#2465](https://github.com/Rdatatable/data.table/issues/2465). This prevents usage of wrong indices as well as unexpected row reordering in join results. Thanks to @pannnda for reporting and providing a reproducible example and to @MarkusBonsch for fixing.

27. `setnames` of whole table when original table had `NA` names skipped replacing those, [#2475](https://github.com/Rdatatable/data.table/issues/2475). Thanks to @franknarf1 and [BenoitLondon on StackOverflow](https://stackoverflow.com/questions/47228836/) for the report and @MichaelChirico for fixing.

28. CJ() works with multiple empty vectors now [#2511](https://github.com/Rdatatable/data.table/issues/2511). Thanks to @MarkusBonsch for fixing.

29. `:=` assignment of one vector to two or more columns, e.g. `DT[, c("x", "y") := 1:10]`, failed to copy the `1:10` data causing errors later if and when those columns were updated by reference, [#2540](https://github.com/Rdatatable/data.table/issues/2540). This is an old issue ([#185](https://github.com/Rdatatable/data.table/issues/185)) that had been fixed but reappeared when code was refactored. Thanks to @patrickhowerter for the detailed report with reproducible example and to @MarkusBonsch for fixing and strengthening tests so it doesn't reappear again.

30. "Negative length vectors not allowed" error when grouping `median` and `var` fixed, [#2046](https://github.com/Rdatatable/data.table/issues/2046) and [#2111](https://github.com/Rdatatable/data.table/issues/2111). Thanks to @caneff and @osofr for reporting and to @kmillar for debugging and explaining the cause.

31. Fixed a bug on Windows where `data.table`s containing non-UTF8 strings in `key`s were not properly sorted, [#2462](https://github.com/Rdatatable/data.table/issues/2462), [#1826](https://github.com/Rdatatable/data.table/issues/1826)  and [StackOverflow](https://stackoverflow.com/questions/47599934/why-doesnt-r-data-table-support-well-for-non-ascii-keys-on-windows). Thanks to @shrektan for reporting and fixing.

32. `x.` prefixes during joins sometimes resulted in a "column not found" error. This is now fixed. Closes [#2313](https://github.com/Rdatatable/data.table/issues/2313). Thanks to @franknarf1 for the MRE.

33. `setattr()` no longer segfaults when setting 'class' to empty character vector, [#2386](https://github.com/Rdatatable/data.table/issues/2386). Thanks to @hatal175 for reporting and to @MarkusBonsch for fixing.

34. Fixed cases where the result of `merge.data.table()` would contain duplicate column names if `by.x` was also in `names(y)`.
`merge.data.table()` gains the `no.dups` argument (default TRUE) to match the correpsonding patched behaviour in `base:::merge.data.frame()`. Now, when `by.x` is also in `names(y)` the column name from `y` has the corresponding `suffixes` added to it. `by.x` remains unchanged for backwards compatibility reasons.
In addition, where duplicate column names arise anyway (i.e. `suffixes = c("", "")`) `merge.data.table()` will now throw a warning to match the behaviour of `base:::merge.data.frame()`. 
Thanks to @sritchie73 for reporting and fixing [PR#2631](https://github.com/Rdatatable/data.table/pull/2631) and [PR#2653](https://github.com/Rdatatable/data.table/pull/2653)

35. `CJ()` now fails with proper error message when results would exceed max integer, [#2636](https://github.com/Rdatatable/data.table/issues/2636).

36. `NA` in character columns now display as `<NA>` just like base R to distinguish from `""` and `"NA"`.

#### NOTES

0. The license has been changed from GPL to MPL (Mozilla Public License). All contributors were consulted and approved. [PR#2456](https://github.com/Rdatatable/data.table/pull/2456) details the reasons for the change.

1. `?data.table` makes explicit the option of using a `logical` vector in `j` to select columns, [#1978](https://github.com/Rdatatable/data.table/issues/1978). Thanks @Henrik-P for the note and @MichaelChirico for filing.

2. Test 1675.1 updated to cope with a change in R-devel in June 2017 related to `factor()` and `NA` levels.

3. Package `ezknitr` has been added to the whitelist of packages that run user code and should be consider data.table-aware, [#2266](https://github.com/Rdatatable/data.table/issues/2266). Thanks to Matt Mills for testing and reporting.

4. Printing with `quote = TRUE` now quotes column names as well, [#1319](https://github.com/Rdatatable/data.table/issues/1319). Thanks @jan-glx for the suggestion and @MichaelChirico for the PR.

5. Added a blurb to `?melt.data.table` explicating the subtle difference in behavior of the `id.vars` argument vis-a-vis its analog in `reshape2::melt`, [#1699](https://github.com/Rdatatable/data.table/issues/1699). Thanks @MichaelChirico for uncovering and filing.

6. Added some clarification about the usage of `on` to `?data.table`, [#2383](https://github.com/Rdatatable/data.table/issues/2383). Thanks to @peterlittlejohn for volunteering his confusion and @MichaelChirico for brushing things up.

7. Clarified that "data.table always sorts in `C-locale`" means that upper-case letters are sorted before lower-case letters by ordering in data.table (e.g. `setorder`, `setkey`, `DT[order(...)]`). Thanks to @hughparsonage for the pull request editing the documentation. Note this makes no difference in most cases of data; e.g. ids where only uppercase or lowercase letters are used (`"AB123"<"AC234"` is always true, regardless), or country names and words which are consistently capitalized. For example, `"America" < "Brazil"` is not affected (it's always true), and neither is `"america" < "brazil"` (always true too); since the first letter is consistently capitalized. But, whether `"america" < "Brazil"` (the words are not consistently capitalized) is true or false in base R depends on the locale of your R session. In America it is true by default and false if you i) type `Sys.setlocale(locale="C")`, ii) the R session has been started in a C locale for you which can happen on servers/services (the locale comes from the environment the R session is started in). However, `"america" < "Brazil"` is always, consistently false in data.table which can be a surprise because it differs to base R by default in most regions. It is false because `"B"<"a"` is true because all upper-case letters come first, followed by all lower case letters (the ascii number of each letter determines the order, which is what is meant by `C-locale`).

8. `data.table`'s dependency has been moved forward from R 3.0.0 (Apr 2013) to R 3.1.0 (Apr 2014; i.e. 3.5 years old). We keep this dependency as old as possible for as long as possible as requested by users in managed environments. Thanks to Jan Gorecki, the test suite from latest dev now runs on R 3.1.0 continously, as well as R-release (currently 3.4.2) and latest R-devel snapshot. [Our CRAN release procedures](https://github.com/Rdatatable/data.table/blob/master/CRAN_Release.cmd) also double check with this stated dependency before release to CRAN. The primary motivation for the bump to R 3.1.0 was allowing one new test which relies on better non-copying behaviour in that version, [#2484](https://github.com/Rdatatable/data.table/issues/2484). It also allows further internal simplifications. Thanks to @MichaelChirico for fixing another test that failed on R 3.1.0 due to slightly different behaviour of `base::read.csv` in R 3.1.0-only which the test was comparing to, [#2489](https://github.com/Rdatatable/data.table/pull/2489).

9. New vignette added: _Importing data.table_ - focused on using data.table as a dependency in R packages. Answers most commonly asked questions and promote good practices.

10. As warned in v1.9.8 release notes below in this file (on CRAN 25 Nov 2016) it has been 1 year since then and so use of `options(datatable.old.unique.by.key=TRUE)` to restore the old default is now deprecated with warning. The new warning states that this option still works and repeats the request to pass `by=key(DT)` explicitly to `unique()`, `duplicated()`, `uniqueN()` and `anyDuplicated()` and to stop using this option. In another year, this warning will become error. Another year after that the option will be removed.

### Changes in v1.10.4-3  (on CRAN 20 Oct 2017)

1. Fixed crash/hang on MacOS when `parallel::mclapply` is used and data.table is merely loaded, [#2418](https://github.com/Rdatatable/data.table/issues/2418). Oddly, all tests including test 1705 (which tests `mclapply` with data.table) passed fine on CRAN. It appears to be some versions of MacOS or some versions of libraries on MacOS, perhaps. Many thanks to Martin Morgan for reporting and confirming this fix works. Thanks also to @asenabouth, Joe Thorley and Danton Noriega for testing, debugging and confirming that automatic parallelism inside data.table (such as `fwrite`) works well even on these MacOS installations. See also news items below for 1.10.4-1 and 1.10.4-2.


### Changes in v1.10.4-2  (on CRAN 12 Oct 2017)

1. OpenMP on MacOS is now supported by CRAN and included in CRAN's package binaries for Mac. But installing v1.10.4-1 from source on MacOS failed when OpenMP was not enabled at compile time, [#2409](https://github.com/Rdatatable/data.table/issues/2409). Thanks to Liz Macfie and @fupangpangpang for reporting. The startup message when OpenMP is not enabled has been updated.

2. Two rare potential memory faults fixed, thanks to CRAN's automated use of latest compiler tools; e.g. clang-5 and gcc-7


### Changes in v1.10.4-1  (on CRAN 09 Oct 2017)

1. The `nanotime` v0.2.0 update on CRAN 22 June 2017 changed from `integer64` to `S4` and broke `fwrite` of `nanotime` columns. Fixed to work with `nanotime` both before and after v0.2.0.

2. Pass R-devel changes related to `deparse(,backtick=)` and `factor()`.

3. Internal `NAMED()==2` now `MAYBE_SHARED()`, [#2330](https://github.com/Rdatatable/data.table/issues/2330). Back-ported to pass under the stated dependency, R 3.0.0.

4. Attempted improvement on Mac-only when the `parallel` package is used too (which forks), [#2137](https://github.com/Rdatatable/data.table/issues/2137). Intel's OpenMP implementation appears to leave threads running after the OpenMP parallel region (inside data.table) has finished unlike GNU libgomp. So, if and when `parallel`'s `fork` is invoked by the user after data.table has run in parallel already, instability occurs. The problem only occurs with Mac package binaries from CRAN because they are built by CRAN with Intel's OpenMP library. No known problems on Windows or Linux and no known problems on any platform when `parallel` is not used. If this Mac-only fix still doesn't work, call `setDTthreads(1)` immediately after `library(data.table)` which has been reported to fix the problem by putting `data.table` into single threaded mode earlier.

5. When `fread()` and `print()` see `integer64` columns are present but package `bit64` is not installed, the warning is now displayed as intended. Thanks to a question by Santosh on r-help and forwarded by Bill Dunlap.


### Changes in v1.10.4  (on CRAN 01 Feb 2017)

#### BUG FIXES

1. The new specialized `nanotime` writer in `fwrite()` type punned using `*(long long *)&REAL(column)[i]` which, strictly, is undefined behavour under C standards. It passed a plethora of tests on linux (gcc 5.4 and clang 3.8), win-builder and 6 out 10 CRAN flavours using gcc. But failed (wrong data written) with the newest version of clang (3.9.1) as used by CRAN on the failing flavors, and solaris-sparc. Replaced with the union method and added a grep to CRAN_Release.cmd.


### Changes in v1.10.2  (on CRAN 31 Jan 2017)

#### NEW FEATURES

1. When `j` is a symbol prefixed with `..` it will be looked up in calling scope and its value taken to be column names or numbers.
    ```R
    myCols = c("colA","colB")
    DT[, myCols, with=FALSE]
    DT[, ..myCols]              # same
    ```
   When you see the `..` prefix think _one-level-up_ like the directory `..` in all operating systems meaning the parent directory. In future the `..` prefix could be made to work on all symbols apearing anywhere inside `DT[...]`. It is intended to be a convenient way to protect your code from accidentally picking up a column name. Similar to how `x.` and `i.` prefixes (analogous to SQL table aliases) can already be used to disambiguate the same column name present in both `x` and `i`. A symbol prefix rather than a `..()` _function_ will be easier for us to optimize internally and more convenient if you have many variables in calling scope that you wish to use in your expressions safely. This feature was first raised in 2012 and long wished for, [#633](https://github.com/Rdatatable/data.table/issues/633). It is experimental.

2. When `fread()` or `print()` see `integer64` columns are present, `bit64`'s namespace is now automatically loaded for convenience.

3. `fwrite()` now supports the new [`nanotime`](https://cran.r-project.org/package=nanotime) type by Dirk Eddelbuettel, [#1982](https://github.com/Rdatatable/data.table/issues/1982). Aside: `data.table` already automatically supported `nanotime` in grouping and joining operations via longstanding support of its underlying `integer64` type.

4. `indices()` gains a new argument `vectors`, default `FALSE`. This strsplits the index names by `__` for you, [#1589](https://github.com/Rdatatable/data.table/issues/1589).
    ```R
    DT = data.table(A=1:3, B=6:4)
    setindex(DT, B)
    setindex(DT, B, A)
    indices(DT)
    [1] "B"    "B__A"
    indices(DT, vectors=TRUE)
    [[1]]
    [1] "B"
    [[2]]
    [1] "B" "A"
    ```

#### BUG FIXES

1. Some long-standing potential instability has been discovered and resolved many thanks to a detailed report from Bill Dunlap and Michael Sannella. At C level any call of the form `setAttrib(x, install(), allocVector())` can be unstable in any R package. Despite `setAttrib()` PROTECTing its inputs, the 3rd argument (`allocVector`) can be executed first only for its result to to be released by `install()`'s potential GC before reaching `setAttrib`'s PROTECTion of its inputs. Fixed by either PROTECTing or pre-`install()`ing. Added to CRAN_Release.cmd procedures: i) `grep`s to prevent usage of this idiom in future and ii) running data.table's test suite with `gctorture(TRUE)`.

2. A new potential instability introduced in the last release (v1.10.0) in GForce optimized grouping has been fixed by reverting one change from malloc to R_alloc. Thanks again to Michael Sannella for the detailed report.

3. `fwrite()` could write floating point values incorrectly, [#1968](https://github.com/Rdatatable/data.table/issues/1968). A thread-local variable was incorrectly thread-global. This variable's usage lifetime is only a few clock cycles so it needed large data and many threads for several threads to overlap their usage of it and cause the problem. Many thanks to @mgahan and @jmosser for finding and reporting.

#### NOTES

1. `fwrite()`'s `..turbo` option has been removed as the warning message warned. If you've found a problem, please [report it](https://github.com/Rdatatable/data.table/issues).

2. No known issues have arisen due to `DT[,1]` and `DT[,c("colA","colB")]` now returning columns as introduced in v1.9.8. However, as we've moved forward by setting `options('datatable.WhenJisSymbolThenCallingScope'=TRUE)` introduced then too, it has become clear a better solution is needed. All 340 CRAN and Bioconductor packages that use data.table have been checked with this option on. 331 lines would need to be changed in 59 packages. Their usage is elegant, correct and recommended, though. Examples are `DT[1, encoding]` in quanteda and `DT[winner=="first", freq]` in xgboost. These are looking up the columns `encoding` and `freq` respectively and returning them as vectors. But if, for some reason, those columns are removed from `DT` and `encoding` or `freq` are still variables in calling scope, their values in calling scope would be returned. Which cannot be what was intended and could lead to silent bugs. That was the risk we were trying to avoid. <br>
`options('datatable.WhenJisSymbolThenCallingScope')` is now removed. A migration timeline is no longer needed. The new strategy needs no code changes and has no breakage. It was proposed and discussed in point 2 [here](https://github.com/Rdatatable/data.table/issues/1188#issuecomment-127824969), as follows.<br>
When `j` is a symbol (as in the quanteda and xgboost examples above) it will continue to be looked up as a column name and returned as a vector, as has always been the case.  If it's not a column name however, it is now a helpful error explaining that data.table is different to data.frame and what to do instead (use `..` prefix or `with=FALSE`).  The old behaviour of returning the symbol's value in calling scope can never have been useful to anybody and therefore not depended on. Just as the `DT[,1]` change could be made in v1.9.8, this change can be made now. This change increases robustness with no downside. Rerunning all 340 CRAN and Bioconductor package checks reveal 2 packages throwing the new error: partools and simcausal. Their maintainers have been informed that there is a likely bug on those lines due to data.table's (now remedied) weakness. This is exactly what we wanted to reveal and improve.

3. As before, and as we can see is in common use in CRAN and Bioconductor packages using data.table, `DT[,myCols,with=FALSE]` continues to lookup `myCols` in calling scope and take its value as column names or numbers.  You can move to the new experimental convenience feature `DT[, ..myCols]` if you wish at leisure.


### Changes in v1.10.0  (on CRAN 3 Dec 2016)

#### BUG FIXES

1. `fwrite(..., quote='auto')` already quoted a field if it contained a `sep` or `\n`, or `sep2[2]` when `list` columns are present. Now it also quotes a field if it contains a double quote (`"`) as documented, [#1925](https://github.com/Rdatatable/data.table/issues/1925). Thanks to Aki Matsuo for reporting. Tests added. The `qmethod` tests did test escaping embedded double quotes, but only when `sep` or `\n` was present in the field as well to trigger the quoting of the field.

2. Fixed 3 test failures on Solaris only, [#1934](https://github.com/Rdatatable/data.table/issues/1934). Two were on both sparc and x86 and related to a `tzone` attribute difference between `as.POSIXct` and `as.POSIXlt` even when passed the default `tz=""`. The third was on sparc only: a minor rounding issue in `fwrite()` of 1e-305.

3. Regression crash fixed when 0's occur at the end of a non-empty subset of an empty table, [#1937](https://github.com/Rdatatable/data.table/issues/1937). Thanks Arun for tracking down. Tests added. For example, subsetting the empty `DT=data.table(a=character())` with `DT[c(1,0)]` should return a 1 row result with one `NA` since 1 is past the end of `nrow(DT)==0`, the same result as `DT[1]`.

4. Fixed newly reported crash that also occurred in old v1.9.6 when `by=.EACHI`, `nomatch=0`, the first item in `i` has no match AND `j` has a function call that is passed a key column, [#1933](https://github.com/Rdatatable/data.table/issues/1933). Many thanks to Reino Bruner for finding and reporting with a reproducible example. Tests added.

5. Fixed `fread()` error occurring for a subset of Windows users: `showProgress is not type integer but type 'logical'.`, [#1944](https://github.com/Rdatatable/data.table/issues/1944) and [#1111](https://github.com/Rdatatable/data.table/issues/1111). Our tests cover this usage (it is just default usage), pass on AppVeyor (Windows), win-builder (Windows) and CRAN's Windows so perhaps it only occurs on a specific and different version of Windows to all those. Thanks to @demydd for reporting. Fixed by using strictly `logical` type at R level and `Rboolean` at C level, consistently throughout.

6. Combining `on=` (new in v1.9.6) with `by=` or `keyby=` gave incorrect results, [#1943](https://github.com/Rdatatable/data.table/issues/1943). Many thanks to Henrik-P for the detailed and reproducible report. Tests added.

7. New function `rleidv` was ignoring its `cols` argument, [#1942](https://github.com/Rdatatable/data.table/issues/1942). Thanks Josh O'Brien for reporting. Tests added.

#### NOTES

1. It seems OpenMP is not available on CRAN's Mac platform; NOTEs appeared in [CRAN checks](https://cran.r-project.org/web/checks/check_results_data.table.html) for v1.9.8. Moved `Rprintf` from `init.c` to `packageStartupMessage` to avoid the NOTE as requested urgently by Professor Ripley. Also fixed the bad grammar of the message: 'single threaded' now 'single-threaded'. If you have a Mac and run macOS or OS X on it (I run Ubuntu on mine) please contact CRAN maintainers and/or Apple if you'd like CRAN's Mac binary to support OpenMP. Otherwise, please follow [these instructions for OpenMP on Mac](https://github.com/Rdatatable/data.table/wiki/Installation) which people have reported success with.

2. Just to state explicitly: data.table does not now depend on or require OpenMP. If you don't have it (as on CRAN's Mac it appears but not in general on Mac) then data.table should build, run and pass all tests just fine.

3. There are now 5,910 raw tests as reported by `test.data.table()`. Tests cover 91% of the 4k lines of R and 89% of the 7k lines of C. These stats are now known thanks to Jim Hester's [Covr](https://CRAN.R-project.org/package=covr) package and [Codecov.io](https://codecov.io/). If anyone is looking for something to help with, creating tests to hit the missed lines shown by clicking the `R` and `src` folders at the bottom [here](https://codecov.io/github/Rdatatable/data.table?branch=master) would be very much appreciated.

4. The FAQ vignette has been revised given the changes in v1.9.8. In particular, the very first FAQ.

5. With hindsight, the last release v1.9.8 should have been named v1.10.0 to convey it wasn't just a patch release from .6 to .8 owing to the 'potentially breaking changes' items. Thanks to @neomantic for correctly pointing out. The best we can do now is now bump to 1.10.0.


### Old news from v1.9.8 (Nov 2016) back to v1.2 (Aug 2008) has been moved to [NEWS.0.md](https://github.com/Rdatatable/data.table/blob/master/NEWS.0.md)


