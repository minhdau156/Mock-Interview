it occurr because it have the race condition, when 2 thread using same resource it is count variable
we just see it count++;
but actually the computer do that
get the value in count 
add 1 to this value
set the value to the count again

so the race condition can occurr like this 
when having 2 or more thread 
the counter now have the value is 10 just for example
A thread is add 1 to the current value in count the value is 11; but the count just 10;
so the B thread is get value in count, add 1 to the value and set the value before the thread A set the value again to count

so we can see 2 thread a set the count is 11; but it must be 12,

so if we want to fix it we need to use some lock , the easily approach is use MUTEX (mutual exclusive); it will lock the thread until this lock is done is will release and the next thread cotinues proccessing 

in the above example we can see the thread B processed between the proccessing of thread A so it can make the count have the wrong value
so when use the mutex lock 
the thread B must wait util the thread A proccess done, so that the resource (count) can not have the wrong value
