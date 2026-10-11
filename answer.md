after the hint i understand that
1. the list use this method need have the id sort in ascending order because the condition v < id it will work properly if the id is sorted and this is the requirement in binary serach

the method is just O(log(N)) but we need to sort the Order asc base on it's id so that we need O(nlog(n)) compare to loop 

in this case the loop will work better because it just need O(n)

when i need to use HashMap i think when the method need to cache the result so that if we have many search each will take O(n) so instead of this we can use a Map to cache the result and return without searching