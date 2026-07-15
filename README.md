# ThingsPanel Device Connector SDK (Go)

Go SDK for building ThingsPanel protocol and service connectors. The SDK handles
stable HTTP contracts, heartbeat, and graceful shutdown. Device discovery,
configuration reconciliation, credentials, and telemetry policy stay in the
connector implementation.

## What the SDK does for you

- Registers the stable ThingsPanel callback routes
- Receives platform notifications at `/api/v1/plugin/notification`
- Keeps `/api/v1/notify/event` as a receive-only compatibility alias
- Exposes `/health` for K8S readiness and liveness probes
- Sends `POST /api/v1/plugin/heartbeat` on a configurable interval
- Reads runtime identity and MQTT broker from environment variables
- Handles graceful shutdown on SIGTERM

## Capability model

Pass any connector implementation to `NewServer`. Implement only the capabilities
the connector needs. Unsupported capabilities return HTTP 501 instead of requiring
empty methods or causing a nil callback panic.

## Optional interfaces

| Interface | Purpose |
|---|---|
| `FormConfigHandler` | Return the connector's default configuration form |
| `FormConfigProvider` | Return different schemas for access-point vs per-device forms |
| `RawFormDataProvider` | Return non-schema payloads for legacy form types (e.g. VCRT) |
| `DeviceLister` | Discover devices from a cloud credential (service-access pattern) |
| `CommandHandler` | Handle an optional HTTP command callback |
| `DisconnectHandler` | Disconnect a device on platform request |
| `NotificationHandler` | Receive platform notifications and trigger connector reconciliation |

## Connector patterns

### Direct-device connector

Devices connect directly to the connector (e.g. Modbus TCP, raw MQTT).

- `FormConfig` returns the credential/config schema
- `OnDisconnect` handles an explicit reconnect request when needed
- Report data: publish to `devices/telemetry` with MQTT access token

### Service-access connector (cloud discovery)

Devices live on a third-party cloud platform (e.g. Ezviz, Xiaomi, HomeAssistant).

- Implement `DeviceLister` — ThingsPanel calls it to show the device picker
- Implement `NotificationHandler` to receive `service_access.updated`
- On notification, the connector fetches and reconciles its bound devices
- The connector polls or subscribes to the cloud platform and reports data via MQTT
- The connector owns startup reconciliation and retry policy; the SDK never
  interprets or merges service vouchers and device configuration

```
service-access flow:
  user fills access-point form (app_key, app_secret)
  → ThingsPanel calls GET /api/v1/plugin/device/list?voucher=...
  → connector calls cloud API, returns DiscoveredDevice list
  → user selects devices and binds them to a template
  → ThingsPanel calls POST /api/v1/plugin/notification once
  → connector reconciles devices, starts polling/subscribing cloud platform
  → connector publishes telemetry via MQTT
```

## Minimal example

```go
package main

import (
    "context"
    "os/signal"
    "syscall"

    sdk "github.com/thingspanel/device-connector-sdk-go"
)

type myHandler struct{}

func (h *myHandler) FormConfig(ctx context.Context) (sdk.FormConfig, error) {
    return sdk.FormConfig{
        Schema: map[string]any{
            "type": "object",
            "properties": map[string]any{
                "host": map[string]any{"type": "string", "title": "Host"},
                "port": map[string]any{"type": "integer", "title": "Port", "default": 502},
            },
            "required": []string{"host"},
        },
    }, nil
}

func (h *myHandler) OnEvent(ctx context.Context, ev sdk.EventNotification) error {
    // A service connector reconciles its own state when service_access.updated arrives.
    return nil
}

func main() {
    info := sdk.FromEnv()
    server := sdk.NewServer(info, &myHandler{})

    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    server.Run(ctx)
}
```

## Publishing telemetry

Use the MQTT credential obtained during connector-owned reconciliation as the
username. Publish to topic `devices/telemetry`:

```go
// connect with the device's MQTT access token
opts := mqtt.NewClientOptions()
opts.AddBroker(info.MQTTBroker)   // sdk.ConnectorInfo.MQTTBroker from env
opts.SetUsername(accessToken)
client := mqtt.NewClient(opts)
client.Connect()

// publish telemetry
payload, _ := json.Marshal(map[string]any{
    "temperature": 23.5,
    "humidity":    60,
})
client.Publish("devices/telemetry", 0, false, payload)

// publish online/offline status
client.Publish("devices/status/"+deviceID, 0, false, []byte("1")) // 1=online, 0=offline
```

## Environment variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `CONNECTOR_SERVICE_IDENTIFIER` | yes | — | Must match `service_plugins.service_identifier` |
| `CONNECTOR_INSTANCE_ID` | yes | — | Connector instance UUID from control plane |
| `CONNECTOR_LISTEN_ADDR` | no | `:9001` | HTTP bind address |
| `THINGSPANEL_BACKEND_URL` | yes* | — | Backend base URL for heartbeat |
| `CONNECTOR_HEARTBEAT_INTERVAL` | no | `30s` | How often to POST heartbeat |
| `TP_MQTT_BROKER` | no† | — | MQTT broker address (preferred name) |
| `MQTT_BROKER` | no† | — | MQTT broker address (fallback name) |

*Heartbeat is disabled (with a warning) if unset.
†`info.MQTTBroker` is empty if neither is set; connector should handle gracefully.

## Local development

```bash
CONNECTOR_SERVICE_IDENTIFIER=my-connector \
CONNECTOR_INSTANCE_ID=local-dev \
THINGSPANEL_BACKEND_URL=http://localhost:9999 \
TP_MQTT_BROKER=tcp://localhost:1883 \
go run .
```

Verify:

```bash
curl http://localhost:9001/health
curl "http://localhost:9001/api/v1/form/config"
```

## Changelog

### Unreleased
- `/api/v1/plugin/notification` is the canonical notification endpoint
- `/api/v1/notify/event` remains a receive-only compatibility alias
- Connector features are optional capability interfaces; unsupported features return 501
- Removed SDK-owned startup device sync and the direct device add/delete/config routes
- Heartbeat requests have a bounded HTTP timeout

### v0.2.0
- `ConnectorInfo.MQTTBroker` — reads `TP_MQTT_BROKER` / `MQTT_BROKER` from env
- `DeviceAddRequest.DeviceNumber` — human-readable device identifier now populated during startup sync
- `syncBoundDevices` — automatic on first heartbeat; replays `OnDeviceAdd` with merged `device_config` and `DeviceNumber`
- `RawFormDataProvider` interface for VCRT/legacy form types

### v0.1.0
- Initial release: HTTP routing, heartbeat, `Handler` interface
