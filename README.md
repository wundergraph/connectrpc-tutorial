# ConnectRPC Demo
### From GraphQL to OpenAPI and Typed SDKs

---

This demo project provides a tutorial on getting started with WunderGraph ConnectRPC.

* [Overview](#overview)
* [Who is this for?](#who-this-is-for)
* [0. Pre Requisites](#0-pre-requisites)
* [1. Operations to Proto](#1-operations-to-proto)
* [2. ConnectRPC Server](#2-connectrpc-server)
* [3. Consuming the API](#3-consuming-the-api)
  * [3a. curl](#3a-curl)
  * [3b. grpcurl](#3b-grpcurl)
  * [3c. Generating SDKs and OpenAPI specs](#3c-generating-sdks-and-openapi-specs)

## Overview

This demo shows how to expose an existing GraphQL API in Cosmo as a ConnectRPC Service, without rewriting resolvers or changing your underlying Graph.

By compiling **Trusted Documents** into a strongly typed RPC interface, Cosmo acts as a translation layer enabling different API Consumers to interact with the Graph using the protocol and tooling which fits them best.

The goal is to reduce friction between API producers and API consumers. Platform and API teams can continue to design, evolve and govern their APIs in GraphQL, while consumers gain access through **strongly typed SDKs**, **OpenAPI**, **gRPC** - without the need to understand or adopt GraphQL themselves.

**By the end of this demo, you will have:**

- Taken an existing federated GraphQL API running in Cosmo
- Defined trusted GraphQL operations as stable API contracts
- Generated a strongly typed ConnectRPC service from those operations
- Served that service using the Cosmo Router
- Consumed the same API via:
  - curl (HTTP/JSON)
  - grpcurl (gRPC)
  - Generated TypeScript and Go SDKs
  - An OpenAPI specification

## Who this is for

This demo is intended for users who already have a **monolithic or federated graph running in Cosmo** and want to expose it via ConnectRPC.

It is especially relevant for the following roles:

### API Producers

Teams building and evolving GraphQL schemas who want to:

* Expose APIs to non-GraphQL consumers safely via trusted operations
* Avoid maintaining multiple API implementations for the same domain

### API Consumers

Teams consuming APIs who want to:

* Use strongly typed SDKs, OpenAPI, or RPC-style APIs
* Integrate with familiar tooling and workflows
* Gain value from GraphQL-backed services without needing to learn GraphQL

## 0. Pre Requisites

For this project, we will assume that you have already onboarded onto Cosmo, and have the demo subgraphs and federated graph up and running
as per our onboarding documentation [Cosmo Cloud Onboarding](https://cosmo-docs.wundergraph.com/getting-started/cosmo-cloud-onboarding).

#### Sanity Check

- [ ] You have completed the Cosmo Cloud onboarding
- [ ] A federated graph named `demo` exists in the `development` namespace
- [ ] The `Products`, `Employee`, `Mood`, and `Availability` subgraphs are running
- [ ] You can successfully query the federated graph via GraphQL
- [ ] The `wgc` CLI is installed and available in your `$PATH`
- [ ] You have a valid `GRAPH_API_TOKEN`

## 1. Operations to Proto

For your convenience, we have copied the federated graph schema into [`schema.graphqls`](./schema.graphqls).

We need to author *Named Operations* (also known as *Trusted Documents*), and save them as `.graphql` files into the `./services` directory within this repository.

Feel free to create your own operations, we have prepared three for you below to copy into your terminal.

The operation name will map to an rpc method in the generated proto. So it's good to think about the job-to-be-done or action of the rpc. 

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

Once we have our collection of operations, in the services directory, we then need to use the wgc cli tool to compile the schema and operations into a proto.

```shell
wgc grpc-service generate \
  --with-operations ./services \
  --input ./schema.graphqls \
  --output ./services \
  --package-name employees.v1 \
  HRService
```

* `--with-operations`: Directory containing your .graphql files
* `--input`: Your GraphQL schema file
* `--output`: Where to write the generated proto files
* `--package-name`: Proto package namespace (use reverse domain notation)
* `HRService`: The name of your gRPC service

Upon running `wgc grpc-service generate`, we should see two new files in the `./services` directory, a `service.proto` and `service.proto.lock.json`.

The generated `service.proto.lock.json` is for maintaining forward-compatibility and field-number stability as your API evolves and as you add more operations.

Let's examine the generated `service.proto`.

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

From here, we can see that as requested, our generated proto is in the package `employees.v1`, the Service is called `HrService` and we have three RPCs, where each rpc method name maps to the respective named operation.

> ℹ️ **Note**
>
> Query operations are generated with the proto method option  
> `option idempotency_level = NO_SIDE_EFFECTS;`.
> 
> This enables the client to send GET requests which enables CDN caching.

Looking into the generated messages, we can see that the request message is analogous to the GraphQL Query variables, and the response message maps to the selection set:

```protobuf
message GetEmployeeByIdRequest {
  int32 id = 1;
}

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
    string start_date = 8;
  }
  Employee employee = 1;
}

...
```

#### Sanity Check

- [ ] You have 5 files in the `services/` directory
- [ ] `GetEmployeeById.graphql` exists
- [ ] `GetEmployees.graphql` exists
- [ ] `UpdateEmployeeMood.graphql` exists
- [ ] `service.proto` exists
- [ ] `service.proto.lock.json` exists

You can verify this by running:

```shell
ls -1 services/         

# GetEmployeeById.graphql
# GetEmployees.graphql
# UpdateEmployeeMood.graphql
# service.proto
# service.proto.lock.json
```

## 2. ConnectRPC Server

We have pre-prepared a [`connect.config.yaml`](./connect.config.yaml) file for you which turns on ConnectRPC Server mode on the router.

```yaml
# ./connect.config.yaml
connect_rpc:
  enabled: true
  server:
    listen_addr: "localhost:5026"
  services_provider_id: "fs-services"
  graphql_endpoint: "http://localhost:3002/graphql"

storage_providers:
  file_system:
    - id: "fs-services"
      path: "./services"
```

We configure the router to load services from the `./services` directory, and to start the ConnectRPC Server on port 5026.

```shell
docker run \
  --name cosmo-router \
  --rm \
  -p 3002:3002 \
  -p 5026:5026 \
  --add-host=host.docker.internal:host-gateway \
  --pull always \
  -e DEV_MODE=true \
  -e LISTEN_ADDR=0.0.0.0:3002 \
  -e CONFIG_PATH=/config/connect.config.yaml \
  -e GRAPH_API_TOKEN="<graph-api-token>" \
  -v "$(pwd)/connect.config.yaml:/config/connect.config.yaml:ro" \
  -v "$(pwd)/services:/services:ro" \
  ghcr.io/wundergraph/cosmo/router:latest
```

Key docker command flags:

* `-p 5026:5026`: Maps ConnectRPC server port (host:container)
* `--add-host=host.docker.internal:host-gateway`: Allows container to reach services on host machine because your subgraphs will be on your host machine
* `-e LISTEN_ADDR=0.0.0.0:3002`: Binds GraphQL server to all interfaces
* `-e CONFIG_PATH=/config/connect.config.yaml`: Path to config file inside container
* `-e GRAPH_API_TOKEN="<graph-api-token>"`: Authentication token for Cosmo Cloud (you will have this token from prerequisites)
* `-v "$(pwd)/connect.config.yaml:/config/connect.config.yaml:ro"`: Mounts local config file (read-only)
* `-v "$(pwd)/services:/services:ro"`: Mounts services directory containing proto files (read-only

Inspecting the router logs, we should see:

```text
INFO core/router.go:873 GraphQL schema coverage metrics enabled {... }
INFO connectrpc/service_discovery.go:127 discovered service {... , "full_name": "employees.v1.HrService", "package": "employees.v1", "service": "HrService", "dir": "./services", "proto_files": 1, "operation_files": 3}
INFO connectrpc/service_discovery.go:148 service discovery complete {... , "total_services": 1, "services_dir": "./services"}
INFO connectrpc/operation_registry.go:130 loaded operations for service {... , "service": "employees.v1.HrService", "operation_count": 3}
INFO connectrpc/vanguard_service.go:118 registering services {... , "package_count": 1, "service_count": 1, "total_methods": 3}
INFO core/router.go:1033 ConnectRPC server ready {... , "listen_addr": "localhost:5026", "services": 1, "operations": 3}
INFO core/router.go:1054 Serving GraphQL playground {... , "url": "http://localhost:3002/"}
INFO connectrpc/server.go:198 ConnectRPC server ready {... , "addr": "127.0.0.1:5026"}
```

## 3. Consuming the API

At this point, the GraphQL operations you authored have been compiled into a stable, strongly typed ConnectRPC service and are being served by the Cosmo Router. From the perspective of an API consumer, GraphQL is no longer part of the picture - the API now behaves like a conventional gRPC or HTTP service with well-defined request and response contracts.

In this section, we'll explore several different ways to interact with the same API surface, all backed by the exact same GraphQL operations:

### 3a. `curl`

GetEmployees (POST)
```shell
curl -X POST http://localhost:5026/employees.v1.HrService/GetEmployees \
  -H "Content-Type: application/json" \
  -H "Connect-Protocol-Version: 1" \
  -d '{}'
```

GetEmployees (GET)
```shell
 curl --get \
  --data-urlencode 'encoding=json' \
  --data-urlencode 'message={}' \
  --data-urlencode 'connect=v1' \
  http://localhost:5026/employees.v1.HrService/GetEmployees
```

GetEmployeeById (POST)
```shell
curl -X POST http://localhost:5026/employees.v1.HrService/GetEmployeeById \
    -H "Content-Type: application/json" \
    -H "Connect-Protocol-Version: 1" \
    -d '{"id": 1}'         
```

GetEmployeeById (GET)
```shell
curl --get \
  --data-urlencode 'encoding=json' \
  --data-urlencode 'message={"id": 1}' \
  --data-urlencode 'connect=v1' \
  http://localhost:5026/employees.v1.HrService/GetEmployeeById
```

UpdateEmployeeMood (POST)
```shell
curl -X POST http://localhost:5026/employees.v1.HrService/UpdateEmployeeMood \
    -H "Content-Type: application/json" \
    -H "Connect-Protocol-Version: 1" \
    -d '{"id": 1, "mood": "MOOD_HAPPY"}'
```

### 3b. `grpcurl`

GetEmployees
```shell
grpcurl -plaintext \
  -proto ./services/service.proto \
  -d '{}' \
  localhost:5026 \
  employees.v1.HrService/GetEmployees
```

GetEmployeeById
```shell
grpcurl -plaintext \
  -proto ./services/service.proto \
  -d '{"id": 1}' \
  localhost:5026 \
  employees.v1.HrService/GetEmployeeById
```

UpdateEmployeeMood
```shell
grpcurl -plaintext \
  -proto ./services/service.proto \
  -d '{"id": 1, "mood": "MOOD_HAPPY"}' \
  localhost:5026 \
  employees.v1.HrService/UpdateEmployeeMood
```

### 3c. Generating SDKs and OpenAPI specs

To generate OpenAPI specifications and client SDKs from your proto files, you'll use [Buf](https://buf.build), a modern tool for working with Protocol Buffers. Buf provides a streamlined way to generate code and documentation from proto files, supporting multiple output formats including OpenAPI specs, TypeScript SDKs, and Go SDKs.

#### Prerequisites

It is possible to use remote plugins, but these can come with rate-limits, so for the purposes of this demo, we will use local tools to achieve the same.

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

#### Configuration

This project includes two Buf configuration files:

**[`buf.yaml`](buf.yaml)** - Configures the Buf module:
* Defines the module name and path
* Excludes `node_modules` from processing
* Enables standard linting rules to ensure proto file quality
* Configures breaking change detection to maintain API compatibility

**[`buf.gen.yaml`](buf.gen.yaml)** - Configures code generation:
* **Managed mode**: Automatically manages Go package paths
* **Go SDK plugins**: Generates Go client and server stubs using `protoc-gen-go` and `protoc-gen-connect-go`
* **TypeScript SDK plugins**: Generates TypeScript client code using `protoc-gen-es` and `protoc-gen-connect-es`
* **OpenAPI plugin**: Generates OpenAPI specification using `protoc-gen-connect-openapi`
* All outputs are organized into the `gen/` directory by type

#### Generate SDKs and OpenAPI

Generate TypeScript SDK, Go SDK, and OpenAPI specification with a single command:

```shell
buf generate
```

Buf will automatically discover the proto files in your module (as configured in [`buf.yaml`](buf.yaml)) and generate all outputs according to the plugins defined in [`buf.gen.yaml`](buf.gen.yaml).

This will create:
* **Go SDK**: `gen/go/` - Go client and server stubs
* **TypeScript SDK**: `gen/ts/` - TypeScript client code
* **OpenAPI**: `gen/openapi/` - OpenAPI specification

The generated OpenAPI spec describes all your RPC methods as HTTP endpoints, making it easy to:
* Import into API documentation tools (Swagger UI, Redoc, etc.)
* Generate client SDKs in various languages
* Share API contracts with external teams
* Test APIs using OpenAPI-compatible tools

You can view the generated OpenAPI specification:

```shell
cat gen/openapi/service.openapi.yaml
```

#### Using the Generated SDKs

##### TypeScript SDK

Install the required runtime dependencies:

```shell
npm install @connectrpc/connect @connectrpc/connect-web
```

Example usage:

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

##### Go SDK

Install the required runtime dependencies:

```shell
go get connectrpc.com/connect
```

Example usage:

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

#### Updating SDKs

Whenever you modify your GraphQL operations or regenerate your proto files, simply run:

```shell
buf generate
```

This ensures your SDKs stay in sync with your API contract.
