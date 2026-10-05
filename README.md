<!--
SPDX-License-Identifier: Apache-2.0
-->

# OpenSOVD Starter Template

> Build a Diagnostic Fault Manager (DFM), run the OpenSOVD gateway, and expose
> live faults over a standards-compliant SOVD HTTP interface.

This template shows you how to stand up a minimal **diagnostic chain**:

```
your app --(iceoryx2 IPC)--> DFM (fault-lib) --(dfm/query)--> OpenSOVD gateway
    --> SOVD HTTP: GET /sovd/v1/apps/<app>/faults
```

- **DFM** (from [`fault-lib`](https://github.com/eclipse-opensovd/fault-lib))
  stores and manages the ISO 14229 (UDS) fault lifecycle.
- **OpenSOVD Core** (from
  [`opensovd-core`](https://github.com/eclipse-opensovd/opensovd-core))
  serves faults over the
  **SOVD (Service-Oriented Vehicle Diagnostics)** HTTP API defined by
  **[ISO 17978-3](https://standards.iso.org/iso/17978/-3/ed-1/en/)**
  (OpenAPI electronic inserts).

### Source repositories

| Component | Repository |
|-----------|------------|
| OpenSOVD Core (gateway) | <https://github.com/eclipse-opensovd/opensovd-core> |
| DFM (fault-lib, incl. `dfm_bin`) | <https://github.com/eclipse-opensovd/fault-lib/> |
| SOVD OpenAPI (ISO 17978-3) | <https://standards.iso.org/iso/17978/-3/ed-1/en/> |

> [!NOTE]
> Sections 2 and 3 work with the upstream repositories as-is. Section 4
> (DFM → OpenSOVD faults) is left for the challenge makers to fill in.

---

## 1. Prerequisites

| Tool | Why |
|------|-----|
| [Rust](https://rustup.rs/) toolchain | Builds OpenSOVD Core and the DFM (auto-pinned via `rust-toolchain.toml`) |
| `protobuf-compiler` (`protoc`) | Proto build step for providers |
| `jq`, `curl` | Inspecting the SOVD HTTP responses |
| Docker (optional) | Running the prebuilt gateway image |
| [`uv`](https://docs.astral.sh/uv/) (optional) | Python integration tests |

Linux is assumed; the DFM uses **iceoryx2** shared-memory IPC.

---

## 2. Build and run OpenSOVD Core

### 2.1 Clone and build

```bash
git clone https://github.com/eclipse-opensovd/opensovd-core.git
cd opensovd-core
cargo build
```

### 2.2 Run the gateway (mock data, no DFM)

The fastest way to see a live SOVD endpoint — serves built-in mock entities:

```bash
# From source
cargo run -p opensovd-gateway -- --mock

# Or the published container image
docker run -p 7690:7690 ghcr.io/eclipse-opensovd/opensovd-gateway --mock
```

Verify it is up:

```bash
curl -s http://127.0.0.1:7690/sovd/version-info | jq
```

```json
{
  "sovd_info": [
    {
      "version": "1.1",
      "base_uri": "http://127.0.0.1:7690/sovd/v1",
      "vendor_info": { "version": "0.1.1", "name": "OpenSOVD" }
    }
  ]
}
```

The gateway binds `http://localhost:7690/sovd` by default. Override with
`--url http://0.0.0.0:8080/sovd` (or `SOVD_URL`), or listen on a Unix socket
with `--unix-socket /tmp/opensovd.sock`.


---

## 3. Build and run the DFM (fault-lib)

The DFM is provided by the `fault-lib` workspace. It ships a standalone binary,
`dfm_bin`, that loads JSON fault catalogs and persists fault state in a KVS
store.

### 3.1 Clone and build

```bash
git clone https://github.com/eclipse-opensovd/fault-lib.git
cd fault-lib

# Build just the standalone manager binary
cargo build --bin dfm_bin

# Or build the whole workspace (libraries, examples, tests)
cargo build --workspace
```

### 3.2 Run the DFM

`dfm_bin` requires **both** a catalog directory and a storage directory.
Omitting `--storage-dir` makes it exit immediately.

```bash
mkdir -p ./dfm-storage

./target/debug/dfm_bin \
  --catalog-dir ./diagnostics/catalog \
  --storage-dir ./dfm-storage
```

- `--catalog-dir` — directory of `*.json` fault catalogs. Each catalog's app
  ID (e.g. `battery_guardian`) becomes a SOVD entity path.
- `--storage-dir` — directory for KVS persistent fault state (survives restart).

Once running, the DFM exposes an iceoryx2 `dfm/query` service that the OpenSOVD
gateway connects to.

### 3.3 (Optional) Try it with the library examples

```bash
# Terminal 1 — start a DFM with built-in demo catalogs
cargo run -p dfm_lib --example dfm

# Terminal 2 — a reporter app that publishes faults over IPC
cargo run -p fault_lib --features testutils --example tst_app -- \
  -c src/fault_lib/tests/data/hvac_fault_catalog.json
```

A **fault catalog** is a JSON file describing each fault code and its trigger,
for example:

| Fault code | Trigger |
|---|---|
| `BatteryOverTempWarning`  | Max cell temp ≥ 45 °C |
| `BatteryOverTempCritical` | Max cell temp ≥ 55 °C |
| `BatteryTempSignalStale`  | No fresh sample within the freshness deadline |

Your own application reports `Failed`/`Passed` transitions for these codes to
the DFM via the fault-lib `Reporter` API (iceoryx2 IPC).

---

## 4. Let OpenSOVD read from the DFM and serve the faults interface

> [!NOTE]
> **To be filled in by the challenge makers.**

**Goal:** the OpenSOVD gateway reads fault records from the running DFM
(section 3.2) and serves them on the SOVD faults endpoint defined by
[ISO 17978-3](https://standards.iso.org/iso/17978/-3/ed-1/en/), e.g.
`GET /sovd/v1/apps/<app>/faults`.

### 4.1 Build the gateway with DFM support

```bash
# TODO(challenge makers): build command
```

### 4.2 Connect the gateway to the DFM

```bash
# TODO(challenge makers): run command / configuration
```

### 4.3 Query the SOVD faults endpoint

```bash
# TODO(challenge makers): example request and expected response
```

### 4.4 End-to-end order of operations

<!-- TODO(challenge makers): describe the startup order and verification steps -->


---

## 5. The SOVD interface (ISO 17978-3)

OpenSOVD implements **SOVD (Service-Oriented Vehicle Diagnostics)**, the
HTTP/REST + JSON diagnostic API standardized as **ISO 17978**. Part 3 defines
the machine-readable **OpenAPI** description of the interface:

- Spec / OpenAPI inserts: <https://standards.iso.org/iso/17978/-3/ed-1/en/>

Commonly used routes exposed by the gateway:

| Method & path | Purpose |
|---|---|
| `GET /sovd/version-info` | Gateway and SOVD version / base URI |
| `GET /sovd/v1/apps/<app>/faults` | List faults for an app entity |
| `GET /sovd/v1/components/<id>/faults` | List faults for a component entity |

Because the interface follows the ISO OpenAPI contract, any SOVD-compliant
client (or generated SDK) can read the faults your DFM produces.

---

## 6. Troubleshooting

| Symptom | Fix |
|---|---|
| `dfm_bin` exits immediately | Pass **both** `--catalog-dir` and `--storage-dir`. |
| `/faults` returns empty | Ensure the DFM is running first and a reporter has published fault transitions. |
| Connection refused on 7690 | Check the gateway bound address via `--url` / `SOVD_URL`. |
| Proto build fails | Install `protobuf-compiler` (`protoc`). |

---

## 7. References

- [OpenSOVD Core](https://github.com/eclipse-opensovd/opensovd-core) — <https://github.com/eclipse-opensovd/opensovd-core>
- [fault-lib (DFM)](https://github.com/eclipse-opensovd/fault-lib/) — <https://github.com/eclipse-opensovd/fault-lib/>
- [ISO 17978-3 SOVD OpenAPI](https://standards.iso.org/iso/17978/-3/ed-1/en/)
- [iceoryx2 IPC](https://github.com/eclipse-iceoryx/iceoryx2)

## License

Apache License 2.0.
