# Flight Delays

Flight delays are coded for company performance evaluation and trend analysis. This project looked at what caused flight delays and how to recover them for better on-time performance.

The focus of the exploration was on the type of delay and in which airports they were occuring. By reorganizing the flight records, I was able to determine that many delays were not coded or were given a generic code of "late in, late out", despite additional delay time being added.

Because I was using flat files of historical records, I was able use some DDL in a copy of the data to better organize everything. The records are chronological, but I realized I needed to order it differently to see how the delays were occuring. There was also a lot of noise in the data that needed to be cleaned out.

Once that was done, it was clear the two biggest issues were the gap in the data and the catchall code.

I then worked with the department heads of the airport staff and IT to find the loopholes in the process where this was occurring.

The recommendation to senior leadership was to close technical loopholes and modify how delays were tracked for better future evaluation. This led to a 73% reduction in delay time and 47% fewer delays.

```

SQL Code for Delays Project

-- Organize FLIFO Output by Aircraft Day --
SELECT * FROM [dbo].[flifoData_2017_2023_DELAYED_MOD]
	ORDER BY [Date], [Aircraft], [Out_Time]


	-- Remove Spare Pucks and SSR Keywords--
	DELETE FROM [dbo].[flifoData_2017_2023_DELAYED_MOD]
		WHERE [SSR] LIKE '%SPARE START%'
			OR [SSR] LIKE '%SPARE END%'
			OR [SSR] LIKE '%SPERE%'
			OR [SSR] LIKE '%TRAIN%'
			OR [SSR] LIKE '%CHECKRIDE%'
			OR [SSR] LIKE '%CHECK RIDE%'
			OR [SSR] LIKE '%GROUNDSCHOOL%'
			OR [SSR] LIKE '%GROUND SCHOOL%'
			OR [SSR] LIKE '%CURRENCY%'
			OR [SSR] LIKE '%FCF%'
			OR [SSR] LIKE '%SFP%'
			OR [SSR] LIKE '%FERRY%'
			OR [SSR] LIKE '%TRAVEL%'
			OR [SSR] LIKE '%TRANSITION%'

	-- Remove CXL Flights --
	DELETE FROM [dbo].[flifoData_2017_2023_DELAYED_MOD]
		WHERE [Cancelled] = 1
		
	--Remove Deleted Flights --
	DELETE FROM [dbo].[flifoData_2017_2023_DELAYED_MOD]
		WHERE [Deleted] = 1

	-- Remove Training Flights, Repos, Pucks (Preserves Mail and Papers) --
	DELETE FROM [dbo].[flifoData_2017_2023_DELAYED_MOD]
		WHERE [Flight] BETWEEN 6200 AND 6249
			OR [Flight] BETWEEN 6260 AND 6263
			OR [Flight] BETWEEN 6270 AND 6999

	-- Remove Flights That Never Happened --
	DELETE FROM [dbo].[flifoData_2017_2023_DELAYED_MOD]
		WHERE [Out_Time] is NULL




	-- Add Departure Delayed Minutes and D00 and D14 Boolean Columns --
	ALTER TABLE [dbo].[flifoData_2017_2023_DELAYED_MOD] ADD Dep_Delayed_Time AS DATEDIFF(MINUTE, [Sched_Departure], [Out_Time])
	ALTER TABLE [dbo].[flifoData_2017_2023_DELAYED_MOD] ADD D00 BIT
	ALTER TABLE [dbo].[flifoData_2017_2023_DELAYED_MOD] ADD D14 BIT
	
	-- Add Arrival Delayed Minutes and A00 and A14 Boolean Columns --
	ALTER TABLE [dbo].[flifoData_2017_2023_DELAYED_MOD] ADD Arr_Delayed_Time AS DATEDIFF(MINUTE, [Sched_Arrival], [In_Time])
	ALTER TABLE [dbo].[flifoData_2017_2023_DELAYED_MOD] ADD A00 BIT	
	ALTER TABLE [dbo].[flifoData_2017_2023_DELAYED_MOD] ADD A14 BIT

	-- Add First Flights Column --
	ALTER TABLE [dbo].[flifoData_2017_2023_DELAYED_MOD] ADD First_Flight BIT

	-- Add LAE Code Test and Standard Trn Difference --
	ALTER TABLE [dbo].[flifoData_2017_2023_DELAYED_MOD] ADD Bad_LAE BIT
	ALTER TABLE [dbo].[flifoData_2017_2023_DELAYED_MOD] ADD Bad_LAE_Time INT

	-- Add Flight Group ID --
	ALTER TABLE [dbo].[flifoData_2017_2023_DELAYED_MOD] ADD Flight_Group NVARCHAR(50)

	-- Add LAE Original Cause --
	ALTER TABLE [dbo].[flifoData_2017_2023_DELAYED_MOD] ADD LAE_Cause NVARCHAR(50)

	-- Add Pilot Info Columns --
	ALTER TABLE [dbo].[flifoData_2017_2023_DELAYED_MOD] ADD Pathway NVARCHAR(50)
	ALTER TABLE [dbo].[flifoData_2017_2023_DELAYED_MOD] ADD Hire_Date DATETIME2
	ALTER TABLE [dbo].[flifoData_2017_2023_DELAYED_MOD] ADD Commitment_Length TINYINT
	ALTER TABLE [dbo].[flifoData_2017_2023_DELAYED_MOD] ADD Commitment_Date DATETIME2
	ALTER TABLE [dbo].[flifoData_2017_2023_DELAYED_MOD] ADD Fulfilled BIT
	ALTER TABLE [dbo].[flifoData_2017_2023_DELAYED_MOD] ADD Pilot_Status NVARCHAR(50)
	ALTER TABLE [dbo].[flifoData_2017_2023_DELAYED_MOD] ADD Eligibility NVARCHAR(50)




	-- Insert Data to Added Columns --
	UPDATE [dbo].[flifoData_2017_2023_DELAYED_MOD]
			SET [D00] = CASE WHEN [Dep_Delayed_Time] > 0 THEN 1
			ELSE 0
			END

	UPDATE [dbo].[flifoData_2017_2023_DELAYED_MOD]
			SET [D14] = CASE WHEN [Dep_Delayed_Time] > 14 THEN 1
			ELSE 0
			END

	UPDATE [dbo].[flifoData_2017_2023_DELAYED_MOD]
			SET [First_Flight] = CASE WHEN [Turn_Time] = 0 THEN 1
			ELSE 0
			END

	UPDATE [dbo].[flifoData_2017_2023_DELAYED_MOD]
			SET [A00] = CASE WHEN [Arr_Delayed_Time] > 0 THEN 1
			ELSE 0
			END
	
	UPDATE [dbo].[flifoData_2017_2023_DELAYED_MOD]
			SET [A14] = CASE WHEN [Arr_Delayed_Time] > 14 THEN 1
			ELSE 0
			END

	UPDATE [dbo].[flifoData_2017_2023_DELAYED_MOD]
			SET [Bad_LAE] = CASE WHEN
				[Delay_Code] = 'DC70' AND
				[Turn_Time] > 25
				THEN 1
				ELSE 0
				END

	UPDATE [dbo].[flifoData_2017_2023_DELAYED_MOD]
		SET [Bad_LAE_Time] = [Turn_Time] - 25
		WHERE [Bad_LAE] = 1

	UPDATE 	[dbo].[flifoData_2017_2023_DELAYED_MOD]
		SET [Flight_Group] = CONCAT(FORMAT([Date], 'yyyy-MM-dd'),[Aircraft])

	UPDATE [dbo].[flifoData_2017_2023_DELAYED_MOD]
			SET [Aircraft_Type] = CASE WHEN 
				[Aircraft] = 'N14834' OR 
				[Aircraft] = 'N42836' THEN 'ATR'
			WHEN 
				[Aircraft] = 'N510BN' OR 
				[Aircraft] = 'N520BN' OR 
				[Aircraft] = 'N530BN' OR 
				[Aircraft] = 'N540BN' THEN 'BN2'
			WHEN 
				[Aircraft] = 'N133CA' OR 
				[Aircraft] = 'N135CA' OR 
				[Aircraft] = 'N244CA' OR 
				[Aircraft] = 'N255CA' OR 
				[Aircraft] = 'N266CA' OR 
				[Aircraft] = 'N288CA' OR 
				[Aircraft] = 'N335CA' OR 
				[Aircraft] = 'N345CA' OR 
				[Aircraft] = 'N357CA' OR 
				[Aircraft] = 'N396CA' OR 
				[Aircraft] = 'N411CA' OR 
				[Aircraft] = 'N443CA' OR 
				[Aircraft] = 'N508CA' OR
				[Aircraft] = 'N566CA' OR 
				[Aircraft] = 'N651CA' OR 
				[Aircraft] = 'N699CA' OR
				[Aircraft] = 'N722CA' OR 
				[Aircraft] = 'N743CA' OR 
				[Aircraft] = 'N785CA' OR 
				[Aircraft] = 'N833CA' OR 
				[Aircraft] = 'N843CA' OR 
				[Aircraft] = 'N915CA' OR 
				[Aircraft] = 'N929CA' OR
				[Aircraft] = 'N945CA' OR
				[Aircraft] = 'N949CA' OR
				[Aircraft] = 'N965CA' OR 
				[Aircraft] = 'N969CA' OR 
				[Aircraft] = 'N979CA' OR 
				[Aircraft] = 'N989CA' OR 
				[Aircraft] = 'N995CA' THEN 'P2012'
			WHEN
				[Aircraft] IS NULL THEN NULL
			WHEN
				[Aircraft] = 'N' THEN NULL

				ELSE 'C402'
			END
	
	UPDATE [dbo].[flifoData_2017_2023_DELAYED_MOD]
		SET [dbo].[flifoData_2017_2023_DELAYED_MOD].[Pathway] = [dbo].[Pilot_Data].[Pathway_Code],
			[dbo].[flifoData_2017_2023_DELAYED_MOD].[Hire_Date] = [dbo].[Pilot_Data].[Date_Of_Hire],
			[dbo].[flifoData_2017_2023_DELAYED_MOD].[Commitment_Months] = [dbo].[Pilot_Data].[Commitment_Length],
			[dbo].[flifoData_2017_2023_DELAYED_MOD].[Commitment_Date] = [dbo].[Pilot_Data].[End_Of_Commitment_Date],
			[dbo].[flifoData_2017_2023_DELAYED_MOD].[Fulfilled] = [dbo].[Pilot_Data].[Fulfilled_Commitment],
			[dbo].[flifoData_2017_2023_DELAYED_MOD].[Pilot_Status] = [dbo].[Pilot_Data].[Status_Code],
			[dbo].[flifoData_2017_2023_DELAYED_MOD].[Eligibility] = [dbo].[Pilot_Data].[Rehire_Eligibility_Code]
		FROM [dbo].[Pilot_Data], [dbo].[flifoData_2017_2023_DELAYED_MOD]
	WHERE [dbo].[flifoData_2017_2023_DELAYED_MOD].[Pilot] = [dbo].[Pilot_Data].[Flifoid]


	-- Add Incentive and Cancellation Info --
	select Bidding.[Date],Bidding.Pilots, Bidding.Num_Lines_on_Bids, Incentive.DATE_OF_ADD, Incentive.Date_of_shift, Incentive.AVG_cost_per_shift, cxls.FLTS_CXLD, cxls.CTRL_CXLD
		from [myDatabase].[dbo].[Pilot_And_Bidding_Numbers] Bidding
	full outer join [myDatabase].[dbo].[Incentive 2014-2023] Incentive
		on Bidding.[Date] = Incentive.[DATE]
	full outer join [myDatabase].[dbo].[CXLD_FLTS_2018_2022] CXLs
		on CXLs.Date = Incentive.Date
	where incentive.date between '01/01/2015' and '2/28/2023'
  
  }
  ```

![](https://github.com/sfisher2277/Delays/blob/main/images/Delay%20Time.JPG)

![](https://github.com/sfisher2277/Delays/blob/main/images/Delays%20by%20Code.jpg)

![](https://github.com/sfisher2277/Delays/blob/main/images/Delays%20Pie.JPG)

![](https://github.com/sfisher2277/Delays/blob/main/images/Length%20Delays.JPG)




