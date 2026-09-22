# Deep Dive

### 1. Create Order

We have added an idempotency key to be sent through headers to prevent client to create order twice either because of retries or a poor network connection.

The API should validate that there suffcient amount to place this order and once the user added the order to its cart, and the product state should be reserved and the total available amount of the product should be decreased 

This will prevent an important edge case where there is only one remaining product 
allowing 2 users to place it to their cart may end up with race condition that affect the correctness.


### 2. Process the order

After the order is created on the database the user reveiving 201 created, but there are a lot of work to be done after that we should place where this order existed and then allocate the robot who can achieve this task based on the weight and time and availability.

Another major concern here is the post being created on the database level but failed to compelete the rest of business logic.

We couldn't create transaction on adding the new order to db and these other operations which would be implemented asynchrounusly. So I would use a message broker like kafka but this also will not gurantee to success together

### 3. Out of box pattern

To solve the previously mentioned problem, and to benefit from the database transaction which insterting the order to database and add the event to the out of box table, and this should succeed togther and this is gurantee even if kafka failed, the event was saved and would be retried when available

### 4. Inventory

Placing where each order is existed is a highly requested service so for butter performance this information should be stored in redis but this wouldn't be the source of truth, 
it just would be ephemeral data, our primary database would be postgresql 

### 5. Robot allocation

As mentioned the robot allocation algorithm will be treated as a black box but there's couple of things should be considered the robot will send a heartbeat
each 10 seconds this will help us now the online robots, the robot may use its location as a heartbeat and it would be stored on redis and also we could apply
a geohashing index on it for faster retreival.

### 6. Robot failure

when a robot doesn't send a heartbeat over than 10s it should consider as offline and we check if it is has a task that doesn't start yet we assign it to another robot
and if it is already working on a task we can wait for a reasonable amount of time (like e.g. 1 hour TTL) if it still not working we will find another available robot to complete it

### 7. Redis Failure

We depend on redis for heartbeat and location update, what we would do if it fails? falling back to the database would be massive if all these update suddenly goes to it,
for some senarios using a different cluster may solve the issue but if all of them unavailable. The system should be tolerant to this fail without crash