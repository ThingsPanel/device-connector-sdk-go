# ThingsPanel Device Connector SDK (Go)

Go SDK for building ThingsPanel device connectors. The SDK handles HTTP routing,
heartbeat, startup device sync, and graceful shutdown so connector authors focus
only on device capability mapping.

## What the SDK does for you

- Registers all ThingsPanel HTTP callback routes (`/api/v1/form/config`,
  `/api/v1/device/add`, `/api/v1/device/disconnect`, etc.)
- Exposes `/health` for K8S readiness and liveness probes
- Sends `POST /api/v1/plugin/heartbeat` on a configurable interval
- On first successful heartbeat: calls `/api/v1/plugin/service/access/list` and
  replays `OnDeviceAdd` for every already-bound device (startup sync)
- Reads runtime identity and MQTT broker from environment variables
- Handles graceful shutdown on SIGTERM

## Handler interface

Every connector implements `sdk.Handler`:

```go
type Handler interface {
    FormConfig(ctx context.Context) (FormConfig, error)
    OnDeviceAdd(ctx context.Context, req DeviceAddRequest) error
    OnDeviceDelete(ctx context.Context, req DeviceDeleteRequest) error
    OnCommand(ctx context.Context, req CommandRequest) (CommandResponse, error)
    OnConfigUpdate(ctx context.Context, req ConfigUpdateRequest) error
    OnDisconnect(ctx context.Context, req DisconnectRequest) error
    OnEvent(ctx context.Context, ev EventNotification) error
}
```

## Optional interfaces

| Interface | Purpose |
|---|---|
| `FormConfigProvider` | Return different schemas for access-point vs per-device forms |
| `RawFormDataProvider` | Return non-schema payloads for legacy form types (e.g. VCRT) |
| `DeviceLister` | Discover devices from a cloud credential (service-access pattern) |

## Connector patterns

### Direct-device connector

Devices connect directly to the connector (e.g. Modbus TCP, raw MQTT).

- `FormConfig` returns the credential/config schema
- `OnDeviceAdd` stores credentials; the device then connects
- Report data: publish to `devices/telemetry` with MQTT access token

### Service-access connector (cloud discovery)

Devices live on a third-party cloud platform (e.g. Ezviz, Xiaomi, HomeAssistant).

- Implement `DeviceLister` — ThingsPanel calls it to show the device picker
- `OnDeviceAdd` receives merged `device_config` (service voucher + per-device config)
- The connector polls or subscribes to the cloud platform and reports data via MQTT
- The SDK automatically calls `syncBoundDevices` on startup so no devices are lost
  after a process restart

```
service-access flow:
  user fills access-point form (app_key, app_secret)
  → ThingsPanel calls GET /api/v1/plugin/device/list?voucher=...
  → connector calls cloud API, returns DiscoveredDevice list
  → user selects devices and binds them to a template
  → ThingsPanel calls POST /api/v1/device/add for each selected device
  → connector stores config, starts polling/subscribing cloud platform
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

func (h *myHandler) OnDeviceAdd(ctx context.Context, req sdk.DeviceAddRequest) error {
    // req.DeviceID    — ThingsPanel device UUID
    // req.DeviceNumber — human-readable identifier (e.g. "modbus-192.168.1.10-1")
    // req.DeviceConfig — merged map from form fields
    // req.AccessToken — MQTT credential for publishing telemetry
    return nil
}

func (h *myHandler) OnDeviceDelete(ctx context.Context, req sdk.DeviceDeleteRequest) error { return nil }
func (h *myHandler) OnCommand(ctx context.Context, req sdk.CommandRequest) (sdk.CommandResponse, error) {
    return sdk.CommandResponse{OK: true}, nil
}
func (h *myHandler) OnConfigUpdate(ctx context.Context, req sdk.ConfigUpdateRequest) error { return nil }
func (h *myHandler) OnDisconnect(ctx context.Context, req sdk.DisconnectRequest) error     { return nil }
func (h *myHandler) OnEvent(ctx context.Context, ev sdk.EventNotification) error           { return nil }

func main() {
    info := sdk.FromEnv()
    server := sdk.NewServer(info, &myHandler{})

    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    server.Run(ctx)
}
```

## Publishing telemetry

Use `req.AccessToken` as the MQTT username. Publish to topic `devices/telemetry`:

```go
// connect with username = req.AccessToken
opts := mqtt.NewClientOptions()
opts.AddBroker(info.MQTTBroker)   // sdk.ConnectorInfo.MQTTBroker from env
opts.SetUsername(req.AccessToken)
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
| `THINGSPANEL_BACKEND_URL` | yes* | — | Backend base URL for heartbeat and startup sync |
| `CONNECTOR_HEARTBEAT_INTERVAL` | no | `30s` | How often to POST heartbeat |
| `TP_MQTT_BROKER` | no† | — | MQTT broker address (preferred name) |
| `MQTT_BROKER` | no† | — | MQTT broker address (fallback name) |

*Heartbeat and startup sync are disabled (with a warning) if unset.  
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

### v0.2.0
- `ConnectorInfo.MQTTBroker` — reads `TP_MQTT_BROKER` / `MQTT_BROKER` from env
- `DeviceAddRequest.DeviceNumber` — human-readable device identifier now populated during startup sync
- `syncBoundDevices` — automatic on first heartbeat; replays `OnDeviceAdd` with merged `device_config` and `DeviceNumber`
- `RawFormDataProvider` interface for VCRT/legacy form types

### v0.1.0
- Initial release: HTTP routing, heartbeat, `Handler` interface
