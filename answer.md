i think when get the thread it will cost the memory,
the thread for the current request and the thread for send confirmationso if have 5000 request in few minute it can have 10000 thread so if each thread can not handle it, it also create and the memory have full space and having out of memory error so 

in this one i can use MessegeQueue for the send confirmation by email so that we just need create the MQ and push to it we don't need to create the new thread 