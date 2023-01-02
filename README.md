
## SQL TRICKS AND NOTES

### FORMAT FOR SOLVING [chasm / fan trap] and JOINING 2 Queries/Tables to one CTE
```

WITH OnlineS (OnTotal,Yr)
AS (
	select sum(os.SubTotal)
		,cal.[Year]
	from f.OnlineSales os
	INNER JOIN dim.Calendar cal
	ON cal.[Date] = os.OrderDate
	GROUP BY cal.[Year]

),
RetailS(RtTotal,Yr) 
AS (
	select sum(rs.SubTotal)
		,cal.[Year]
	from f.RetailerSales rs
	INNER JOIN dim.Calendar cal
	ON cal.[Date] = rs.OrderDate
	GROUP BY cal.[Year]

)
SELECT OnTotal
		,RtTotal
		,ons.Yr
FROM OnlineS ons
INNER JOIN RetailS rts
ON ons.Yr = rts.Yr

```
### SQL CALENDA

```

DECLARE @StartDate  date = '20110101';
DECLARE @CutoffDate date = DATEADD(DAY, -1, DATEADD(YEAR, 4, @StartDate));

;WITH seq(n) AS 
(
  SELECT 0 UNION ALL SELECT n + 1 FROM seq
  WHERE n < DATEDIFF(DAY, @StartDate, @CutoffDate)
),
d(d) AS 
(
  SELECT DATEADD(DAY, n, @StartDate) FROM seq
),
src AS
(
  SELECT
   bkDateKey   = CAST(REPLACE(CONVERT(varchar(10), d),'-','') as INT),
    Date         = CONVERT(date, d),
	--DateKey		   = CAST(REPLACE(CONVERT(varchar(10), d),'-','') as INT),
    DayofMonth   = DATEPART(DAY,       d),
    DayName      = DATENAME(WEEKDAY,   d),
    WeekOfYear    = DATEPART(WEEK,      d),
    ISOWeek      = DATEPART(ISO_WEEK,  d),
    DayOfWeek    = DATEPART(WEEKDAY,   d),
    Month        = DATEPART(MONTH,     d),
    MonthName    = DATENAME(MONTH,     d),
	MonthAbbrev  = LEFT(DATENAME(MONTH, d),3),
    Quarter      = DATEPART(Quarter,   d),
	Qtr          =(CASE
						WHEN DATEPART(Quarter,   d) = 1 THEN 'Q1'
						WHEN DATEPART(Quarter,   d) = 2 THEN 'Q2'
						WHEN DATEPART(Quarter,   d) = 3 THEN 'Q3'
						WHEN DATEPART(Quarter,   d) = 4 THEN 'Q4'
						ELSE 'Err'
					END
					),
    Year         = DATEPART(YEAR,      d),
    FirstOfMonth = DATEFROMPARTS(YEAR(d), MONTH(d), 1),
    LastOfYear   = DATEFROMPARTS(YEAR(d), 12, 31),
    DayOfYear    = DATEPART(DAYOFYEAR, d)
  FROM d
)
SELECT * 
--INTO dim.Calendar
FROM src
  ORDER BY Date
  OPTION (MAXRECURSION 0);

  ```
  ### CLEAN SALARIES DATA WITH SQL

  ```

  
--USE dbo.ds_Salaries
---DICTIONARY AND INFO

--data source --https://www.kaggle.com/datasets/ruchi798/data-science-job-salaries

------------------------------------------------------------------------------------------------

--BUSINESS/RESERCH QUESTION

--1. An Insight into the highest payin tech roles by Regions
--2. An Insight into the highest top 5 paying tech roles by Countries ( US, UK, CA)
--3. An Insight into the Highest paying top 5 tech toles in Canada
--4. An Insight into the trend of work from home and remote work since 2020 covid by countries and region


--https://github.com/lukes/ISO-3166-Countries-with-Regional-Codes/blob/master/all/all.csv
--Country Code  ISO 3166 country code.

--Step 1 Clen data, improved on City and added the full country names, employment type and company size

--Stel 1b --Identified the Dim tables required to answer the Business Questiona nd Sub Questions

 --STEP 2 Identified Gains, Created Key and Dim tables

 --Ste 3 created Highrachy in DIM table with Country as Grain



 --Final Stages
 --Created Views to answer the questions

 ----Highest Salaries overall, by Country (USA and Canada) and by Region


--Challanges faced: has to convaty NULL data to int because the data was inported as Text fro csv


---------------------------------------------------------------------------------------------------------------------------------

USE Salaries

DROP TABLE IF EXISTS stg.fact
DROP TABLE IF EXISTS dim.JobTitle
DROP TABLE IF EXISTS dim.EmploymentType
DROP TABLE IF EXISTS dim.ExperienceLevel
DROP TABLE IF EXISTS dim.CompanySize
DROP TABLE IF EXISTS dim.CompLocationCountry
DROP TABLE IF EXISTS dim.SalaryInUSD
DROP TABLE IF EXISTS dim.RemoteWorkRatio
DROP TABLE IF EXISTS dim.RemoteWork



---Fact Table
select --[Column 0] as 'ID'
		 ROW_NUMBER() OVER(ORDER BY work_year) + 1000 as 'JobTitleID'
		,ROW_NUMBER() OVER(ORDER BY work_year) + 2000 as 'EmpTypeID'
		,ROW_NUMBER() OVER(ORDER BY work_year) + 3000 as 'ExpLevelID'
	    ,ROW_NUMBER() OVER(ORDER BY work_year) + 4000 as 'CompSizeID' ---
		,ROW_NUMBER() OVER(ORDER BY work_year) + 5000 as 'CompLocationID'
		,ROW_NUMBER() OVER(ORDER BY work_year) + 6000 as 'SalUSDID'
		,ROW_NUMBER() OVER(ORDER BY work_year) + 7000 as 'RmtWrkRatioID' ---
		,ROW_NUMBER() OVER(ORDER BY work_year ) + 8000 as 'RmtWrkID'
		,employee_residence as 'EmpResidenceCountryCode'
		,remote_ratio as'RemoteWork'
		,work_year as 'WorkYear'
		INTO stg.fact
		from dbo.ds_salaries
		--ORDER BY ID

GO
--------------------------------------------------------------------------------------------------------------

--RemoteWorkRatio DIM
Select ROW_NUMBER() OVER(ORDER BY sal.work_year) +7000 as 'RmtWrkRatioID'
,sal.work_year 
,avg(cast (sal.remote_ratio as int))  as 'RemoteWorkRatio'--HANDLING NULL
INTO dim.RemoteWorkRatio
from dbo.ds_salaries sal
group by sal.work_year 

GO

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
--RemoteWork DIM
Select ROW_NUMBER() OVER(ORDER BY remote_ratio ) + 8000 as 'RmtWrkID'
   ,remote_ratio 
  ,(CASE
		WHEN  remote_ratio < 49 THEN 'OnSiteWork'
		WHEN  remote_ratio = 50 THEN 'Hybrid'
		WHEN  remote_ratio > 50 THEN 'FullyRemote'
END) as 'RemoteWork'
 INTO dim.RemoteWork
from dbo.ds_salaries

----------------------------------------------------------------------------------------------------------------------
----EmploymentType  Dim
GO

	SELECT ROW_NUMBER() OVER(ORDER BY work_year) + 2000 as 'EmpTypeID'
			,(CASE
			WHEN employment_type = 'FT' THEN 'FullTime'
			WHEN employment_type = 'PT' THEN 'PartTime'
			WHEN employment_type ='CT' THEN 'Contract'
			WHEN employment_type = 'FL' THEN 'Freelance'
		END)   as 'EmploymentType'
		INTO dim.EmploymentType
		FROM dbo.ds_salaries

----------------------------------------------------------------------------------------------------------------------------
GO
--Job JobTitle DIM	
	Select ROW_NUMBER() OVER(ORDER BY work_year) + 1000 as 'JobTitleID'
		,job_Title as 'JobTitle'
	INTO dim.JobTitle
		FROM dbo.ds_salaries

--------------------------------------------------------------------------------------------------------------------------------

--Job Experience Level DIM
GO

Select ROW_NUMBER() OVER(ORDER BY work_year) + 3000 as 'ExpLevelID'
			,(CASE
			WHEN experience_level  = 'EN' THEN 'EntryLevel'
			WHEN experience_level  = 'EX' THEN 'Executive'
			WHEN experience_level  ='MI' THEN 'MidLevel'
			WHEN experience_level = 'SE' THEN 'Senior'
		END)   as 'ExperienceLevel'
	INTO dim.ExperienceLevel
		FROM dbo.ds_salaries

-----------------------------------------------------------------------------------------------------------------------------
GO
---CompLocationCountry DIM

Select ROW_NUMBER() OVER(ORDER BY work_year) + 4000 as 'CompSizeID'
			,(CASE
			WHEN company_size = 'L' THEN 'Large'
			WHEN company_size  = 'M' THEN 'Medium'
			WHEN company_size  ='S' THEN 'Small'		
		END)   as 'CompanySize'
	INTO dim.CompanySize
		FROM dbo.ds_salaries

------------------------------------------------------------------------------------------------------------------------------------
--Company Location DIM
  GO
  
 WITH CTE as 
 (
 Select ROW_NUMBER() OVER(ORDER BY work_year) + 5000 as 'CompLocationID'
		,company_location AS 'CompanyLocationCode'
		,(CASE
			WHEN company_location = 'AE' THEN 'United Arab Emirates'
			WHEN company_location  = 'AS' THEN 'American Samoa'
			WHEN company_location  ='AT' THEN 'Austria'
			
			WHEN company_location = 'AU' THEN 'Australia'
			WHEN company_location  = 'BE' THEN 'Belgium'
			WHEN company_location  ='BR' THEN 'Brazil'

			WHEN company_location = 'CA' THEN 'Canada'
			WHEN company_location  = 'CH' THEN 'Switzerland'
			WHEN company_location  ='CL' THEN 'Chile'

			WHEN company_location = 'CO' THEN 'Colombia'
			WHEN company_location  = 'CZ' THEN 'CzechRepublic'
			WHEN company_location  = 'CN' THEN 'China'
			WHEN company_location  ='DE' THEN 'Germany'

			WHEN company_location = 'DK' THEN 'Denmark'
			WHEN company_location  = 'DZ' THEN 'Algeria'
			WHEN company_location  ='EE' THEN 'Estonia'

			WHEN company_location = 'ES' THEN 'Spain'
			WHEN company_location  = 'FR' THEN 'France'
			WHEN company_location  ='GB' THEN 'Great Britain'

			WHEN company_location = 'GR' THEN 'Greece'
			WHEN company_location  = 'HN' THEN 'Honduras'
			WHEN company_location  ='HR' THEN 'Croatia'

			WHEN company_location = 'HU' THEN 'Hungary'
			WHEN company_location  = 'IE' THEN 'Ireland'
			WHEN company_location  ='IL' THEN 'Isreal'

			WHEN company_location = 'IN' THEN 'India'
			WHEN company_location  = 'IQ' THEN 'Iraq'
			WHEN company_location  ='IT' THEN 'Italy'

			WHEN company_location = 'JP' THEN 'Japan'
			WHEN company_location  = 'KE' THEN 'Kenya'
			WHEN company_location  ='MY' THEN 'Malaysia'

			WHEN company_location = 'LU' THEN 'Luxembourg'
			WHEN company_location  = 'MD' THEN 'Moldova'
			WHEN company_location  ='MT' THEN 'Malta'

			WHEN company_location = 'MX' THEN 'Mexico'
			WHEN company_location  = 'NG' THEN 'Nigeria'
			WHEN company_location  ='NL' THEN 'The Netherlands'

			WHEN company_location = 'NZ' THEN 'New Zealand'
			WHEN company_location  = 'PK' THEN 'Pakistan'
			WHEN company_location  ='PL' THEN 'Poland'

			WHEN company_location = 'PT' THEN 'Portugal'
			WHEN company_location  = 'RO' THEN 'Romania'
			WHEN company_location  ='RU' THEN 'The Russian Federation'

			WHEN company_location = 'UA' THEN 'Ukraine'
			WHEN company_location  = 'US' THEN 'United States of America'
			WHEN company_location  ='VN' THEN 'VietNam'

			WHEN company_location = 'SG' THEN 'Singapore'
			WHEN company_location  = 'SI' THEN 'Slovenia'
			WHEN company_location  ='TR' THEN 'Türkiye'
			WHEN company_location  ='IR' THEN 'Iran'
		END)   as 'CompLocationCountry'
		--INTO dim.CompLocationCountry
		FROM dbo.ds_salaries
	)
Select 
		*
		,(CASE 

			WHEN  compLocationCountry in ('Germany','Austria','Belgium','Switzerland','CzechRepublic','Denmark','Estonia','Spain','France','Great Britain',
			'Greece','Croatia','Hungary','Ireland','Italy','Luxembourg','Moldova','Malta','The Netherlands','Poland','Portugal','Romania','The Russian Federation'
			,'Ukraine','Slovenia') THEN 'Europe'

			WHEN  compLocationCountry in ('United Arab Emirates','China','Isreal','India','Iraq','Japan','Malaysia','Pakistan','VietNam','Singapore','Türkiye','Iran') THEN 'Asia'
			WHEN  compLocationCountry in ('American Samoa','Australia','New Zealand') THEN 'Oceania'
			WHEN  compLocationCountry in ('Brazil','Canada','Chile','Colombia','Honduras','Mexico','United States of America') THEN 'Americas'
			WHEN  compLocationCountry in ('Algeria','Kenya','Nigeria') THEN 'Africa'
			ELSE 'CHECK'
	end ) as 'Region'
 INTO dim.CompLocationCountry
	from CTE
-------------------------------------------------------------------------------------------------------------------------------------
GO
--Salary in USD DIM
	Select ROW_NUMBER() OVER(ORDER BY work_year) + 6000 as 'SalUSDID'
		  ,cast(salary_in_usd as int) as 'SalaryinUSD'
		  INTO dim.SalaryInUSD
		  FROM dbo.ds_salaries

GO
--------------------------------------------------------------------------------------------------------------------------------------
--**************CREATETION OF VIEW STARTS HERE****************
-----------------------------------------------------------------------------------------------------------------------------------


GO

CREATE OR ALTER VIEW vw.HighestPayingJobs 
AS
Select jb.JobTitle
		,avg(sal.SalaryInUSD)  as 'AverageSalaryinUSD'
from  stg.fact ft 
join dim.JobTitle  jb
ON ft.jobTitleID = Jb.jobTitleID
join dim.SalaryInUSD sal
on ft.SalUSDID  = sal.SalUSDID
group by jb.JobTitle






-----------------------------------------------------------------------------------------------------------------
---HIGHEST PAYING TECH JOBS IN USA

GO
CREATE OR ALTER VIEW vw.HighestPayinginUSA

AS

Select   jb.JobTitle as 'JobTitle'
		,avg(sal.SalaryInUSD)  as 'AverageSalaryinUSD'
		,comp.CompLocationCountry as 'Country'
	
from  stg.fact ft 
join dim.JobTitle  jb
ON ft.jobTitleID = Jb.jobTitleID
join dim.SalaryInUSD sal
on ft.SalUSDID  = sal.SalUSDID
join dim.CompLocationCountry comp
 on ft.compLocationID = comp.compLocationID
 where comp.CompLocationCountry = ('United States of America')
--comp.Region
group by jb.JobTitle,comp.CompLocationCountry



-------------------------------------------------------------------------------------------------------------------------

---HIGHEST PAYING TECH JOBS IN USA

GO
CREATE OR ALTER VIEW vw.HighestPayinginCanada

AS

Select   jb.JobTitle as 'JobTitle'
		,avg(sal.SalaryInUSD)  as 'AverageSalaryinUSD'
		,comp.CompLocationCountry as 'Country'
	
from  stg.fact ft 
join dim.JobTitle  jb
ON ft.jobTitleID = Jb.jobTitleID
join dim.SalaryInUSD sal
on ft.SalUSDID  = sal.SalUSDID
join dim.CompLocationCountry comp
 on ft.compLocationID = comp.compLocationID
 where comp.CompLocationCountry = ('Canada')

group by jb.JobTitle,comp.CompLocationCountry

GO

-------------------------------------------------------------------------------------------------------------------------
CREATE OR ALTER VIEW vw.HighestPayingbyRegion
AS
Select   jb.JobTitle as 'JobTitle'
		,avg(sal.SalaryInUSD)  as 'AverageSalaryinUSD'
		,comp.CompLocationCountry as 'Country'
		,comp.Region  as 'Region'
from  stg.fact ft 
join dim.JobTitle  jb
ON ft.jobTitleID = Jb.jobTitleID
join dim.SalaryInUSD sal
on ft.SalUSDID  = sal.SalUSDID
 join dim.CompLocationCountry comp
 on ft.compLocationID = comp.compLocationID
 where comp.Region = ('Americas')
--comp.Region
--group by jb.JobTitle,comp.CompLocationCountry
group by jb.JobTitle,comp.Region,comp.CompLocationCountry


----------------------------------------------------------------------------------------------------------------------------

GO
CREATE OR ALTER VIEW vw.fact 
AS
Select * from stg.fact

GO
CREATE OR ALTER VIEW vw.JobTitle 
AS
Select * from dim.JobTitle

GO
CREATE OR ALTER VIEW vw.EmploymentType 
AS
Select * from dim.EmploymentType

GO

CREATE OR ALTER VIEW vw.ExperienceLevel 
AS
Select * from dim.ExperienceLevel

GO
CREATE OR ALTER VIEW vw.CompanySize 
AS
Select * from dim.CompanySize

GO
CREATE OR ALTER VIEW vw.CompLocationCountry 
AS
Select * from dim.CompLocationCountry


GO

CREATE OR ALTER VIEW vw.SalaryInUSD
AS
Select * from dim.SalaryInUSD

GO
CREATE OR ALTER VIEW vw.RemoteWorkRatio
AS
Select * from dim.RemoteWorkRatio


GO

CREATE OR ALTER VIEW vw.RemoteWork
AS
Select * from dim.RemoteWork

```#   S Q L - 2  
 