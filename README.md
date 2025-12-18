# ConnectRPC Demo

This demo project provides a tutorial on getting started with WunderGraph ConnectRPC.

* [0. Pre Requisites](#0-pre-requisites)
* [1. Operations to Proto](#1-operations-to-proto)
* [2. ConnectRPC Server](#2-connectrpc-server)
* [3. Consuming the API](#3-consuming-the-api)
  * [3a. curl](#3a-curl)
  * [3b. grpcurl](#3b-grpcurl)
  * [3c. Generating OpenAPI](#3c-generating-openapi)
  * [3d. Generating Client SDKs](#3d-generating-client-sdks)

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

For your convenience, we have copied the federated graph schema into [schema.graphqls](./schema.graphqls).

We need to author *Named Operations* (also known as *Trusted Documents*), and save them as `.graphql` files into the `./services` directory within this repository.

Feel free to create your own operations, we have prepared three for you below to copy into your terminal.

The operation name will map to an rpc method in the generated proto. So it's a good to think about the job-to-be-done or action of the rpc. 

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
    startDate
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
mutation UpdateEmployeeMood($employeeID: Int!, $mood: Mood!) {
  updateMood(employeeID: $employeeID, mood: $mood) {
    id
    currentMood
    derivedMood
    updatedAt
  }
}
EOF
```

Once we have ouf collection of operations, in the services directory, we then need to use the wgc cli tool to compile the schema and operations into a proto.

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

We have pre-prepared a `connect.config.yaml` file for you which turns on ConnectRPC Server mode on the router.

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

TODO

### 3a. curl

TODO

### 3b. grpcurl

TODO

### 3c. Generating OpenAPI

TODO

### 3d. Generating Client SDKs

TODO

