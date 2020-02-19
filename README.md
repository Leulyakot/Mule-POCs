# MongoDB Mule 4 Sample API

## Built With

* [MongoDB Connector v5.3](https://www.mulesoft.com/exchange/com.mulesoft.connectors/mule-mongodb-connector/) - connector
* [Mule 4.2.2]- Runtime version

## Prerequisites
* Anypoint Studio(7 or higher)
* MongoDB instance
* Set of valid credentials, including the required MongoDB endpoints, pointing to your instance

## Running the tests using postman

you can use postman or any other client to send a request to the API

### To Find one document form MongoDB 

```
(db.COLLECTION_NAME.find(document))
```

GET call to find one document from mongodb ("Fail on not found" option can be set to [TRUE] to throw an exception)

```
http://localhost:8081/api/inventory/find?item=whiteboard
```

### To Insert Documents 

```
(db.COLLECTION_NAME.insert(document))
```

POST call to insert a document

```
{ 
"item": "tablet", 
"qty": 70, 
"tags": ["electronics"], 
"size": { "h": 10, "w": 10, "uom": "cm" } 
}
```



