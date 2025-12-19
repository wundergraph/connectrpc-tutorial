# Part 3: Consuming the API

**Learning Goal:** Learn multiple ways to interact with your ConnectRPC service and generate client SDKs.

**Time:** ~20 minutes

---

## What You'll Learn

In this part, you'll learn:
- How to call your API using HTTP/JSON with `curl`
- How to call your API using gRPC with `grpcurl`
- How to generate TypeScript and Go SDKs
- How to generate an OpenAPI specification
- How to use the generated SDKs in your applications

## Concepts

### Protocol Flexibility

One of the key benefits of ConnectRPC is protocol flexibility. The same service can be consumed via:
- **HTTP/JSON** - Simple REST-like calls using curl or any HTTP client
- **gRPC** - High-performance binary protocol
- **Typed SDKs** - Generated client libraries with full type safety
- **OpenAPI** - Standard API documentation format

From the API consumer's perspective, GraphQL is no longer part of the picture. The API behaves like a conventional gRPC or HTTP service with well-defined request and response contracts.

---

## Section 1: Using curl (HTTP/JSON)

The simplest way to interact with your API is using HTTP/JSON requests.

### Query: GetEmployees (POST)

Retrieve all employees:

```shell
curl -X POST http://localhost:5026/employees.v1.HrService/GetEmployees \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{}'
```

### Query: GetEmployees (GET)

The same query using GET (enabled by the `NO_SIDE_EFFECTS` option):

```shell
curl --get \
  --data-urlencode 'encoding=json' \
  --data-urlencode 'message={}' \
  --data-urlencode 'connect=v1' \
  http://localhost:5026/employees.v1.HrService/GetEmployees
```

**Why GET matters:** GET requests can be cached by CDNs and browsers, improving performance for read operations.

### Query: GetEmployeeById (POST)

Retrieve a specific employee:

```shell
curl -X POST http://localhost:5026/employees.v1.HrService/GetEmployeeById \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{"id": 1}'
```

### Query: GetEmployeeById (GET)

```shell
curl --get \
  --data-urlencode 'encoding=json' \
  --data-urlencode 'message={"id": 1}' \
  --data-urlencode 'connect=v1' \
  http://localhost:5026/employees.v1.HrService/GetEmployeeById
```

### Mutation: UpdateEmployeeMood (POST)

Update an employee's mood:

```shell
curl -X POST http://localhost:5026/employees.v1.HrService/UpdateEmployeeMood \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{"id": 1, "mood": "MOOD_HAPPY"}'
```

**Note:** Mutations only support POST (not GET) because they have side effects.

---

## Section 2: Using grpcurl (gRPC)

For gRPC clients, you can use `grpcurl` to interact with your service.

### Query: GetEmployees

```shell
grpcurl -plaintext \
  -proto ./services/service.proto \
  -d '{}' \
  localhost:5026 \
  employees.v1.HrService/GetEmployees
```

### Query: GetEmployeeById

```shell
grpcurl -plaintext \
  -proto ./services/service.proto \
  -d '{"id": 1}' \
  localhost:5026 \
  employees.v1.HrService/GetEmployeeById
```

### Mutation: UpdateEmployeeMood

```shell
grpcurl -plaintext \
  -proto ./services/service.proto \
  -d '{"id": 1, "mood": "MOOD_HAPPY"}' \
  localhost:5026 \
  employees.v1.HrService/UpdateEmployeeMood
```

**Command breakdown:**
- `-plaintext` - Use unencrypted connection (for local development)
- `-proto` - Path to your proto file
- `-d` - Request data in JSON format
- `localhost:5026` - Server address
- `employees.v1.HrService/GetEmployees` - Fully qualified method name

---

## Section 3: Generating SDKs and OpenAPI

To generate client SDKs and OpenAPI specifications, you'll use [Buf](https://buf.build), a modern tool for working with Protocol Buffers.

### Prerequisites

Install the required tools:

```shell
# Install Buf CLI
brew install bufbuild/buf/buf

# Install Go plugins for Go SDK generation
go install connectrpc.com/connect/cmd/protoc-gen-connect-go@latest
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest

# Install TypeScript/JavaScript plugins for TypeScript SDK generation
npm install --save-dev @bufbuild/protoc-gen-es @connectrpc/protoc-gen-connect-es

# Install OpenAPI plugin
go install connectrpc.com/connect/cmd/protoc-gen-connect-openapi@latest
```

Ensure the Go bin directory is in your PATH:

```shell
export PATH="$PATH:$(go env GOPATH)/bin"
```

### Configuration Files

This project includes two Buf configuration files:

**[`buf.yaml`](../buf.yaml)** - Configures the Buf module:
- Defines the module name and path
- Excludes `node_modules` from processing
- Enables standard linting rules to ensure proto file quality
- Configures breaking change detection to maintain API compatibility

**[`buf.gen.yaml`](../buf.gen.yaml)** - Configures code generation:
- **Managed mode**: Automatically manages Go package paths
- **Go SDK plugins**: Generates Go client and server stubs
- **TypeScript SDK plugins**: Generates TypeScript client code
- **OpenAPI plugin**: Generates OpenAPI specification
- All outputs are organized into the `gen/` directory by type

### Generate Everything

Generate TypeScript SDK, Go SDK, and OpenAPI specification with a single command:

```shell
buf generate
```

Buf will automatically discover the proto files in your module and generate all outputs.

