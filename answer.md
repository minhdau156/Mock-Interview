i think i will make the relation between 2 table become n - n so when we add the field category_id_2 it will make our db more bigger

so we need to create the table that it will have 2 primary key of product and category 

it like this

product_id reference products(id)
category_id reference categories(id);

so it can make our table clean