-- Creating EVETNTS Dimension Table
CREATE TABLE EVENTS_DIM (
    ID INT PRIMARY KEY,
    BASE_EVENT_ID INT, 
    EVENT_GENDER varchar2(1)
    -- Add additional columns as needed
);

--Populating Events Dimension table with the OLYM.OLYM_EVENTS table
--No data is manually inserted
INSERT INTO EVENTS_DIM (ID, BASE_EVENT_ID, EVENT_GENDER)
SELECT ID, BASE_EVENT_ID, EVENT_GENDER
FROM OLYM.OLYM_EVENTS;

-- Creating BASE_EVENTS Dimension Table
CREATE TABLE BASE_EVENTS_DIM (
    ID INT PRIMARY KEY,
    DISCIPLINE_ID INT, 
    EVENT varchar2(60)
);

--Populating BASE_EVENTS Dimension table with the OLYM.OLYM_BASE_EVENTS table
--No data is manually inserted
INSERT INTO BASE_EVENTS_DIM (ID,DISCIPLINE_ID, EVENT)
SELECT ID, DISCIPLINE_ID, EVENT
FROM OLYM.OLYM_BASE_EVENTS;


-- Creating DISCIPLINE Dimension Table
CREATE TABLE DISCIPLINE_DIM (
    ID INT PRIMARY KEY,
    DISCIPLINE varchar2(30)
);

--Populating DISCIPLINE Dimension table with the OLYM.OLYM_DISCIPLINES table
--No data is manually inserted
INSERT INTO DISCIPLINE_DIM (ID, DISCIPLINE)
SELECT ID, DISCIPLINE
FROM OLYM.OLYM_DISCIPLINES;



--Creating Fact table that contains that contains 2 FK as PK and 4 measures(Bronze,Silver,Gold,Total medal numbers per Event)
CREATE TABLE OLYM_MEDALS_FACT (
    Event_ID INT,
    Base_Event_ID INT,
    Bronze_Medals INT,
    Silver_Medals INT,
    Gold_Medals INT,
    Total_Medals INT,
    FOREIGN KEY (Event_ID) REFERENCES EVENTS_DIM(ID),
    FOREIGN KEY (Base_Event_ID) REFERENCES BASE_EVENTS_DIM(ID)
);

--Inserting Event_ID into my fact table
INSERT INTO OLYM_MEDALS_FACT (Event_ID)
SELECT ID
FROM EVENTS_DIM;

--Inserting Bronze_Medal number into my fact table using OLYM.OLYM_MEDALS 
UPDATE OLYM_MEDALS_FACT MF
SET MF.Bronze_Medals = (
    SELECT COUNT(M.ID)
    FROM OLYM.OLYM_MEDALS M
    WHERE M.MEDAL = 'Bronze' AND M.EVENT_ID = MF.Event_ID
);

--Inserting Silver_Medal number into my fact table using OLYM.OLYM_MEDALS 
UPDATE OLYM_MEDALS_FACT MF
SET MF.Silver_Medals = (
    SELECT COUNT(M.ID)
    FROM OLYM.OLYM_MEDALS M
    WHERE M.MEDAL = 'Silver' AND M.EVENT_ID = MF.Event_ID
);

--Inserting Gold_Medal number into my fact table using OLYM.OLYM_MEDALS 
UPDATE OLYM_MEDALS_FACT MF
SET MF.Gold_Medals = (
    SELECT COUNT(M.ID)
    FROM OLYM.OLYM_MEDALS M
    WHERE M.MEDAL = 'Gold' AND M.EVENT_ID = MF.Event_ID
);

--Inserting Total_Medal number into my fact table using OLYM.OLYM_MEDALS and combining all medal types
UPDATE OLYM_MEDALS_FACT MF
SET MF.TOTAL_MEDALS = (
    SELECT COUNT(M.ID)
    FROM OLYM.OLYM_MEDALS M
    WHERE (M.MEDAL = 'Gold' OR M.MEDAL = 'Silver' OR M.MEDAL = 'Bronze') AND  M.EVENT_ID = MF.Event_ID
);

--Inserting Base_Event  ID into my fact table using OLYM.OLYM_EVENTS  
UPDATE OLYM_MEDALS_FACT MF
SET MF.Base_Event_ID = (
    SELECT E.BASE_EVENT_ID
    FROM OLYM.OLYM_EVENTS E
    WHERE  E.ID = MF.EVENT_ID
);


