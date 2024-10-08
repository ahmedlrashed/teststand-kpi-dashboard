![](img/ReportHeader.jpg)

**PURPOSE:** Extract the manufacturing data from TestStand generated Access mdb file. Build a Power BI dashboard app for Test engineers and Mfg managers to uncover process trends and statistical control metrics. Background information on TestStand's native database logger and schema can be found HERE.

1.  **IMPORT AND TRANSFORM DATA**

![](img/niSchema.png)

We create a System DSN using ODBC to connect to the local MS Access database file. Then we use Power BI's native data importer to get the relevant tables needed:
-   PROP_RESULT

-   STEP_RESULT

-   STEP_SEQCALL

-   UUT_RESULT

However, these tables are not useful to Mfg or Test engineers in their current format. We construct a new aggregated table using the folloiwng SQL query:

```
SELECT 
  STEP_SEQCALL.SEQUENCE_FILE_PATH, 
  UUT_RESULT.ID, 
  UUT_RESULT.START_DATE_TIME, 
  UUT_RESULT.STATION_ID, 
  UUT_RESULT.UUT_SERIAL_NUMBER, 
  UUT_RESULT.TEST_SOCKET_INDEX, 
  UUT_RESULT.UUT_STATUS, 
  UUT_RESULT.EXECUTION_TIME, 
  STEP_RESULT.STEP_NAME, 
  ROUND(PROP_RESULT.DATA, 3) 
FROM 
  (
    (
      UUT_RESULT 
      LEFT JOIN STEP_RESULT ON UUT_RESULT.ID = STEP_RESULT.UUT_RESULT
    ) 
    LEFT JOIN PROP_RESULT ON STEP_RESULT.ID = PROP_RESULT.STEP_RESULT
  ) 
  LEFT JOIN STEP_SEQCALL ON STEP_SEQCALL.STEP_RESULT = STEP_RESULT.STEP_PARENT 
WHERE 
  UUT_STATUS <> 'Terminated' 
  and UUT_STATUS <> 'Error' 
  and STATION_ID is not NULL 
  and STEP_RESULT.STEP_TYPE = 'NumericLimitTest' 
  and PROP_RESULT.TYPE_NAME = 'NumericLimitTest' 
ORDER BY 
  START_DATE_TIME, 
  TEST_SOCKET_INDEX, 
  STEP_RESULT.ORDER_NUMBER ASC

```

**NOTE:** The extra parentheses are to make the query compatible with MS Access ODBC driver syntax.

Our final data model is shown below:

![](img/teststand-data-model.jpg)

2.  **KPI SUMMARY**

![](img/KPIReport.jpg)

**Slicers:**
-   Date Range: Filter to show only data from selected day, week, month, or year.

-   Sequence Name: Filters to show only data from selected TestStand test sequence.

-   Test Name: Filters to show only data from selected TestStand test step.
  
**Visuals:**
-   KPI Cards: Displays statistical test metrics for selected test sequence and test step name.

-   Test Data (Selected Name): Displays line chart of Test_Value for selected test sequence/step-name.

-   Data Histogram (Selected Name): Displays histogram of Test Data bins for selected test sequence/step-name.

-   Top Failure Modes: Displays count of "Failed" results for each Test Step Name.

-   Top Error Modes: Displays count of "Error" or "Terminated" results for each Test Step Name.

**Calculations:**
-   ```ThreeSigma = 3*STDEV.P(TestResults[Test_Value])```
-   ```LowerControlLimit = AVERAGE(TestResults[Test_Value])-[ThreeSigma]```
-   ```UpperControlLimit = AVERAGE(TestResults[Test_Value])+[ThreeSigma]```
```
Count Failure Modes = 
    COUNTROWS(
        FILTER(
            STEP_RESULT,
            [STATUS] = "Failed"
        )
    )
```
```
Count Error Modes = 
    COUNTROWS(
        FILTER(
            STEP_RESULT,
            [STATUS] = "Error" || [STATUS] = "Terminated"
        )
    )
```

3.  **MFG CAPACITY**

![](img/CapacityReport.jpg)

**Slicers:**
-   Date Range: Filter to show only data from selected day, week, month, or year.

-   Sequence Name: Filters to show only data from selected TestStand test sequence.
  
**Visuals:**
-   KPI Cards: Displays statistical process metrics for selected test sequences.

-   Test Time (Selected Sequence): Displays line chart of Median_Test_Time for selected test sequence.

-   Time Histogram (Selected Name): Displays histogram of Test Time bins for selected test sequence.

-   Total Units Produced per Day: Displays vertical bar chart of how many distinct units were produced each day for each test sequence.

-   Total Units by Sequence: Displays horizontal bar chart of how many distinct units were produced over the entire date range for each test sequence.

**Calculations:**
***
*PRODUCTION CAPACITY*
```
MFG Period = 
VAR _max =
    MAXX ( ALLSELECTED ( 'TestResults' ), 'TestResults'[Test_Start] )
VAR _min =
    MINX ( ALLSELECTED ( 'TestResults' ), TestResults[Test_Start] )
RETURN
    DATEDIFF ( _min, _max, DAY ) + 1
```
```
Potential Units Prod = 
    [MFG Period]                                // # of Days in slicer
    * 2                                         // # of shifts in a day
    * 7                                         // # of work hours in a shift
    * 3600                                      // # of seconds in an hour
    / ( MEDIAN(TestResults[Test_TIME]) + 120 )  // divide by Average-Test-Time (including 2 minutes to swich over)
```
-   ```Actual Units Prod = DISTINCTCOUNT(UUT_RESULT[UUT_SERIAL_NUMBER])```
-   ```Capacity Utilization = [Actual Units Prod] / [Potential Units Prod]```
***
*PRODUCTION YIELDS*
```
Total Yield = 
// Calculate all unique serial numbers (excluding dummy SN's with digits != 7)
VAR distinctCountTotal =
    CALCULATE (
        DISTINCTCOUNT ( UUT_RESULT[UUT_SERIAL_NUMBER] ),
        LEN ( UUT_RESULT[UUT_SERIAL_NUMBER] ) = 7
    )
RETURN distinctCountTotal
```
```
Total Pass = 
VAR distinctCountPass =
    CALCULATE (
        DISTINCTCOUNT ( UUT_RESULT[UUT_SERIAL_NUMBER] ),
        LEN ( UUT_RESULT[UUT_SERIAL_NUMBER] ) = 7,
        UUT_RESULT[UUT_STATUS] = "Passed"
    )
RETURN distinctCountPass
```

-   ```Total Scrap = [Total Yield] - [Total Pass]```
```
Total Fail = 
// Calculate unique serial numbers of failed products (excluding dummy SN's with digits != 7)
VAR distinctCountFail =
    CALCULATE (
        DISTINCTCOUNT ( UUT_RESULT[UUT_SERIAL_NUMBER] ),
        LEN ( UUT_RESULT[UUT_SERIAL_NUMBER] ) = 7,
        UUT_RESULT[UUT_STATUS] = "Failed"
    )
RETURN distinctCountFail
```
-   ```First Pass Count = [Total Yield] - [Total Fail]```
-   ```First Pass Yield = DIVIDE([First Pass Count], [Total Yield])```

-   ```Throughput Yield = DIVIDE ( [Total Pass], [Total Yield] )```

-   ```Scrap Rate = DIVIDE([Total Scrap], [Total Pass])```


