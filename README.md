# SCD2-4-MySQL
```
-- drop table Customer;
create table Customer (
	-- tbl_id int AUTO_INCREMENT,
    	cust_id int not null AUTO_INCREMENT primary key,
    	fullname varchar(50) not null,
    	position varchar(30) not null,
    	start_date date not null
);

-- drop table Customer_History;
create table Customer_History (
	hist_id int AUTO_INCREMENT primary key,
    	cust_id int not null,
    	fullname varchar(50),
    	position varchar(30),
    	eff_start_date date,
    	eff_end_date date,
    	curr_flag char(1),
    	change_flag varchar(10),
    	change_version_id int
);

insert into Customer (fullname, position, start_date)
values
	("Dan Wilson", "Developer",  '2019-02-01'),
    	("Tara Karlston", "Junior Developer",  '2020-02-01'),
    	("Brenda Fulston", "CRM Developer", "2020-03-01"),
	("MIles Fulston", "CRM Integrator", "2020-04-01");
  
########################################################
########################################################

-- expire the records that exist in source
UPDATE Customer_History TGT, Customer SRC
SET eff_end_date = Subdate(start_date, 1), curr_flag = 'N' -- , change_flag = 'UPDATE'
WHERE TGT.cust_id = SRC.cust_id
AND (TGT.fullname <> SRC.fullname OR TGT.position <> SRC.position)
AND eff_end_date = '9999-12-31'
AND TGT.curr_flag = 'Y';

-- invalidate records that are deleted in source
UPDATE Customer_History TGT
SET eff_end_date = current_date(), curr_flag = 'N', change_flag = 'DELETE'
WHERE TGT.cust_id IN (
	SELECT * FROM (
		SELECT CH.cust_id
		FROM Customer_History CH
		WHERE CH.cust_id not in (
			SELECT cust_id FROM Customer)
			AND curr_flag = 'Y') t)
		AND TGT.curr_flag = 'Y';

-- add a new row for the changing records
INSERT INTO Customer_History
SELECT NULL,
	SRC.cust_id,
	SRC.fullname,
	SRC.position,
	SRC.start_date,
	'9999-12-31',
	'Y',
    	'UPDATE', -- change_flag
    	change_version_id + 1
FROM Customer_History TGT, Customer SRC
WHERE TGT.cust_id = SRC.cust_id
AND (TGT.fullname <> SRC.fullname OR TGT.position <> SRC.position)
AND EXISTS(
	SELECT * FROM Customer_History CH
	WHERE SRC.cust_id = CH.cust_id
	AND TGT.eff_end_date = Subdate(start_date, 1)
	AND NOT EXISTS(
		SELECT *
		FROM Customer_History CH2
		WHERE SRC.cust_id = CH2.cust_id
		AND CH2.eff_end_date = '9999-12-31'));
        
-- add new records
INSERT INTO Customer_History
SELECT NULL,
	cust_id,
	fullname,
	position,
	start_date,
	'9999-12-31',
	'Y',
    	'INSERT',
    	1
FROM Customer
WHERE cust_id NOT IN(
	SELECT CH2.cust_id
	FROM Customer_History CH, Customer CH2
	WHERE CH.cust_id = CH2.cust_id);
```
