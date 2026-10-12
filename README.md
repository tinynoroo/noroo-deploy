# noroo-deploy

`deployctl`, a deploy tool for hosting many sites on a single server, and the rulebook used to run that server.

Pick a repository and one command takes it from port assignment to a verified HTTPS response. If anything fails along the way, only what that run created is rolled back.

```
deployctl init
```

## What's in here

| File | Contents |
|---|---|
| [`deployctl`](deployctl) | The deploy tool. A single file using only the Python standard library |
| [`CLAUDE.md`](CLAUDE.md) | The server rulebook (Korean). The rules Claude Code follows when working on this server, and the reasoning behind how `deployctl` behaves |
| [`mac-server-setup.md`](mac-server-setup.md) | Setup notes (Korean) from when this started on a Mac mini. No longer maintained |

## Architecture

```
Internet ──► Caddy (80/443, automatic certificates)
                │  127.0.0.1:9000–9099
                ▼
          One container per site (Docker Compose)
                │  dbnet
                ▼
          Shared Postgres (a dedicated database and user per site)
```

- Only ports 80 and 443 face the internet, and only Caddy uses them. Containers bind to `127.0.0.1` only.
- Postgres publishes no ports. Each site can reach only its own database, and isolation is verified right after the database is created.
- A site is defined by a single file, `~/srv/<site>/app.json`. The Caddy config is generated from it.
- Deploys run on a GitHub Actions self-hosted runner, one per repository, which calls `deployctl deploy`.

## Commands

| Command | What it does |
|---|---|
| `init` | Registers a site: pick a repository → clone → assign a port → create a database → set up the workflow and runner → first deploy |
| `validate` | Checks the rules: port and domain conflicts, DNS, image architecture, `127.0.0.1` binding, connection headroom, and more |
| `deploy` | Validate → build → start → register with Caddy → confirm HTTPS. Rolls back on failure |
| `status` | Containers, Caddy registration, HTTPS, and connection usage |
| `scaffold` | Drafts a missing `Dockerfile` or compose file. The Dockerfile is written by `claude -p` and verified with `docker build` |
| `workflow` | Generates `.github/workflows/deploy.yml` |
| `runner` | Registers or removes a repository's dedicated Actions runner |
| `remove` | Tears down the containers and the Caddy site. The database and manifest are kept |

Run `deployctl <command> --help` for options. The details and the reasons behind them are in [`CLAUDE.md` §11](CLAUDE.md).

## Design principles

- **All or nothing.** If `init` fails partway, it removes only the containers, Caddy file, database, and manifest created by that run. Anything that already existed is left alone.
- **Existing files are never overwritten.** A hand-written `Dockerfile`, compose file, or workflow stays as it is unless you pass `--force`. Rule violations are reported, not fixed.
- **Rules are checked by the machine.** What a person would otherwise have to remember, `validate` checks before every deploy.
- **No LLM in the deploy path.** `claude` is used only by `scaffold`, to produce drafts. Deploys on the runner must be reproducible.
- **Self-hosted runners attach to private repositories only.** Public repositories are left out of the `init` list.

## Prerequisites

`deployctl` is not a general-purpose tool. It is one server's way of operating, written down as code. To use it as is, you need:

- Linux (amd64/arm64) or macOS, Python 3, and Docker with the Compose plugin
- Caddy running on the host with its admin API open at `127.0.0.1:2019`
- A shared Postgres at `~/srv/postgres`, running as a container named `postgres` on the `dbnet` network
- The `gh` CLI, logged in (used for the repository list, cloning, and runner registration)
- The Caddy config path is fixed at `/opt/homebrew/etc/`. On Linux, put a symlink there pointing to `/etc/caddy` ([`CLAUDE.md` §6-1](CLAUDE.md))

Paths, the port range, and similar values are constants at the top of `deployctl`.

## About the names in the docs

The account name, domain, and site name were replaced with `deploy`, `example.com`, and `myapp` for publication. The dated work notes in the docs are a real record of moving the server from macOS to Linux.
