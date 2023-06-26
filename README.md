<a name="readme-top"></a>
<div align="center">
  <h3 align="center">TG Stored Procedure - Best Practices</h3>
</div>
<!-- TABLE OF CONTENTS -->
<details>
  <summary>Table of Contents</summary>
  <ol>
    <li><a href="#use-keyword">USE Keyword</a></li>
    <li><a href="#syntax">Syntax</a></li>
    <li><a href="#try-catch">Try Catch</a></li>
    <li><a href="#transaction">Transaction</a></li>
    <li><a href="#temporary-table-vs-table-variable">Temporary Table vs Table Variable</a></li>
    <li>
        <a href="#template">Template</a>
        <ul>
        <li><a href="#non-transactional-stored-procedure">Non Transactional Stored Procedure</a></li>
        <li><a href="#transactional-stored-procedure">Transactional Stored Procedure</a></li>
      </ul>
    </li>
    <li><a href="#reference">Reference</a></li>
  </ol>
</details>

<!-- USE !-->
## USE Keyword
When create a new Stored Procedure or alter existing Stored Procedure make sure the USE keyword is label on top of the Query / Script

Example 

```
USE DATABASE

CREATE PROCEDURE dbo.StoredProcedureName

AS
BEGIN
```

```
USE DATABASE

ALTER PROCEDURE dbo.StoredProcedureName

AS
BEGIN
```

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- Syntax !--> 
## Syntax
- Spacing for **Special Characters**
  - DECLARE @paramemter as numeric ~~(18,0)~~  ➡️  numeric **(18, 0)**
  - ~~@variable1+@variable2~~  ➡️  **@variable1 + @variable2**
  - INSERT INTO TABLE ~~(column1,column2,column3)~~  ➡️  **(column1, column2, column3)**
  - IF ~~condition=valueA~~ ➡️  **condition = valueA**
- TABLE JOIN
  - primary table should be starting point of the query, and consistently place it on the left side of the join condition
    Wrong Example
    ```
    SELECT *
    FROM tableA
    INNER JOIN tableB on tableB.foreignKey = tableA.primaryKey
    ```
    Correct Example
    ```
    SELECT *
    FROM tableA
    INNER JOIN tableB on tableA.primaryKey = tableB.foreignKey
    ```
  - use more descriptive aliases
    ```
    SELECT *
    FROM teUserMemberSIMAcc as acc / simAcc
    INNER JOIN teUserMember as mem / userMem on acc.userMemberID = mem.ID
    ```
- Aligning the code
  - aligned the code is easier to understand and read.
  - more visually organized and structured
  - ease developers to comprehend the logic and flow of the code
  - allows devs to quickly locate specific sections efficiently
  - saves time and reduces the likelihood of introducing errors during code modifications.
      
  Wrong Example
  ```
  BEGIN TRY
  SELECT * FROM TABLE
  IF value = @param
  BEGIN
  SELECT 'HERE'
  END
  ELSE
  BEGIN
  SELECT HERE
  END
  END TRY
  ```
  Correct Example
  ```
  BEGIN TRY
      SELECT * FROM TABLE
      IF value = @param
      BEGIN
          SELECT 'HERE'
      END
      ELSE
      BEGIN
          SELECT 'HERE'
      END
  END TRY
  
  ```
    
<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Try Catch

Stored Procedure must start with TRY CATCH.

```
BEGIN TRY
    DECLARE @parameter as int
    SELECT column1, column2 FROM table WHERE condition = @paramemter
END TRY
BEGIN CATCH
    PRINT 'Error occurred: ' + ERROR_MESSAGE();
END CATCH

```
<!-- Transaction !-->
## Transaction
Stored Prodcedure that excecuting transactions like UPDATE, DELETE or INSERT is important to have the following options :

