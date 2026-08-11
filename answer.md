i think first they use different JOIN

they should use the LEFT JOIN so this can get the customer who don't have any order yet and we can count it in this 

and the second one is apply the discount_code so the target they want to check all the order that include either discount or not so we need to remove it 

so the good query is like that

SELECT c.name, COUNT(o.id) AS order_count
FROM customers c
LEFT OUTER JOIN orders o ON c.id = o.customer_id
GROUP BY c.name;
