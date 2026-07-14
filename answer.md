i think i will use the layer architecture for this API
have 3 folder controller, service, repository

the controller will have the file BookController have all the endpoint that can use like

add POST ("api/v1/books", body = {...})
view GET ("api/v1/books/{id})
update PATCH ("api/v1/books/{id})
delete DELETE ("api/v1/books/{id})

the service folder have the BookService that it will handle the bussiness logic like "how we can add the book, get each field in a book or all field, update each field , when delete just need to update the status or delete from the DB"

and the final repository folder have the BookRepository where can interact with the DB we can save, find, update, delete with the DB

the schema it should look like this

Book {
    UUID id;
    String name
    String category;
    String authorName;
    String publisher;
    Instant createdAt;
    Instant updatedAt;
}