### SET XACT_ABORT ON;
**XACT_ABORT** options determine the behavior of a transaction when an error occurs. if an error occurs at any point within the transaction, the entire transaction is automatically rolled back, and is essential for a more reliable error and transaction handling.
```
SET XACT_ABORT ON; -- Enable XACT_ABORT

BEGIN TRY
    BEGIN TRANSACTION;
    -- Update / Insert / Delete Statement here
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;
END CATCH;
```
TRY-CATCH does not catch compilation error that occur in the same scope
and if XACT_ABORT is not turn on and statement is running transaction, it will keep on active until the session died. Below is is an example when XACT_ABORT is ON and XACT_ABORT is OFF
```
BEGIN TRY
    BEGIN TRANSACTION;
        DELETE FROM NOEXISTSTABLE WHERE ID = 1
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;
END CATCH;

SELECT @@TRANCOUNT -- will be 1
```

```
SET XACT_ABORT ON; -- Enable XACT_ABORT

BEGIN TRY
    BEGIN TRANSACTION;
        DELETE FROM NOEXISTSTABLE WHERE ID = 1
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;
END CATCH;

SELECT @@TRANCOUNT -- will be 0
```
<!-- Catching Error !-->
## Catching Error
Our current practice to insert error log to table have some limitation 
- Performaince impact
  - might incurs a perfomance overhead due to the additional insert operation. and may impact overall performance.
- Concurrent access
  - if multiple sessions or connections execute the stored procedure concurrently and encounter erros it will cause potential blocking.

```
BEGIN TRY
    BEGIN TRANSACTION
        UPDATE table SET column1 = value1 WHERE condition = value2
    COMMIT TRANSACTION
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;

    INSERT INTO teSPErrorLog(storeprodname, errormessage, remark, createddatetime)
    SELECT ERROR_PROCEDURE(), ERROR_MESSAGE(), 'Line : ' + CAST(ERROR_LINE() as VARCHAR(10))
    + ' Number :'+CAST(ERROR_NUMBER() as VARCHAR(10)) + ' Severity :' + CAST(ERROR_SEVERITY() as VARCHAR(10))
    + ' State :' + CAST(ERROR_STATE() as VARCHAR(10)), GETDATE()
END CATCH
```
so we change our approach, instead of insert log to table, we just return the error message to our application and log at application level.
please refer sample as below and it should be apply to all stored procedure. 

```
BEGIN TRY
    BEGIN TRANSACTION
        UPDATE table SET column1 = value1 WHERE condition = value2
    COMMIT TRANSACTION
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;
		
    SELECT 'Stored Prodcedure : ' + ERROR_PROCEDURE() 
    + ' Error Message : ' + ERROR_MESSAGE()
    + ' Line : ' + CAST(ERROR_LINE() as VARCHAR(10)) 
    + ' Number : ' + CAST(ERROR_NUMBER() as VARCHAR(10))
    + ' Severity : ' + CAST(ERROR_SEVERITY() as VARCHAR(10))
    + ' State : ' + CAST(ERROR_STATE() as VARCHAR(10))
    + ' Date Time : ' + CAST(GETDATE() AS VARCHAR(20)) as stored_prod_error
END CATCH
```

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- Temp vs Variable !-->
## Temporary Table vs Table Variable
In our current approach either small or large dataset we used temporary table which is not a good practice. 
Based on the log below, sometimes temporary table were drop before stored procedure manage execute finish
and it will cause dead lock if is involve transaction. 
```
Jun 13 16:47:38 ip-10-88-53-245 server: org.springframework.jdbc.BadSqlGrammarException:
CallableStatementCallback; bad SQL grammar [{call tw_ResetPassword_v2(?, ?, ?)}];
nested exception is java.sql.SQLException: Invalid object name '#tesystemmessage'.
```
to avoid the above issue happen, we can use table variable to handle small dataset recommended below 100 rows 

