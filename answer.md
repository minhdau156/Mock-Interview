no it not doing the same thing

the first is stateful and the second is stateless 

so when we use two way to handle the login it will conflict

because the JWT will attach to header and session id is also but when the request go to my app i need to handle both of them , validate the session_id and jwt so it will make the app more complex and hard
in my opinion i will pick JWT for both because the jwt is stateless it don't save any info about the user like using session in statefull, it just need validate the JWT using the secret so when valid it just pass through ortherwise the statefull will make our memory in server is higher