**Generated files:**
- **Go SDK**: `gen/go/` - Go client and server stubs
- **TypeScript SDK**: `gen/ts/` - TypeScript client code
- **OpenAPI**: `gen/openapi/` - OpenAPI specification

### View the OpenAPI Specification

```shell
cat gen/openapi/service.openapi.yaml
```

The OpenAPI spec describes all your RPC methods as HTTP endpoints, making it easy to:
- Import into API documentation tools (Swagger UI, Redoc, etc.)
- Generate client SDKs in various languages
- Share API contracts with external teams
- Test APIs using OpenAPI-compatible tools

---

## Section 4: Using the Generated SDKs

### TypeScript SDK

#### Install Runtime Dependencies

```shell
npm install @connectrpc/connect @connectrpc/connect-web
```

#### Example Usage

```typescript
import { createPromiseClient } from "@connectrpc/connect";
import { createConnectTransport } from "@connectrpc/connect-web";
import { HrService } from "./gen/ts/service_connect";

// Create a transport
const transport = createConnectTransport({
  baseUrl: "http://localhost:5026",
});

// Create a client
const client = createPromiseClient(HrService, transport);

// Call the API
const response = await client.getEmployeeById({ id: 1 });
console.log(response.employee);
```

**Benefits:**
- ✅ Full TypeScript type safety
- ✅ Autocomplete in your IDE
- ✅ Compile-time error checking
- ✅ No need to write request/response types manually

### Go SDK

#### Install Runtime Dependencies

```shell
go get connectrpc.com/connect
```

#### Example Usage

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"

    "connectrpc.com/connect"
    employeesv1 "github.com/wundergraph/connectrpc-demo/gen/go/employees/v1"
    "github.com/wundergraph/connectrpc-demo/gen/go/employees/v1/employeesv1connect"
)

func main() {
    client := employeesv1connect.NewHrServiceClient(
        http.DefaultClient,
        "http://localhost:5026",
    )

    req := connect.NewRequest(&employeesv1.GetEmployeeByIdRequest{
        Id: 1,
    })

    res, err := client.GetEmployeeById(context.Background(), req)
    if err != nil {
        log.Fatal(err)
    }

    fmt.Printf("Employee: %+v\n", res.Msg.Employee)
}
```

**Benefits:**
- ✅ Strongly typed Go structs
- ✅ Compile-time type checking
- ✅ Idiomatic Go error handling
- ✅ High performance with gRPC protocol support

---

## Section 5: Keeping SDKs in Sync

Whenever you modify your GraphQL operations or regenerate your proto files, simply run:

```shell
buf generate
```

This ensures your SDKs stay in sync with your API contract.

**Workflow:**
1. Update GraphQL operations in `services/*.graphql`
2. Regenerate proto: `wgc grpc-service generate ...`
3. Regenerate SDKs: `buf generate`
4. Restart the router to pick up changes

---

## Checkpoint

**Checklist:**
- [ ] You can call the API using `curl` (HTTP/JSON)
- [ ] You can call the API using `grpcurl` (gRPC)
- [ ] You have generated TypeScript SDK in `gen/ts/`
- [ ] You have generated Go SDK in `gen/go/`
- [ ] You have generated OpenAPI spec in `gen/openapi/`
- [ ] You understand how to keep SDKs in sync with your API

**Verify generated files:**

```shell
ls -R gen/
```

You should see directories for `go/`, `ts/`, and `openapi/`.

---

## What You Learned

✅ How to consume your API using HTTP/JSON with curl  
✅ How to consume your API using gRPC with grpcurl  
✅ The difference between GET and POST for queries  
✅ How to use Buf to generate client SDKs  
✅ How to generate OpenAPI specifications  
✅ How to use TypeScript and Go SDKs in your applications  
✅ How to keep SDKs in sync with your API contract

---

## Tutorial Complete! 🎉

Congratulations! You've successfully:

- ✅ Converted GraphQL operations into Protocol Buffer definitions
- ✅ Run a ConnectRPC server using the Cosmo Router
- ✅ Consumed the same API via multiple protocols
- ✅ Generated strongly typed SDKs for TypeScript and Go
- ✅ Generated an OpenAPI specification

### Key Takeaways

**For API Producers:**
- You can expose GraphQL APIs to non-GraphQL consumers without rewriting resolvers
- Trusted Documents provide stable API contracts that can be versioned
- The router handles all protocol translation automatically

**For API Consumers:**
- You can use familiar tools and protocols (HTTP, gRPC, typed SDKs)
- You get full type safety without learning GraphQL
- The same API surface works across multiple languages and platforms

### Next Steps

- Explore adding more operations to your service
- Try generating SDKs for other languages using Buf plugins
- Integrate the generated SDKs into your applications
- Set up CI/CD to automatically regenerate SDKs when operations change
- Explore versioning strategies for your proto definitions

### Resources

- 📚 [Cosmo Documentation](https://cosmo-docs.wundergraph.com/)
- 📚 [ConnectRPC Documentation](https://connectrpc.com/)
- 📚 [Buf Documentation](https://buf.build/docs)
- 💬 [WunderGraph Community](https://wundergraph.com/community)
- 🐛 [Report an Issue](https://github.com/wundergraph/connectrpc-demo/issues)

---

**Navigation:**
- [← Back to Part 2: Running the ConnectRPC Server](./part-2-connectrpc-server.md)
- [← Back to Tutorial Overview](../README.md)
