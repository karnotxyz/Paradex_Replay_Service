
## Context :

This is a transaction replay service, it picks up transaction from one node (hear-on : original node) and replays them to another node (here-on : syncing node).

The service currently supports Paradex blocks 95271 till latest.
i.e :
- `0.13.2` & above.

## Pre-requisites :
### Images :

- Madara : [paradex_sync_d13b45c1b041bfa6b490f15ce21509d50ac15fd1](https://hub.docker.com/layers/prkpandey942/madara/paradex_sync_d13b45c1b041bfa6b490f15ce21509d50ac15fd1/images/sha256-df6625997176672a640bc05b9d05fe238cf430935692ce9f2da51ecefe480726)
- Replay Service: [d809c15fb74dd6c33559773a590ae7adec43787b_runner](https://hub.docker.com/layers/prkpandey942/transaction_syncing_service/d809c15fb74dd6c33559773a590ae7adec43787b_runner/images/sha256-4ead1f28fb3b72b5e1fec0b8c808d2c85777020f27766152507b7c7e950c6a44)
- Paradex Chain Config file, provided in `configs` folder
- Versioned Constants, provided in `configs` folder

!IMP : Plase note to change the `latest_protocol_version` field in `configs/paradex.yaml` to the version you are replaying

### Database :

Based on the support, you will require `Madara database` >=  95270 Paradex blocks.
_Contact for support here !_


## Commands :

### ENVs :
```
RPC_URL_ORIGINAL_NODE=<PARADEX_NETWORK_NODE_RPC_URL>
RPC_URL_SYNCING_NODE=<MADARA_NODE_RPC_URL>
ADMIN_RPC_URL_SYNCING_NODE=<MADARA_NODE_ADMIN_RPC_URL>
```

### Transaction Syncing Service :

```sh
docker run -d \
  --name transaction_syncing_service \
  -p 3000:3000 \
  -e RPC_URL_ORIGINAL_NODE=PARADEX_NETWORK_NODE_RPC_URL \
  -e RPC_URL_SYNCING_NODE=MADARA_NODE_RPC_URL \
  -e ADMIN_RPC_URL_SYNCING_NODE=MADARA_NODE_ADMIN_RPC_URL \
  DOCKER_IMAGE_TAG
```

### Madara :
```sh
docker run -d \
  --name madara \
  -p 9944:9944 \
  -p 9943:9943 \
  -p 8080:8080 \
  -v /path/to/madara-db:/data \
  -v /path/to/configs:/configs \
  -e RPC_URL_ORIGINAL_NODE=PARADEX_NETWORK_NODE_RPC_URL/rpc/v0_8 \
  -e RUST_LOG=info \
  DOCKER_IMAGE_TAG \
  --name madara \
  --base-path /data \
  --rpc-port 9944 \
  --rpc-admin-port 9943 \
  --rpc-cors "*" \
  --gateway-port 8080 \
  --strk-per-eth 1 \
  --rpc-external \
  --rpc-admin \
  --rpc-admin-external \
  --feeder-gateway-enable \
  --no-l1-sync \
  --gateway-enable \
  --gateway-external \
  --sequencer \
  --chain-config-path /configs/paradex.yaml
```

## API Documentation

### Base Configuration

- **Base URL**: `http://localhost:3000`
- **Content-Type**: `application/json`
### Endpoints

#### 1. Start Sync Process

Initiates a new blockchain synchronisation process for a specified block range.

**Endpoint**: `POST /sync`

##### Request Body

| Parameter      | Type    | Required | Description                                               |
| -------------- | ------- | -------- | --------------------------------------------------------- |
| `syncFrom`     | integer | Yes      | Starting block number for synchronization                 |
| `syncTo`       | integer | Yes      | Ending block number for synchronization                   |
| `startTxIndex` | integer | Yes      | Transaction index to start from within the starting block |

##### Example Request

```bash
curl --request POST \
  --url http://localhost:3000/sync \
  --header 'Content-Type: application/json' \
  --data '{
    "syncFrom": 830006,
    "syncTo": 830010,
    "startTxIndex": 0
  }'
```

##### Success Response

**Status Code**: `200 OK`

```json
{
  "message": "Sync process started successfully",
  "processId": "024630a7-3b1f-4997-95dd-60048e5e6264",
  "syncMode": "fixed_range",
  "status": {
    "syncFrom": 830006,
    "syncTo": 830010,
    "startTxIndex": 0,
    "estimatedBlocks": 5
  }
}
```

##### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `message` | string | Status message indicating success |
| `processId` | string | Unique identifier for the sync process (UUID format) |
| `syncMode` | string | Type of synchronization mode |
| `status.syncFrom` | integer | Starting block number |
| `status.syncTo` | integer | Ending block number |
| `status.startTxIndex` | integer | Starting transaction index |
| `status.estimatedBlocks` | integer | Total number of blocks to be processed |

---

#### 2. Cancel Sync Process

Cancels an active synchronisation process with options for graceful shutdown.

**Endpoint**: `POST /sync/cancel

##### Path Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `processId` | string | Yes | The process ID returned from the start sync endpoint |

##### Request Body

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `complete_current_block` | boolean | Yes | Controls how the cancellation is handled:<br>• `true`: Completes processing the current block before stopping<br>• `false`: Stops the sync process immediately |

##### Example Request

```bash
curl --request POST \
  --url http://localhost:3000/sync/cancel \
  --header 'Content-Type: application/json' \
  --data '{
    "complete_current_block": true
  }'
```

---

#### 3. Health Check

Checks the overall health and availability of the API service.

**Endpoint**: `GET /health`

##### Example Request

```bash
curl --request GET \
  --url http://localhost:3000/health
```

##### Success Response

**Status Code**: `200 OK`

Returns the current health status of the service.

---

#### 4. Sync Status

Retrieves the current status of all sync processes or the overall sync system.

**Endpoint**: `GET /sync/status`

##### Example Request

```bash
curl --request GET \
  --url http://localhost:3000/sync/status
```

##### Success Response

**Status Code**: `200 OK`

Returns detailed information about current sync operations, including progress and status.

### Notes

- Process IDs are generated as UUIDs and should be stored for future reference
- The `complete_current_block` parameter allows for graceful shutdown to maintain data integrity
- Block ranges are inclusive (both `syncFrom` and `syncTo` blocks will be processed)
- Transaction indexing starts from 0 within each block
