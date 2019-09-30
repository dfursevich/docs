### Example 
Order and Stock. Create a new order and update stock for a selected item.

### 1. ACID transaction
Order and Stock are tables in a database. Within transaction boundaries insert a record into the Order table and update a record in the Stack table.

### 2. No transaction
Create new order and update stock are two separate API calls.
First call **create new order** API second call **update stock** API. If second call fails we are in inconsistent state when the order is created but stock is not updated. It can be handled manually by monitoring exceptions in a log file.

### 3. Eventual consistent distributed transaction
Create new order and update stock are two separate API calls.
First call **create new order** API second call **update stock** API. The transaction can be in the following states: CREATED, ORDER_CREATED, STOCK_UPDATED. So to track a state of the transaction the database entity in status CREATED is inserted into the database and operation return. 
```
--------------
|Transaction |
--------------
|id          |
|state       |
|create_date |
|try_count   |
|try_date    |
|input       |
--------------
```
Asynchronous thread picks up the transaction in not terminal state, performs corresponding API call and updates transaction state. If there is a failure the async thread will retry the whole operation. Let's consider possible scenarios of the operation when the async thread fetches the transaction in state CREATED
 1. Thread successfully call **create new order** API and then updates transaction state to ORDER_CREATED.
 2. Thread successfully call **create new order** API and fails to update transaction state to ORDER_CREATED because database is not available. As soon as the database will become available the thread will again fetch this transaction in state CREATED and retry it.
 3. Thread successfully call **create new order** API and executing service halts due to blackout for example. As soon as service is live again the async thread will fetch this transaction in state CREATED and retry it.  
 4. Thread fails while calling **create new order** API then updates **try_count** and **try_date** of the transaction and retry it in some period of time. 

So the situation when API call is retried is highly probable which specify additional requirements to API implementation i.e. all APIs
must be **idempotent**. Being idempotent means to be able to process the same call multiple times without changing the state of the system.

#### Event sourcing
Instead of creating transaction object which maintains status we've got an event object. Event object has different semantics with transaction, it's the object that describes a business action with data. Persisting event means the same that persisting transaction object from technical point. In a way that an async thread maintains list of newly crated events and send them to message broker. 
```
--------------
|Event       |
--------------
|entity_id   |
|event_name  |
|event_date  |
|create_date |
|fire_date   |
|fire_count  |
--------------
```
Unlike transactions events have another application. They are used to reconstruct the state of business entity by replaying all events for the same entity from the first event.

Instead of programming separate thread that executes operations asynchronously a special storage can be used that can persist an event and fire it atomically. Kafka is not appropriate choice since if we use it for storage as well we won't be able to fetch events for the single entity.  


### 4. Two-phase commit transaction
TODO
