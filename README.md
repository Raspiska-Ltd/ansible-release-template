# ansible-release-template

A small Ansible role for **atomic, rollback-able deploys** to one or more servers.

Every deploy goes into its own numbered folder. One symlink (`current`) decides which release is live.
Switching is a single rename, and rolling back is the same rename in the other direction.

```
/srv/myapp/
├── current -> releases/5     what your web server or service points at
├── releases/
│   ├── 3/
│   ├── 4/
│   └── 5/                    live
└── shared/
    ├── .env                  survives deploys, linked into every release
    └── storage/
```

## What a deploy does

1. Picks the next release number (1, 2, 3 ...) and creates `releases/N`.
2. Unpacks your artifact (a tarball or zip you built elsewhere) into it.
3. Links the shared files and folders into the release.
4. Runs your release commands (install, warm caches ...), still before anything is live.
5. Switches `current` to the new release (atomic symlink rename).
6. Restarts the service and waits for the health URL to answer `200`.
7. If that fails: points `current` back at the previous release, restarts, and fails with a clear message.
8. Prunes old releases (keeps the newest `keep_releases`, never the live one).

## Quick start

```bash
cp inventory.example.yml inventory.yml     # your servers, group "web"
cp vars/example.yml vars/myapp.yml         # your app: paths, shared files, restart command, health URL
# on each server, once: create /srv/myapp/shared/.env (whatever your app needs)

ansible-playbook deploy.yml -e app=myapp -e artifact=./dist/myapp.tar.gz
ansible-playbook rollback.yml -e app=myapp               # back to the previous release
ansible-playbook rollback.yml -e app=myapp -e to=12      # or to a specific one
```

Handy flags: `--limit web1` (canary: one server first), `-e serial=2` (two at a time), `--check` (dry run).

## Variables

| Variable | Default | Meaning |
|---|---|---|
| `base_dir` | (required) | Where `releases/`, `shared/` and `current` live |
| `artifact` | (required) | Local path of the tarball or zip to ship |
| `app_user` | unset | Owner of the files; leave unset to keep the connecting user |
| `keep_releases` | `5` | Release folders to keep |
| `shared_files` | `[]` | Files in `shared/` linked into each release, e.g. `[".env"]` |
| `shared_dirs` | `[]` | Folders in `shared/` linked into each release (created on first use) |
| `release_commands` | `[]` | Shell commands run inside the new release before it goes live |
| `restart_command` | `""` | Run after every switch, e.g. `systemctl reload myapp` |
| `health_url` | `""` | Must answer `200` after the switch; empty skips the check |
| `health_retries` / `health_delay` | `10` / `3` | How long to wait for it |

## Things to know

- **Restart or reload is on you.** A static site needs none. A Node, Python or Ruby process keeps running from the
  old release folder until it is restarted, so set `restart_command`.
- **FPM + nginx:** pass `$realpath_root` (not `$document_root`) as `SCRIPT_FILENAME`, so opcache sees the real
  release path and not the symlink.
- **Migrations:** run them once, not once per server, and keep them backward compatible so that rolling the code back is safe.
- **A failed deploy keeps its folder** (for debugging). Release numbers are never reused.
- The `unarchive` module needs `tar` on the server (GNU tar for `.tar.gz`) or `unzip` for `.zip`.

## Tested

Run against a local directory with Ansible core 2.21: five deploys with pruning, a failed health check (automatic
rollback), rollback to the previous and to a chosen release, rollback to a missing release (clear error), and a missing
shared file (clear error). The artifact in that run was a zip.

MIT licensed, see [LICENSE](LICENSE).
