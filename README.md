# postgres-ha-digitalocean

Desired state for one three-node PostgreSQL failover cluster on DigitalOcean,
converged by the [`postgres-ha`](https://github.com/getcolors/postgres-ha)
Package Skill.

- three `s-2vcpu-4gb` droplets in `ams3`, in the region's default VPC
  (discovered, never configured);
- PostgreSQL 17, one primary and two hot standbys, quorum synchronous commit;
- Patroni 4.1.5 over a colocated three-member etcd 3.5.33;
- HAProxy on every node, so `pg-ha.bigconfig.website:5432` always reaches the
  current primary and `:5433` a standby;
- pgBackRest 2.59 daily full backups and continuous WAL archiving to the
  Cloudflare R2 bucket `postgres-ha-backup`;
- a verified restore every night, which replays the archive rather than only
  the backup.

```sh
direnv allow
./green build
./green create --dry-run
./green create
```

## Connecting

```sh
psql -h pg-ha.bigconfig.website -p 5432 -U postgres -d appdb   # read-write
psql -h pg-ha.bigconfig.website -p 5433 -U postgres -d appdb   # read-only
```

The name resolves to all three nodes. Each node's HAProxy forwards to whichever
one holds the Patroni leader lock, and libpq tries every resolved address in
turn — so a node being down is skipped by the client, and a failover changes
nothing about DNS.

Ingress is restricted to the operator CIDRs in `digitalocean-client-sources`.

## Operating

```sh
./green status                     # members, roles, replication lag
./green switchover                 # planned handover before a reboot
./green failover --node 2          # unplanned promotion, via a live node
./green backup                     # full backup now
./green verify-restore --node 2    # verified restore now
```

## Recovery

The full procedure — losing a standby, losing the primary, restoring to a point
in time, and rebuilding from R2 alone — is in the package's
[configuration reference](https://github.com/getcolors/postgres-ha/blob/main/skills/package-postgres-ha-green/references/configuration.md#recovery).

Briefly:

- **a standby is lost** — nothing to do; Patroni re-clones it when it returns;
- **the primary is lost** — nothing to do; the leader lock expires after 30
  seconds, a standby promotes, and every HAProxy follows within one health
  check, so `pg-ha.bigconfig.website` serves the new primary;
- **a point in time is needed** — stop Patroni everywhere, then
  `pgbackrest --stanza=main --delta --type=time --target='...' restore` on the
  node that will lead, let recovery finish, remove the Patroni cluster state,
  and start Patroni there first.

Credentials live in the gitignored `.envrc.private` as `COLORS_PAR_*` values.
Never set `COLORS_PAR_PROFILE`.