Current approach
```
SELECT * INTO #tesystemmessage FROM teSystemMessage WHERE message_key = 'KEY' -- insert data to temp table

SELECT * FROM #tesystemmessage WHERE message_no = 'CODE' -- rtrieve data from temp table
```
New Approach
```
DECLARE @teSystemMessage TABLE (message_no int, message_description nvarchar(400))

INSERT INTO @teSystemMessage(message_no, message_description)
SELECT meesage_no, message_description FROM teSystemMessage WHERE message_key = 'KEY' -- insert data to table variable

SELECT message_no, message_description FROM @tesystemmessage WHERE message_no = 'CODE' - retrieve data from table variable
```

## Template

### Non Transactional Stored Procedure
```
USE DATABASE
-- =============================================
-- Author:  KONG FOOK SHEN -- Developer Name
-- Create date: 1 JUL 2023 & v60.0.1.1 -- Date Created & Branch Name>
-- Description:	TGDEV-8888 Registration <Jira Task No & Title>
-- =============================================
CREATE PROCEDURE dbo.StoredProcedureName

AS
BEGIN
	SET NOCOUNT ON;
    BEGIN TRY
 
	END TRY
	BEGIN CATCH
        SELECT 'Stored Prodcedure : ' + ERROR_PROCEDURE() 
        + ' Error Message : ' + ERROR_MESSAGE()
        + ' Line : ' + CAST(ERROR_LINE() as VARCHAR(10)) 
        + ' Number : ' + CAST(ERROR_NUMBER() as VARCHAR(10))
        + ' Severity : ' + CAST(ERROR_SEVERITY() as VARCHAR(10))
        + ' State : ' + CAST(ERROR_STATE() as VARCHAR(10))
        + ' Date Time : ' + CAST(GETDATE() AS VARCHAR(20)) as stored_prod_error
	END CATCH
END
GO
```

### Transactional Stored Procedure 
```
USE DATABASE
-- =============================================
-- Author:  KONG FOOK SHEN -- Developer Name
-- Create date: 1 JUL 2023 & v60.0.1.1 -- Date Created & Branch Name>
-- Description:	TGDEV-8888 Registration <Jira Task No & Title>
-- =============================================
CREATE PROCEDURE dbo.StoredProcedureName

AS
BEGIN  
    SET NOCOUNT ON;
	SET XACT_ABORT ON;
    BEGIN TRY
        BEGIN TRANSACTION
            -- Statement
        COMMIT TRANSACTION 
	END TRY
	BEGIN CATCH
        IF @@TRANCOUNT > 1
            ROLLBACK TRANSACTION;
 
        SELECT 'Stored Prodcedure : ' + ERROR_PROCEDURE() 
        + ' Error Message : ' + ERROR_MESSAGE()
        + ' Line : ' + CAST(ERROR_LINE() as VARCHAR(10)) 
        + ' Number : ' + CAST(ERROR_NUMBER() as VARCHAR(10))
        + ' Severity : ' + CAST(ERROR_SEVERITY() as VARCHAR(10))
        + ' State : ' + CAST(ERROR_STATE() as VARCHAR(10))
        + ' Date Time : ' + CAST(GETDATE() AS VARCHAR(20)) as stored_prod_error
    END CATCH
END
GO
```

<!-- Reference -->
## Reference

* https://learn.microsoft.com/en-us/sql/t-sql/language-elements/try-catch-transact-sql?view=sql-server-ver16)
* https://stackoverflow.com/questions/70875876/what-is-the-correct-template-to-use-with-try-catch-and-transactions
* https://www.sommarskog.se/error_handling/Part1.html
* https://www.sommarskog.se/error_handling/Part2.html
* https://www.sommarskog.se/error_handling/Part3.html
* https://www.sommarskog.se/share_data.html
* https://stackoverflow.com/questions/11857789/when-should-i-use-a-table-variable-vs-temporary-table-in-sql-server
* https://dba.stackexchange.com/questions/16385/whats-the-difference-between-a-temp-table-and-table-variable-in-sql-server/16386#16386

<p align="right">(<a href="#readme-top">back to top</a>)</p>
