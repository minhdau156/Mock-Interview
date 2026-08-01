it the N + 1 query problem 
when i call to db to get the orders i want to get all orders but when i use this query it vill relevant with the User and it can call the query to select the user having this customer_id

so if i the orders have 10000 record it will call after that 10000 query just to get the order's customer name so we just need 1 query but it took 10001 query so it will down our db , our server if CCU is higher 

so if we want to fix it we need to use each approach

JOIN FETCH in Spring data jpa 

we join 2 table order and user together i select the order and customer name at once (big query)
