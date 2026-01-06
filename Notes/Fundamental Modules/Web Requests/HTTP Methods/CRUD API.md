# CRUD API

In previous sections, we worked with a City Search web application using PHP parameters. In this section, we examine how the same functionality can be implemented using an **API**, and how we can directly interact with its endpoints.

---

## APIs

APIs are commonly used to interact with databases. They allow clients to specify:

* The database table
* The specific record (row)
* The operation to perform, using an HTTP method

For example, to update a city named `london` using an API endpoint:

```bash
curl -X PUT http://<SERVER_IP>:<PORT>/api.php/city/london
```

---

## CRUD Operations

APIs typically support four core operations on database entities. These are collectively referred to as **CRUD**:

| Operation | HTTP Method | Description                      |
| --------- | ----------- | -------------------------------- |
| Create    | POST        | Adds new data to the database    |
| Read      | GET         | Retrieves data from the database |
| Update    | PUT         | Modifies existing data           |
| Delete    | DELETE      | Removes data from the database   |

This pattern is also widely used in **REST APIs**. Access control determines which operations a user is allowed to perform.

---

## Read (GET)

Reading data is usually the first operation performed against an API.

### Reading a Specific Entry

```bash
curl http://<SERVER_IP>:<PORT>/api.php/city/london
```

Response:

```json
[{"city_name":"London","country_name":"(UK)"}]
```

To format the output cleanly, pipe it into `jq`:

```bash
curl -s http://<SERVER_IP>:<PORT>/api.php/city/london | jq
```

### Partial Search

```bash
curl -s http://<SERVER_IP>:<PORT>/api.php/city/le | jq
```

Returns all cities matching the search term.

### Reading All Entries

```bash
curl -s http://<SERVER_IP>:<PORT>/api.php/city/ | jq
```

This retrieves all rows in the table.

---

## Create (POST)

To add a new entry, use **POST** with JSON data. Since the API expects JSON, the `Content-Type` header must be set accordingly.

```bash
curl -X POST \
  http://<SERVER_IP>:<PORT>/api.php/city/ \
  -d '{"city_name":"HTB_City", "country_name":"HTB"}' \
  -H 'Content-Type: application/json'
```

### Verifying Creation

```bash
curl -s http://<SERVER_IP>:<PORT>/api.php/city/HTB_City | jq
```

Response:

```json
[
  {
    "city_name": "HTB_City",
    "country_name": "HTB"
  }
]
```

---

## Update (PUT)

To modify an existing entry, use **PUT**. The entity being updated must be specified in the URL.

> **Note**
> Some APIs use `PATCH` for partial updates and `PUT` for full replacements. In this example, PUT replaces the entire record.

### Updating an Entry

```bash
curl -X PUT \
  http://<SERVER_IP>:<PORT>/api.php/city/london \
  -d '{"city_name":"New_HTB_City", "country_name":"HTB"}' \
  -H 'Content-Type: application/json'
```

### Confirming the Update

```bash
curl -s http://<SERVER_IP>:<PORT>/api.php/city/london | jq
```

```bash
curl -s http://<SERVER_IP>:<PORT>/api.php/city/New_HTB_City | jq
```

Response:

```json
[
  {
    "city_name": "New_HTB_City",
    "country_name": "HTB"
  }
]
```

The original entry no longer exists, and the updated entry has replaced it.

> **Important**
> Some APIs allow PUT to create new entries if they do not already exist. This behavior depends on the API implementation.

---

## Delete (DELETE)

Deleting an entry is straightforward. Specify the entity and use the **DELETE** method.

```bash
curl -X DELETE http://<SERVER_IP>:<PORT>/api.php/city/New_HTB_City
```

### Verifying Deletion

```bash
curl -s http://<SERVER_IP>:<PORT>/api.php/city/New_HTB_City | jq
```

Response:

```json
[]
```

An empty array confirms that the entry has been removed.

---

## Summary

Using cURL, we successfully performed all four CRUD operations:

* Read data using GET
* Create data using POST
* Update data using PUT
* Delete data using DELETE

In real-world applications, unrestricted access to these operations would be a serious security vulnerability. Proper authentication and authorization are essential.

To authenticate API requests, applications typically require:

* Session cookies
* Authorization headers (e.g. Bearer tokens or JWTs)

Once authenticated, API interaction follows the same principles demonstrated in this section.
