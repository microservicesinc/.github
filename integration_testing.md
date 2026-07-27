# Integration Testing Brief: End-to-End Async Order Cycle Verification

This testing brief details the step-by-step commands to execute, monitor, and trace the distributed transaction cycle across your `.NET 10 Orders Service`, `AWS SQS`, and the `Python Inventory Service Lambda`.

---

## 🛠️ Global Environmental Context Setup
Before running the validation loops, set up your global shell terminal alias to target LocalStack seamlessly:
```bash
alias awslocal="aws --endpoint-url=http://localhost:4566 --region us-east-1"
```

---

## 🧪 Chronological Testing & Verification Steps

### 1. Initiate Request & Poll Initial Order Status
Submit the creation wireframe payload directly to the running `.NET 10 Minimal API` engine thread, then immediately capture its current tracking state using its unique identifier.

```bash
# A. Submit HTTP POST to create the order for item1
curl -X POST "http://localhost:5233/api/orders" \
  -H "Content-Type: application/json" \
  -d '{ "itemId": "item1", "traceId": "777a2f3577b34da6a3ce929d0e0e4736" }'

# [CRITICAL]: Copy the returned order id (e.g., ORD_A1B2C3D4E5F6) from the JSON payload response

# B. Submit HTTP GET to instantly check its initialization lifecycle state
curl -X GET "http://localhost:5233/api/orders/PASTE_YOUR_GENERATED_ORD_ID_HERE"
```

### 2. Verify Record in Orders DynamoDB Table
Confirm that the architectural domain translation layers properly committed the object state directly inside your Single-Table design storage layout with a `PENDING` value indicator.

```bash
awslocal dynamodb scan --table-name Orders
```
* **Expected Output Verification**: Look for the item matching your generated ID. It must display `"status": {"S": "PENDING"}` along with correct composite indices (`PK="ORDER#ORD_..."`, `SK="ITEM#item1"`).

### 3. Trace Outbound SQS Message to Inventory Queue
Verify that the `.NET 10` messaging client successfully wrapped and dispatched the integration event onto the message broker lines for the downstream service to consume.

```bash
awslocal sqs receive-message --queue-url http://localhost:4566/000000000000/StockUpdateQueue
```
* **Expected Output Verification**: The message body array wrapper must return your serialized application parameters: `{"itemId":"item1","quantityChange":-6}`.

### 4 & 5. Verify Lambda Execution and Direct Stock Reduction
The `StockUpdateQueue` automatically triggers your serverless Inventory Lambda container inside LocalStack. It parses the incoming JSON string payload and updates your inventory database state. Verify this change directly against the storage table:

```bash
# Query the Composite Table directly for item1's metadata tracking row
awslocal dynamodb get-item \
  --table-name InventoryOrdersTable \
  --key '{"PK": {"S": "ITEM#item1"}, "SK": {"S": "METADATA"}}'
```
* **Expected Output Verification**: The numerical stock indicator map key property should reflect the update, dropping from its baseline value of `9` directly down to **`"stock": {"N": "3"}`**.

---

## 🔮 Future Integration Testing Steps (Saga Completion Phase)

Once you return to code the asynchronous fallback loop mechanics, the final lifecycle confirmation states will be verified using the following execution commands:

### 6. (FUTURE) Mock SQS Confirmation Message for Orders Queue
Simulate your `Inventory Lambda` successfully finishing its transaction and dropping a positive validation callback message onto the Orders-owned reconciliation queue:

```bash
awslocal sqs send-message \
  --queue-url http://localhost:4566/000000000000/OrderValidationResponseQueue \
  --message-body '{"OrderId": "PASTE_YOUR_GENERATED_ORD_ID_HERE", "IsValid": true}'
```

### 7. (FUTURE) Verify Orders Lambda Status Change
Your .NET 10 Orders Lambda (or background responder thread) catches the validation event message, executes an atomic database operation, and cleans up the transaction. Confirm the database record state has moved past its initial boundary:

```bash
awslocal dynamodb scan --table-name Orders
```
* **Expected Future Verification**: The target record string inside your table should instantly transition from its old `PENDING` state to **`"status": {"S": "APPROVED"}`**.

### 8. Final Order Status Verification
Execute a final check against your public HTTP Minimal API endpoint interface to verify that the client can now see the updated, approved status:

```bash
curl -X GET "http://localhost:5233/api/orders/PASTE_YOUR_GENERATED_ORD_ID_HERE"
```
* **Expected JSON Response**:
  `{ "id": "ORD_...", "itemId": "item1", "traceId": "777...", "status": "APPROVED", "createdAt": "..." }`
