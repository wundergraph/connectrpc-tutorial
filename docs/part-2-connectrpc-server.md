# Part 2: Running the ConnectRPC Server

**Learning Goal:** Configure and run the Cosmo Router with ConnectRPC enabled.

**Time:** ~10 minutes

---

## What You'll Learn

In this part, you'll learn:
- How to configure the Cosmo Router for ConnectRPC
- How to mount your proto services into the router
- How to verify the ConnectRPC server is running correctly

## Concepts

### ConnectRPC Server Mode

The Cosmo Router can run in ConnectRPC server mode alongside its GraphQL endpoint. In this mode, the router:
- Loads your proto service definitions
- Maps RPC calls to the corresponding GraphQL operations
- Handles protocol translation (gRPC, HTTP/JSON, etc.)
- Executes the GraphQL operations against your federated graph

This means you don't need to write any server-side code - the router handles everything.

## Step 1: Review the Configuration

A [`connect.config.yaml`](../connect.config.yaml) file has been prepared for you. Let's examine it:

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

**Configuration breakdown:**

- `connect_rpc.enabled: true` - Turns on ConnectRPC server mode
- `server.listen_addr` - The address where the ConnectRPC server will listen (port 5026)
- `services_provider_id` - References the storage provider that contains your proto files
- `graphql_endpoint` - The GraphQL endpoint that will execute the operations
- `storage_providers.file_system` - Configures the router to load services from the `./services` directory

## Step 2: Start the Router

Run the Cosmo Router with ConnectRPC enabled:

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

**Important:** Replace `<graph-api-token>` with your actual token from the prerequisites.

**Docker command breakdown:**

- `-p 3002:3002` - Maps the GraphQL server port (host:container)
- `-p 5026:5026` - Maps the ConnectRPC server port (host:container)
- `--add-host=host.docker.internal:host-gateway` - Allows the container to reach services on your host machine (needed because your subgraphs run on the host)
- `-e LISTEN_ADDR=0.0.0.0:3002` - Binds the GraphQL server to all interfaces
- `-e CONFIG_PATH=/config/connect.config.yaml` - Path to the config file inside the container
- `-e GRAPH_API_TOKEN="<graph-api-token>"` - Your Cosmo Cloud authentication token
- `-v "$(pwd)/connect.config.yaml:/config/connect.config.yaml:ro"` - Mounts your local config file as read-only
- `-v "$(pwd)/services:/services:ro"` - Mounts your services directory containing proto files as read-only

## Step 3: Verify the Server is Running

Inspect the router logs. You should see output similar to:

```text
INFO core/router.go:873 GraphQL schema coverage metrics enabled {...}
INFO connectrpc/service_discovery.go:127 discovered service {..., "full_name": "employees.v1.HrService", "package": "employees.v1", "service": "HrService", "dir": "./services", "proto_files": 1, "operation_files": 3}
INFO connectrpc/service_discovery.go:148 service discovery complete {..., "total_services": 1, "services_dir": "./services"}
INFO connectrpc/operation_registry.go:130 loaded operations for service {..., "service": "employees.v1.HrService", "operation_count": 3}
INFO connectrpc/vanguard_service.go:118 registering services {..., "package_count": 1, "service_count": 1, "total_methods": 3}
INFO core/router.go:1033 ConnectRPC server ready {..., "listen_addr": "localhost:5026", "services": 1, "operations": 3}
INFO core/router.go:1054 Serving GraphQL playground {..., "url": "http://localhost:3002/"}
INFO connectrpc/server.go:198 ConnectRPC server ready {..., "addr": "127.0.0.1:5026"}
```

**Key log messages to look for:**

1. **Service discovery** - Confirms the router found your proto files
   ```
   discovered service {..., "service": "HrService", "proto_files": 1, "operation_files": 3}
   ```

2. **Operations loaded** - Confirms your GraphQL operations were loaded
   ```
   loaded operations for service {..., "operation_count": 3}
   ```

3. **Server ready** - Confirms the ConnectRPC server is listening
   ```
   ConnectRPC server ready {..., "listen_addr": "localhost:5026", "services": 1, "operations": 3}
   ```

## Understanding What Just Happened

The router has:
1. ✅ Discovered your `service.proto` file in the `./services` directory
2. ✅ Found the 3 GraphQL operation files (`.graphql` files)
3. ✅ Mapped each RPC method to its corresponding GraphQL operation
4. ✅ Started listening on port 5026 for ConnectRPC requests
5. ✅ Kept the GraphQL endpoint running on port 3002

Now when a client calls an RPC method like `GetEmployeeById`, the router will:
1. Receive the RPC request (via gRPC or HTTP/JSON)
2. Translate it to the corresponding GraphQL operation
3. Execute the GraphQL query against your federated graph
4. Translate the GraphQL response back to the RPC response format
5. Return it to the client

## Checkpoint

**Checklist:**
- [ ] The router container is running
- [ ] You see "ConnectRPC server ready" in the logs
- [ ] The log shows 1 service discovered
- [ ] The log shows 3 operations loaded
- [ ] Port 5026 is listening for ConnectRPC requests
- [ ] Port 3002 is still serving GraphQL

**Quick test:**

You can verify the server is responding by running:

```shell
curl http://localhost:5026/
```

You should see a response indicating the ConnectRPC server is running.

## What You Learned

✅ How to configure the Cosmo Router for ConnectRPC mode  
✅ How to mount proto services using file system storage providers  
✅ How to run the router with both GraphQL and ConnectRPC endpoints  
✅ How to verify the ConnectRPC server is running correctly  
✅ The router's role as a protocol translation layer

## Next Steps

Now that your ConnectRPC server is running, you're ready to consume the API using different protocols and generate SDKs.

**Continue to:** [Part 3: Consuming the API](./part-3-consuming-the-api.md)

---

**Navigation:**
- [← Back to Part 1: Operations to Proto](./part-1-operations-to-proto.md)
- [Next: Part 3 - Consuming the API →](./part-3-consuming-the-api.md)
