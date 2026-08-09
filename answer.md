because we use the the composite index so the first condition 
'customer_id = 42' is satifie so it will have the Index Scan but when goes to the second condition it is 'YEAR(created_at) = 2026' it sort the data  of created_at is TIMESTAMP so when use the YEAR() function is like the char so it can not match and it need to full scan the table to find some column that YEAR(created_at) = 2026 of customer have id is 42 in the table have 20 million row it will take much time 

so if we want to fix we should compare the created_at with TIMESTAMP

i will re-write the query like this
SELECT * FROM orders WHERE customer_id = 42 AND created_at >= '2026-01-01 00:00:00' AND created_at < '2027-01-01 00:00:00';
