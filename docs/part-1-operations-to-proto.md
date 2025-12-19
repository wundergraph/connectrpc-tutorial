# Part 1: Operations to Proto

**Learning Goal:** Understand how to define GraphQL operations and compile them into Protocol Buffer definitions.

**Time:** ~15 minutes

---

## What You'll Learn

In this part, you'll learn:
- How to author named GraphQL operations (Trusted Documents)
- How to use the `wgc` CLI to generate Protocol Buffer definitions
- How GraphQL operations map to RPC methods and messages

## Concepts

### Trusted Documents

Trusted Documents (also called Named Operations) are pre-defined GraphQL queries and mutations that act as stable API contracts. By compiling these into a strongly typed RPC interface, you create a predictable API surface that can be versioned and evolved safely.

### Protocol Buffers

Protocol Buffers (protobuf) is a language-neutral, platform-neutral mechanism for serializing structured data. The `.proto` file defines your service interface, including:
- **Service**: The collection of RPC methods
- **RPC Methods**: Individual operations (like `GetEmployeeById`)
- **Messages**: Request and response types

## Step 1: Review the GraphQL Schema

For your convenience, the federated graph schema has been copied into [`schema.graphqls`](../schema.graphqls).

This schema defines the types and fields available in your GraphQL API. You'll use this as the foundation for creating operations.

## Step 2: Create GraphQL Operations

You need to author **Named Operations** and save them as `.graphql` files in the [`services/`](../services) directory.

The operation name will map to an RPC method in the generated proto. Think about the job-to-be-done or action of each RPC.

### Create GetEmployeeById Operation

This query retrieves a single employee by ID:

```shell
cat << 'EOF' > services/GetEmployeeById.graphql
query GetEmployeeById($id: Int!) {
  employee(id: $id) {
    id
    details {
      forename
      surname
      nationality
    }
    currentMood
    isAvailable
    role {
      title
      departments
    }
    tag
    updatedAt
  }
}
EOF
```

### Create GetEmployees Operation

This query retrieves all employees:

```shell
cat << 'EOF' > services/GetEmployees.graphql
query GetEmployees {
  employees {
    id
    details {
      forename
      surname
      nationality
    }
    currentMood
    isAvailable
    role {
      title
      departments
    }
    tag
  }
}
EOF
```

### Create UpdateEmployeeMood Operation

This mutation updates an employee's mood:

```shell
cat << 'EOF' > services/UpdateEmployeeMood.graphql
mutation UpdateEmployeeMood($id: Int!, $mood: Mood!) {
  updateMood(employeeID: $id, mood: $mood) {
    id
    currentMood
    derivedMood
    updatedAt
  }
}
EOF
```

## Step 3: Generate the Proto File

Now use the `wgc` CLI to compile the schema and operations into a proto file:

```shell
wgc grpc-service generate \
  --with-operations ./services \
  --input ./schema.graphqls \
  --output ./services \
  --package-name employees.v1 \
  HRService
```

**Command breakdown:**
- `--with-operations`: Directory containing your `.graphql` files
- `--input`: Your GraphQL schema file
- `--output`: Where to write the generated proto files
- `--package-name`: Proto package namespace (use reverse domain notation)
- `HRService`: The name of your gRPC service

**Expected output:**

You should see two new files in the `./services` directory:
- `service.proto` - The Protocol Buffer definition
- `service.proto.lock.json` - Maintains forward-compatibility and field-number stability

## Step 4: Examine the Generated Proto

Let's look at the generated [`service.proto`](../services/service.proto):

```protobuf
syntax = "proto3";
package employees.v1;

service HrService {
  rpc GetEmployeeById(GetEmployeeByIdRequest) returns (GetEmployeeByIdResponse) {
    option idempotency_level = NO_SIDE_EFFECTS;
  }

  rpc GetEmployees(GetEmployeesRequest) returns (GetEmployeesResponse) {
    option idempotency_level = NO_SIDE_EFFECTS;
  }

  rpc UpdateEmployeeMood(UpdateEmployeeMoodRequest) returns (UpdateEmployeeMoodResponse) {}
}
```

**Key observations:**
- Package name is `employees.v1` as specified
- Service is called `HrService` as specified
- Each operation became an RPC method
- Query operations have `option idempotency_level = NO_SIDE_EFFECTS;` which enables GET requests and CDN caching

### Understanding the Message Mapping

The request message maps to GraphQL query variables:

```protobuf
message GetEmployeeByIdRequest {
  int32 id = 1;
}
```

The response message maps to the GraphQL selection set:

```protobuf
message GetEmployeeByIdResponse {
  message Employee {
    message Details {
      string forename = 1;
      string surname = 2;
      Nationality nationality = 3;
    }
    message Role {
      repeated string title = 1;
      repeated Department departments = 2;
    }
    int32 id = 1;
    Details details = 2;
    Mood current_mood = 3;
    bool is_available = 4;
    Role role = 5;
    string tag = 6;
    string updated_at = 7;
  }
  Employee employee = 1;
}
```

Notice how:
- GraphQL types become nested messages
- Field names are converted to snake_case (GraphQL convention → protobuf convention)
- GraphQL types map to protobuf types (String → string, Int → int32, Boolean → bool)

## Checkpoint

Verify you have all the required files:

```shell
ls -1 services/
```

**Expected output:**
```
GetEmployeeById.graphql
GetEmployees.graphql
UpdateEmployeeMood.graphql
service.proto
service.proto.lock.json
```

**Checklist:**
- [ ] You have 5 files in the `services/` directory
- [ ] `GetEmployeeById.graphql` exists
- [ ] `GetEmployees.graphql` exists
- [ ] `UpdateEmployeeMood.graphql` exists
- [ ] `service.proto` exists
- [ ] `service.proto.lock.json` exists

## What You Learned

✅ How to author named GraphQL operations as Trusted Documents  
✅ How to use `wgc grpc-service generate` to create proto files  
✅ How GraphQL operations map to RPC methods  
✅ How GraphQL types and fields map to protobuf messages  
✅ The purpose of the `.proto.lock.json` file for API stability

## Next Steps

Now that you have your proto definitions, you're ready to run the ConnectRPC server.

**Continue to:** [Part 2: Running the ConnectRPC Server](./part-2-connectrpc-server.md)

---

**Navigation:**
- [← Back to Tutorial Overview](../README.md)
- [Next: Part 2 - Running the ConnectRPC Server →](./part-2-connectrpc-server.md)
