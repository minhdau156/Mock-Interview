i think first i will create the VendingManager class

which will have 2 DI

FormulaManager for managing the coin get in the machine
DrinkFactory for dispense the drink that customer choose

and the Drink interface which have 3 field 
code, price, name
and the class Drink that implemnent the Drink class
like Pepsi, Coca, Tea, Coffee

when you get the coin in and type the code of drink the FormulaManager will handle whether you have enough fund or insufficient fund it will throw the error and the Vending Manager will handle the error

so if you have enough we will go to the DrinkFactory to dispense the drink and it will match the suitable code if it have give it to customer else out of stock or no code matching throw exception so VendingManager will handle it