--Simple Select function of the Fact table just to see if everything is working
SELECT *
FROM OLYM_MEDALS_FACT
ORDER BY Event_ID;


--CUBE function on 3 dimension EVENT, BASE_EVENT and MEDAL and showing subtotal and grand total for all the combination that there are.
--Also using CALESCE function to fill in the missing data for Genders and Medal types
SELECT
    COALESCE(E.EVENT_GENDER, 'All Genders') AS EVENT_GENDER,
    BA.EVENT,
    COALESCE(M.MEDAL, 'TIE') AS DOMINANT_MEDAL,
    COUNT(*) AS MedalCount
FROM OLYM.OLYM_MEDALS M
RIGHT JOIN EVENTS_DIM E ON M.Event_ID = E.ID
JOIN BASE_EVENTS_DIM BA ON E.BASE_EVENT_ID = BA.ID
GROUP BY CUBE (E.EVENT_GENDER, BA.EVENT, M.MEDAL);



--The ROLLUP option generates subtotals and grand totals for different levels of aggregation
--SO by using it I am showing the gold_medal_ratio for each gender
--Filling missing gender with coalesce
SELECT
    COALESCE(E.EVENT_GENDER, 'NON SPECIFIED') AS EVENT_GENDER,
    SUM(CASE WHEN M.MEDAL = 'Gold' THEN 1 ELSE 0 END) AS Gold_Medals,
    COUNT(*) AS Total_Medals,
    (SUM(CASE WHEN M.MEDAL = 'Gold' THEN 1 ELSE 0 END) * 1.0) / COUNT(*) AS Gold_Medal_Ratio
FROM OLYM.OLYM_MEDALS M
JOIN EVENTS_DIM E ON M.EVENT_ID = E.ID
GROUP BY ROLLUP (E.EVENT_GENDER);


--This query shows the rank of each event as well as the number of medal types collected in each event
-- RANK function with the OVER clause and the PARTITION BY BASE_EVENT_ID to calculate the rank by base event based on the TOTAL_MEDALS in descending order.
SELECT
    E.EVENT_GENDER,
    BA.EVENT,
    M.Total_Medals,
    SUM(M.GOLD_MEDALS) OVER (PARTITION BY E.ID) AS Gold_Medals_By_Event,
    SUM(M.SILVER_MEDALS) OVER (PARTITION BY E.ID) AS Silver_Medals_By_Event,
    SUM(M.BRONZE_MEDALS) OVER (PARTITION BY E.ID) AS Bronze_Medals_By_Event,
    RANK() OVER (PARTITION BY E.ID ORDER BY M.Total_Medals DESC) AS Rank_By_Event
FROM OLYM_MEDALS_FACT M
JOIN EVENTS_DIM E ON M.Event_ID = E.ID
JOIN BASE_EVENTS_DIM BA ON M.BASE_EVENT_ID = BA.ID;


--Query Number 1
--Query for showing the Base_event and gender of that event as well as the Discipline of that event
SELECT
    BE.EVENT AS Base_Event,
    E.EVENT_GENDER,
    D.DISCIPLINE
FROM BASE_EVENTS_DIM BE
JOIN EVENTS_DIM E ON BE.ID = E.BASE_EVENT_ID
JOIN DISCIPLINE_DIM D ON BE.DISCIPLINE_ID = D.ID;

--Query Number 2
--Query for showing all the gender spesific base_events with discipline where number of gold medals obtained were above 10
--Above 10 gold medals considered above average so that is why I decided to write this query
SELECT
    BE.EVENT AS Base_Event,
    E.EVENT_GENDER,
    D.DISCIPLINE,
    SUM(M.GOLD_MEDALS) AS Number_Of_Gold_Medals
FROM BASE_EVENTS_DIM BE
JOIN EVENTS_DIM E ON BE.ID = E.BASE_EVENT_ID
JOIN DISCIPLINE_DIM D ON BE.DISCIPLINE_ID = D.ID
JOIN OLYM_MEDALS_FACT M ON E.ID = M.EVENT_ID
GROUP BY BE.EVENT, E.EVENT_GENDER, D.DISCIPLINE
HAVING SUM(M.GOLD_MEDALS) > 10;


