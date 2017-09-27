---
layout: post
title:  "Performance comparison of PostgreSQL connectors in Matlab, Part II: retrieving scalar data"
date:   2017-09-20 15:37:13 +0300
authors: Peter Gagarinov & Ilya Rublev
---

<script type="text/javascript">
function toggleMe(a){
var e=document.getElementById(a);
if(!e)return true;
if(e.style.display=="none"){
e.style.display="block"
} else {
e.style.display="none"
}
return false;
}
</script>

# Performance comparison of PostgreSQL connectors in Matlab, Part II: retrieving scalar data

In [**Part I**](https://alliedtesting.github.io/pgmex-blog/2017/06/29/performance-comparison-of-postgresql-connectors-in-matlab-part-I/)
of this paper we started our investigation of **PostgreSQL** connectors in Matlab. Namely, we compared the performance of different
ways to insert data into the **PostgreSQL** database. Part of these ways use **Matlab Database Toolbox** (working with **PosgteSQL**
via a direct JDBC connection). Other ones are implemented in [**PgMex library**](http://pgmex.alliedtesting.com) 
(providing a connection to **PostgreSQL** via libpq library). Here we continue the comparison of **Matlab Database Toolbox** and
[**PgMex library**](http://pgmex.alliedtesting.com) considering data retrieval.
The given part of this paper covers retrieving only in the most simple case of scalar data, both of numeric and non-numeric
types.

The data we used for performance testing discussed below is the same as was used for the case of data inserting. More precisely,
this data is based on real daily prices of stocks on some exchanges. Taking into account the nature of such a financial data
no wonder its processing requires a possibility to retrieve it in huge amounts at low time costs. Below we reveal some latent
restrictions (concerning both performance, volumes and type of data to be processed) that do not allow to use **Matlab Database
Toolbox** in our projects dealing with data of such a type. Another solution, [**PgMex library**](http://pgmex.alliedtesting.com),
was developed by our team specially to overcome these restrictions and to allow us to work with real big data.
<!--end_of_excerpt-->

## Methods of data retrieval in **Matlab Database Toolbox**

There are two ways to retrieve data from the database by means of **Matlab Database Toolbox**. Firstly, one may execute a selection query
by calling **exec** method of **database.jdbc.connection** class returning an object of **database.jdbc.cursor** class and then to retrieve results by calling
**fetch** method of this **database.jdbc.cursor** object. Secondly, it is possible to get results immediately by calling **fetch** method of
**database.jdbc.database** class. There are certain differences concerning these ways not relevant for our investigation (please take a reference
at [Matlab Help](https://www.mathworks.com/help/database/ug/fetch.html) for more details). We investigate here the first way as a more
flexible one.

There are three important parameters set by means of **setdbprefs** function and determining the format, in which
**Matlab Database Toolbox** returns results of data retrieval. They are **DataReturnFormat**, **NullNumberRead** and
**NullStringRead**. What concerns **DataReturnFormat**, this parameter affects the way data is represented in Matlab,
and may be equal to one of three possible values: 'cellarray', 'numeric', and 'structure'. If **DataReturnFormat** equals
'cellarray', data is returned as a two-dimensional cell array, with each cell containing the value of some field for some tuple
(the field corresponds to the column number of this cell within the cell array, while the tuple is determined by its row number).
**NullNumberRead** and **NullStringRead** determine values that are substituted instead of NULL values (in the case these
values are to be returned as numbers and strings, respectively).
If **DataReturnFormat** is 'numeric', data is returned as a **double** matrix with all NULL values and values of all fields having
non-numeric type equal to the number given by **NullNumberRead** parameter. At last, if **DataReturnFormat** is equal to 'structure',
then data is returned as a structure with fields corresponding to the fields of retrieved table (and having the same names as in
this table). The representation of NULL values in Matlab is configured by **NullNumberRead** and **NullStringRead**
(exactly as above in the case when **DataReturnFormat** equals 'cellarray').

## Methods of data retrieval in **PgMex**

The same task is solved via [**PgMex**](http://pgmex.alliedtesting.com) by a way very similar to the one described in the
previous section. This library has two commands, namely, [**exec**](http://pgmex.alliedtesting.com/#exec) for execution of
queries returning a pointer to **PGresult** structure with results and [**getf**](http://pgmex.alliedtesting.com/#getf) for
transforming the mentioned **PGresult** structure into a Matlab friendly format. More precisely,
[**getf**](http://pgmex.alliedtesting.com/#getf) returns a list of structures, one structure per each field of retrieved table.
Each structure has three fields:

- **valueVec** with values extracted from the database;
- **isNullVec** of logical type serving as an indicator of value elements being NULL;
- **isValueNullVec** of logical type indicating if values for some tuples are NULL.

**isNullVec** and **isValueNullVec** differ only in the case some field has an [**array type**](https://www.postgresql.org/docs/9.6/static/arrays.html).
But in the present Part II of the paper we deal only with fields of scalar types (the case of array types is an object of further
investigations). Hence, just for simplicity we may assume that **isNullVec** and **isValueNullVec** are the same.

[**PgMex**](http://pgmex.alliedtesting.com) provides much more safe and consistent way of representing NULLs
in Matlab in comparison with the one available in **Matlab Database Toolbox** through **NullNumberRead** and
**NullStringRead** parameters. In fact, if **NullNumberRead** is equal, for instance, to **NaN**, then ordinary **NaN**
values may be easily confused with NULLs. The same is valid for **NullStringRead** parameter. Thus, for **Matlab Database Toolbox**
one need to assume in advance that some numerical or, respectively, string values cannot occur as "ordinary" ones.

## Types of experiments for comparison between **Matlab Database Toolbox** and **PgMex**

In the next subsections below we compare the methods **exec** and **fetch** from **Matlab Database Toolbox**
with [**exec**](http://pgmex.alliedtesting.com/#exec) and [**getf**](http://pgmex.alliedtesting.com/#getf)
from [**PgMex**](http://pgmex.alliedtesting.com). As was already mentioned above, data for experiments is 
the same as in [Part I](https://alliedtesting.github.io/pgmex-blog/2017/06/29/performance-comparison-of-postgresql-connectors-in-matlab-part-I/#experiment-conditions-for-comparison-between-matlab-database-toolbox-and-pgmex)
(see this link also for other experiment conditions) and is taken from real stock prices.

Let us recall very shortly what are the fields of this data (the types pointed in the terms of **PostgreSQL**):

- **t\_data** of **date** type determining trading dates
- **inst\_id** of **int4** with internal identifiers of instruments (unique for each stock)
- **price\_low** of **float4** with low prices of stocks (adjusted for splits)
- **price\_open** of **float4** with open prices of stocks (adjusted for splits)
- **price\_close** of **float4** with close prices of stocks (adjusted for splits)
- **price\_close\_ini** of **float4** with close prices of stocks unadjusted for splits
- **price\_high** of **float4** with high prices of stocks (adjusted for splits)
- **price\_close_opt** of **float4** with close prices of stocks unadjusted for splits and used in the Black-Sholes formula
- **volume** of **int8** with traded volumes
- **calc\_date** of **date** type with dates when calculations were done

All the experiments considered below are divided by the types of fields retrieved from the database simultaneously 
(for each type we retrieve only some subset of fields mentioned above). We discuss two cases: pure numerical data
and data with timestamps.

## Retrieving scalar numericals

Retrieving only fields of scalar numeric types allows to use all three formats of returning results for **Matlab Database Toolbox**
determined by **DataReturnFormat** parameter including 'numeric'. And at first glance **DataReturnFormat** being equal to 'numeric'
seems rather reasonable choice from the viewpoint of data types mentioned in the previous subsection. But hence
we are forced to exclude fields of non-numeric types from our query (because all these values will be lost in results of **fetch**,
the latter simply changes them on the value of **NullNumberRead**). And these are the fields **t\_data** and **calc\_date**, because
they are of **date** type and cannot be returned in a numerical matrix in spite of the fact that in Matlab they are naturally
represented by serial date numbers. **Matlab Database Toolbox** always returns timestamps as strings (and **DataReturnFormat**
equal to 'numeric' is not applicable if someone likes to retrieve timestamps, the question how timestamps may be retrieved from
the database is discussed in the subsection immediately following this one, see below).

But there is another significant shortcoming of **Matlab Database Toolbox** for all available values of **DataReturnFormat**
simultaneously. It caused by the fact that **fetch** returns all numericals as having **double** Matlab type. Thus, everything is
converted by **fetch** into **double** Matlab type, even 8-byte integers. Firstly, there is no any guarantee that there is no data loss.
There is no problem for the fields containing prices like **price\_low** or **price\_close**. But the guarantee is automatically not there
when we consider the field **volume**. This is because whether a field value becomes incorrect after a type transformation or not depends
significantly on the value itself. As a result we cannot simple cast all values to **double** type of Matlab, because size of
both **double** and **int64** Matlab types (the latter corresponding to **int8** in **PostgreSQL**) equals 8 bytes. Thus, if we
try for example to cast the maximal possible value of **int64** equal to 9223372036854775807 to **double**, you obtain
9223372036854775800 instead of the original value.

But even in the case there is no data loss, the memory size necessary to represent data within Matlab increases essentially
(almost twice). It can be easily seen that the situation may be potentially even worse if we have
fields of logical type (in this case logical values returned by **fetch** would occupy in Matlab 8
times more memory than in their initial form).

But this increasing of memory while retrieving data by **fetch** leads to a shortage of memory when the total number of tuples to be returned
is significant. This can be clearly seen on the following pictures made for **DataReturnFormat** equal to 'numeric',
'cellarray' and 'structure', respectively.

![Retrieving of scalar numericals, 'numeric' mode](pictures/compareRetrieveForNumScalarsAsNumeric_traj.jpeg)
![Retrieving of scalar numericals, 'cellarray' mode](pictures/compareRetrieveForNumScalarsAsCellArray_traj.jpeg)
![Retrieving of scalar numericals, 'structure' mode](pictures/compareRetrieveForNumScalarsAsStruct_traj.jpeg)
<!---
![Retrieving of scalar numericals, 'numeric' mode]({{ site.baseurl }}/img/posts{{ page.path | remove: '_posts' | remove: '.md' }}/compareRetrieveForNumScalarsAsNumeric_traj.jpeg)
![Retrieving of scalar numericals, 'cellarray' mode]({{ site.baseurl }}/img/posts{{ page.path | remove: '_posts' | remove: '.md' }}/compareRetrieveForNumScalarsAsCellArray_traj.jpeg)
![Retrieving of scalar numericals, 'cellarray' mode]({{ site.baseurl }}/img/posts{{ page.path | remove: '_posts' | remove: '.md' }}/compareRetrieveForNumScalarsAsStruct_traj.jpeg)
-->

It may be seen that the red graphs on all the three pictures above (they correspond to **fetch**) "break" when a number of tuples reaches
1200000. This is because **exec** throws exceptions in all modes determined by **DataReturnFormat**. For 'numeric' mode this
exception is:

<!---
<a style="cursor:pointer;" onclick="return toggleMe('para1')">
See more...
</a>
-->

<div class="language-matlab highlighter-rouge">
   <pre class="highlight">
Exception for function fetch, number of tuples 1200000

Caused by:
    Error using database.jdbc.cursor (line 229)
    Java exception occurred: 
    java.lang.OutOfMemoryError: GC overhead limit exceeded<a onclick="return toggleMe('para1')">[see more...]</a>
      <div id="para1" style="display:none;">
    	at java.util.zip.ZipCoder.getBytes(Unknown Source)
    	at java.util.zip.ZipFile.getEntry(Unknown Source)
    	at java.util.jar.JarFile.getEntry(Unknown Source)
    	at java.util.jar.JarFile.getJarEntry(Unknown Source)
    	at sun.misc.URLClassPath$JarLoader.getResource(Unknown Source)
    	at sun.misc.URLClassPath.getResource(Unknown Source)
    	at java.net.URLClassLoader$1.run(Unknown Source)
    	at java.net.URLClassLoader$1.run(Unknown Source)
    	at java.security.AccessController.doPrivileged(Native Method)
    	at java.net.URLClassLoader.findClass(Unknown Source)
    	at java.lang.ClassLoader.loadClass(Unknown Source)
    	at sun.misc.Launcher$AppClassLoader.loadClass(Unknown Source)
    	at java.lang.ClassLoader.loadClass(Unknown Source)
    	at java.lang.ClassLoader.loadClass(Unknown Source)
    	at org.postgresql.core.v3.QueryExecutorImpl.processResults(QueryExecutorImpl.java:2121)
    	at org.postgresql.core.v3.QueryExecutorImpl.execute(QueryExecutorImpl.java:288)
    	at org.postgresql.jdbc.PgStatement.executeInternal(PgStatement.java:430)
    	at org.postgresql.jdbc.PgStatement.execute(PgStatement.java:356)
    	at org.postgresql.jdbc.PgStatement.executeWithFlags(PgStatement.java:303)
    	at org.postgresql.jdbc.PgStatement.executeCachedSql(PgStatement.java:289)
    	at org.postgresql.jdbc.PgStatement.executeWithFlags(PgStatement.java:266)
    	at org.postgresql.jdbc.PgStatement.executeQuery(PgStatement.java:233)
    	at com.mathworks.toolbox.database.sqlExec.executeTheSelectStatement(sqlExec.java:202)
      </div>
   </pre>
</div>

As for 'cellarray' and 'structure' modes, the exception slightly differs from the above one and is as follows:

<div class="language-matlab highlighter-rouge">
   <pre class="highlight">
Exception for function fetch, number of tuples 1200000

Caused by:
    Error using database.jdbc.cursor (line 229)
    Java exception occurred: 
    java.lang.OutOfMemoryError: GC overhead limit exceeded
    	at java.util.Arrays.copyOf(Unknown Source)
    	at java.util.zip.ZipCoder.getBytes(Unknown Source)
    	at java.util.zip.ZipFile.getEntry(Unknown Source)
    	at java.util.jar.JarFile.getEntry(Unknown Source)
    	at java.util.jar.JarFile.getJarEntry(Unknown Source)
    	at sun.misc.URLClassPath$JarLoader.getResource(Unknown Source)
    	at sun.misc.URLClassPath.getResource(Unknown Source)
    	at java.net.URLClassLoader$1.run(Unknown Source)
    	at java.net.URLClassLoader$1.run(Unknown Source)
    	at java.security.AccessController.doPrivileged(Native Method)
    	at java.net.URLClassLoader.findClass(Unknown Source)
    	at java.lang.ClassLoader.loadClass(Unknown Source)
    	at sun.misc.Launcher$AppClassLoader.loadClass(Unknown Source)
    	at java.lang.ClassLoader.loadClass(Unknown Source)
    	at java.lang.ClassLoader.loadClass(Unknown Source)
    	at org.postgresql.core.v3.QueryExecutorImpl.processResults(QueryExecutorImpl.java:2121)
    	at org.postgresql.core.v3.QueryExecutorImpl.execute(QueryExecutorImpl.java:288)
    	at org.postgresql.jdbc.PgStatement.executeInternal(PgStatement.java:430)
    	at org.postgresql.jdbc.PgStatement.execute(PgStatement.java:356)
    	at org.postgresql.jdbc.PgStatement.executeWithFlags(PgStatement.java:303)
    	at org.postgresql.jdbc.PgStatement.executeCachedSql(PgStatement.java:289)
    	at org.postgresql.jdbc.PgStatement.executeWithFlags(PgStatement.java:266)
    	at org.postgresql.jdbc.PgStatement.executeQuery(PgStatement.java:233)
    	at com.mathworks.toolbox.database.sqlExec.executeTheSelectStatement(sqlExec.java:202)
   </pre>
</div>

It should be also noted that the memory size necessary for storing of retrieved data within Matlab
is different for different values of **DataReturnFormat**. Namely, for 1200000 tuples this size
is 73Mb for 'numeric' mode, 1067Mb for 'cellarray' mode and 41Mb for 'structure' mode. 'cellarray'
mode is the most expensive because we need to store each number in a separate cell, 'numeric'
mode is almost twice more expensive than 'structure' because all numbers in 'numeric' mode are
converted into **double** Matlab type.

But it is clear that all these fails have nothing in common with the format of data to be returned by **fetch**.
The reason is directly connected to a shortage of Java Heap memory for storing results of the query to be executed
by means of **exec**.

Starting with 1200000 tuples [**exec**](http://pgmex.alliedtesting.com/#exec) together with [**getf**](http://pgmex.alliedtesting.com/#getf) methods
from [**PgMex**](http://pgmex.alliedtesting.com) library provide the only way that successfully manages
to perform the task. It retrieves 2000000 tuples (122Mb for 'numeric' mode, 1778Mb for 'cellarray' mode and 69Mb for 'structure' mode)
approximately in 6 seconds.

It turns out that if we only take a range [0, 1100000] for a number of tuples (as number of tuples = 1100000 is where the red graphs end)
on average [**exec**](http://pgmex.alliedtesting.com/#exec) and [**getf**](http://pgmex.alliedtesting.com/#getf) are more than 7 times faster
than **exec** together with **fetch** from **Matlab Database Toolbox**. Below are diagrams for 'numeric', 'cellarray' and 'structure'
modes, respectively.

![Retrieving of scalar numericals, 'numeric' mode](pictures/compareRetrieveForNumScalarsAsNumeric_bar.jpeg)
![Retrieving of scalar numericals, 'cellarray' mode](pictures/compareRetrieveForNumScalarsAsCellArray_bar.jpeg)
![Retrieving of scalar numericals, 'structure' mode](pictures/compareRetrieveForNumScalarsAsStruct_bar.jpeg)
<!---
![Inserting of scalar numericals, 'numeric' mode]({{ site.baseurl }}/img/posts{{ page.path | remove: '_posts' | remove: '.md' }}/compareRetrieveForNumScalarsAsNumeric_bar.jpeg)
![Inserting of scalar numericals, 'cellarray' mode]({{ site.baseurl }}/img/posts{{ page.path | remove: '_posts' | remove: '.md' }}/compareRetrieveForNumScalarsAsCellArray_bar.jpeg)
![Inserting of scalar numericals, 'structure' mode]({{ site.baseurl }}/img/posts{{ page.path | remove: '_posts' | remove: '.md' }}/compareRetrieveForNumScalarsAsStruct_bar.jpeg)
-->

## Retrieving timestamps

The gap in performance between [**exec**](http://pgmex.alliedtesting.com/#exec) and [**getf**](http://pgmex.alliedtesting.com/#getf) on the one hand
and **exec** with **fetch** from **Matlab Database Toolbox** on the other hand becomes even greater when we
try to retrieve all the fields from the data mentioned above including the fields **t\_data** and **calc\_date** containing dates.
As already was said, **Matlab Database Toolbox** always returns values of date, time or timestamp types as strings. Besides,
these strings may be returned only via placing them into a cell array. And there are two options. Firstly, if
**DataReturnMode** equals 'cellarray', they are returned in the corresponding columns of a cell array containing all the
retrieved results. Secondly, if **DataReturnMode** is 'structure', values for **t\_data** and **calc\_date** are placed
into the respective fields of the returned structure as char cell arrays.
When it comes to [**PgMex**](http://pgmex.alliedtesting.com), no such a conversion is required, date, time and timestamp values may be processed as
serial date numbers. The results for scalar data with timestamps are as follows (the first two pictures are for
'cellarray' mode, the second two ones are for 'structure' mode):

![Retrieving of scalar data with timestamps, 'cellarray' mode](pictures/compareRetrieveForScalarsInCellAsCellArray_traj.jpeg)
![Retrieving of scalar data with timestamps, 'cellarray' mode](pictures/compareRetrieveForScalarsInCellAsCellArray_bar.jpeg)
<!---
![Retrieving of scalar data with timestamps, 'cellarray' mode]({{ site.baseurl }}/img/posts{{ page.path | remove: '_posts' | remove: '.md' }}/compareRetrieveForScalarsInCellAsCellArray_traj.jpeg)
![Retrieving of scalar data with timestamps, 'cellarray' mode]({{ site.baseurl }}/img/posts{{ page.path | remove: '_posts' | remove: '.md' }}/compareInsertForScalarsInCellAsCellArray_bar.jpeg)
-->
![Retrieving of scalar data with timestamps, 'structure' mode](pictures/compareRetrieveForScalarsInCellAsStruct_traj.jpeg)
![Retrieving of scalar data with timestamps, 'structure' mode](pictures/compareRetrieveForScalarsInCellAsStruct_bar.jpeg)
<!---
![Retrieving of scalar data with timestamps, 'structure' mode]({{ site.baseurl }}/img/posts{{ page.path | remove: '_posts' | remove: '.md' }}/compareRetrieveForScalarsInCellAsStruct_traj.jpeg)
![Retrieving of scalar data with timestamps, 'structure' mode]({{ site.baseurl }}/img/posts{{ page.path | remove: '_posts' | remove: '.md' }}/compareInsertForScalarsInCellAsStruct_bar.jpeg)
-->

It can be easily seen that **exec** reaches 800000 tuples (894Mb for 'cellarray' mode and 40Mb for 'structure' mode) (thanks to absence of converting
numbers to some types of greater size), but fails to retrieve 900000 tuples (1000Mb for 'cellarray' mode and 45Mb for 'structure' mode) by throwing
these exceptions (the first one is for 'cellarray' mode, the second one is for 'structure' mode):

```matlab
Exception for function fetch, number of tuples 900000

Caused by:
    Error using database.jdbc.cursor/fetch (line 199)
    Java exception occurred: 
    java.lang.OutOfMemoryError: GC overhead limit exceeded
    	at org.postgresql.core.Encoding.decode(Encoding.java:184)
    	at org.postgresql.core.Encoding.decode(Encoding.java:195)
    	at org.postgresql.jdbc.PgResultSet.getString(PgResultSet.java:1904)
    	at org.postgresql.jdbc.PgResultSet.getFixedString(PgResultSet.java:2662)
    	at org.postgresql.jdbc.PgResultSet.getDouble(PgResultSet.java:2294)
    	at com.mathworks.toolbox.database.fetchTheData.dataFetchFast(fetchTheData.java:1287)
```

```matlab
Exception for function fetch, number of tuples 900000

Caused by:
    Error using database.jdbc.cursor/fetch (line 199)
    Java exception occurred: 
    java.lang.OutOfMemoryError: GC overhead limit exceeded
    	at sun.misc.FloatingDecimal.readJavaFormatString(Unknown Source)
    	at java.lang.Double.parseDouble(Unknown Source)
    	at org.postgresql.jdbc.PgResultSet.toDouble(PgResultSet.java:2920)
    	at org.postgresql.jdbc.PgResultSet.getDouble(PgResultSet.java:2294)
    	at com.mathworks.toolbox.database.fetchTheData.dataFetchFast(fetchTheData.java:1287)
```

Besides, on average the execution time for [**exec**](http://pgmex.alliedtesting.com/#exec) together with [**getf**](http://pgmex.alliedtesting.com/#getf) is approximately 20% of that time
for **exec** and **fetch** from **Matlab Database Toolbox**, at least for those volumes both function succeeded to retrieve.

## Summary

So as we have seen, the functionality provided by **Matlab Database Toolbox** for JDBC connection to **PostgreSQL** have serious restrictions
not only for inserting data, but also for its retrieval, especially in case of big data. The limits of [**PgMex**](http://pgmex.alliedtesting.com)
are sufficiently higher and this library is free from those shortcomings related to
data types conversions (because it works with native Matlab formats).

In Part III of this paper we will compare different ways to retrieve data of array types from the database using both **Matlab Database Toolbox** and
[**PgMex library**](http://pgmex.alliedtesting.com).