# rapid446_sdl

Docker-compose deployment of the RAPID-446 self-driving lab on MADSci v0.8.
Brings up seven managers (lab, event, experiment, resource, data, workcell,
location) plus the seven node containers that live on the lab host
(`grimm.cels.anl.gov`).

```bash
docker compose up -d        # bring the whole lab up
docker compose ps           # check status
docker compose logs -f      # tail logs
```

The five suestorm-hosted nodes (hidex, bmg, two inheco, biometra) are managed
on their own host and just referenced by URL from `settings.yaml`.

---

## Layout

```
rapid446_sdl/
├── compose.yaml             # top-level — includes the three docker/ files
├── settings.yaml            # SHARED lab defaults  (tracked)
├── .env                     # SHARED secrets       (gitignored)
├── nodes/
│   └── <node_name>/
│       ├── node.settings.yaml   # per-node defaults (tracked)
│       └── node.env             # per-node overrides/secrets (gitignored)
├── docker/
│   ├── compose.infra.yaml       # FerretDB / Valkey / Postgres x2
│   ├── compose.managers.yaml    # 7 managers
│   └── grimm_nodes.compose.yaml # 7 node services on grimm
└── .madsci/                 # runtime state (gitignored) — DBs, registry,
                             # datapoints, backups
```

---

## Changing or overriding a config value

Configuration is layered. Where you put a value depends on two questions:
**is it a default that should ship with the lab, or a local override?** and
**does it apply to the whole lab, or to one node?**

|                       | Shared (whole lab)        | Per-node                                    |
| --------------------- | ------------------------- | ------------------------------------------- |
| **Default**           | `settings.yaml` (tracked) | `nodes/<name>/node.settings.yaml` (tracked) |
| **Override / secret** | `.env` (gitignored)       | `nodes/<name>/node.env` (gitignored)        |

Pydantic-settings merges all four sources at startup. **Highest precedence
wins**, in this order:

1. Process env vars (compose `environment:` blocks, host env)
2. `nodes/<name>/node.env`            ← per-node override / secret
3. `.env` (repo root)                 ← shared override / secret
4. `nodes/<name>/node.settings.yaml`  ← per-node committed default
5. `settings.yaml` (repo root)        ← shared committed default

The walk-up discovery is what makes this work: each node container starts
with `MADSCI_SETTINGS_DIR=/home/madsci/nodes/<name>`, and pydantic-settings
walks up from there until it finds `.madsci/` (the project root sentinel),
picking up the per-node files in the node dir and the shared files in the
repo root along the way.

### Manager example

To change how often the workcell polls its nodes for status:

```yaml
# settings.yaml
workcell_node_update_interval: 1.5      # was 3.0
```

```bash
docker compose restart workcell_manager
```

To inject a database password without committing it:

```dotenv
# .env  (gitignored)
RESOURCE_DB_PASSWORD=...
DOCUMENT_DB_PASSWORD=...
```

### Node example

To change the USB device path that `sealer_sassy` talks to as a permanent
project default — edit the tracked YAML:

```yaml
# nodes/sealer_sassy/node.settings.yaml
device_path: /dev/ttyUSB2
```

To swap to a different device path for one shift without touching git, create
an override file:

```dotenv
# nodes/sealer_sassy/node.env   (gitignored — create on demand)
NODE_DEVICE_PATH=/dev/ttyUSB99
```

Then:

```bash
docker compose restart sealer_sassy
```

Delete `node.env` (or the offending line) when the override is no longer
needed.

### Env-var form of any YAML key

Every YAML key can also be set as an environment variable by **uppercasing
the key** and prefixing it with the relevant settings prefix. The prefixes:

| Settings class | Prefix        | Example key             | Env var                  |
| -------------- | ------------- | ----------------------- | ------------------------ |
| Lab            | `LAB_`        | `lab_manager_name`      | `LAB_MANAGER_NAME`       |
| Event manager  | `EVENT_`      | `event_database_name`   | `EVENT_DATABASE_NAME`    |
| Experiment     | `EXPERIMENT_` | `experiment_server_url` | `EXPERIMENT_SERVER_URL`  |
| Resource       | `RESOURCE_`   | `resource_server_url`   | `RESOURCE_SERVER_URL`    |
| Data           | `DATA_`       | `data_database_name`    | `DATA_DATABASE_NAME`     |
| Workcell       | `WORKCELL_`   | `workcell_nodes`        | `WORKCELL_NODES` (JSON)  |
| Location       | `LOCATION_`   | `location_server_url`   | `LOCATION_SERVER_URL`    |
| Node           | `NODE_`       | `device_path`            | `NODE_DEVICE_PATH`      |

A few `NodeConfig` fields use explicit aliases to avoid double-prefixing
(`NODE_NAME`, `NODE_ID`, `NODE_TYPE` rather than `NODE_NODE_NAME` etc.). The
YAML key is still the unprefixed field name (`node_name`, `node_id`,
`node_type`).

### `node.env` is gitignored on purpose

`nodes/*/node.env` is gitignored so secrets and host-specific overrides don't
get committed. If you find yourself adding the *same* override every time you
recreate the file, that's a hint it should move into `node.settings.yaml`
instead.

---

## Further reading

- [`madsci/CLAUDE.md`](../madsci/CLAUDE.md) — overall MADSci architecture and
  the full configuration precedence rules.
- [`madsci/examples/example_lab/`](../madsci/examples/example_lab/) — the
  canonical reference layout for a v0.8 lab.
- [`MIGRATION_v0.6_to_v0.8.md`](../MIGRATION_v0.6_to_v0.8.md) — the v0.6→v0.8
  migration plan this repo is currently executing.
