# Start redis on demand, only after a successful role assumption

## Context

Two problems with redis today:
1. The container was created with `--restart always`, so Docker auto-restarts it on every daemon/WSL
   start — it runs even when `assume-role` hasn't been used for days.
2. `start-redis` is called at the very top of `assume-role` (line 81), before any credentials exist —
   so it starts even when the run has no valid AWS credentials (a failed or aborted assume).

Goal: redis (still opt-in via `ASSUME_ROLE_REDIS`) starts only when we actually obtained valid
credentials, comes up on demand, and never auto-restarts on boot. The old container/network/volume were
already removed by the user, so `start-redis` recreates them fresh on the next successful opt-in run.

"Valid credentials" = the role assumption succeeded, i.e. `ROLE_SESSION != "fail"` (set at line 408).

## Change 1 — `start-redis()`: on-demand policy + start a stopped container

Container-exists block (lines 47-49):
```bash
  if docker container inspect assume-role-redis >/dev/null 2>&1
  then return
  fi
```
becomes (start it if it exists but is stopped; `docker start` on a running container is a no-op):
```bash
  if docker container inspect assume-role-redis >/dev/null 2>&1
  then docker start assume-role-redis >/dev/null 2>&1; return
  fi
```

`docker run` line 51 — change `--restart always` to `--restart no`:
```bash
  docker run --name assume-role-redis --network assume-role-redis-network -v "assume-role-redis-volume:/data" --restart no -d redis:7.2 redis-server --notify-keyspace-events KEA >/dev/null 2>&1
```

## Change 2 — `assume-role()`: remove the early start (line 81)

Delete the `start-redis` call (and its trailing blank line) that sits right after the `aws` precondition
check, keeping the session-reuse block that follows it:
```bash
  fi

  start-redis          # <-- remove this line

  # reuse saved 12h MFA session when the shell started clean (avoids re-prompting MFA)
```

## Change 3 — `assume-role()`: start + write redis only on success

Wrap the `set-redis` block (lines 536-552) in a success guard and run `start-redis` inside it, so redis
is started and written only when we have valid credentials (indent the 17 `set-redis` lines by two
spaces):
```bash
  # store the session in redis (opt-in) only after a successful role assumption
  if [ "$ROLE_SESSION" != "fail" ]; then
    start-redis
    set-redis "${AWS_ACCOUNT_NAME}:AWS_REGION" $AWS_REGION
    # … the remaining 16 set-redis lines, unchanged except indentation …
    set-redis "${AWS_ACCOUNT_NAME}:AWS_STS_ROLE_ARN" $role_arn
  fi
```
On a failed assume (`ROLE_SESSION == "fail"`), redis is neither started nor written — which also avoids
`docker exec` errors against a container that was never started.

The explicit `assume-role start-redis` and `init` subcommands (lines 572, 576) are unchanged — those are
deliberate manual invocations and still honor the `ASSUME_ROLE_REDIS` flag via `start-redis` itself.

## Verification

1. Syntax: `bash -n assume-role` → clean.
2. Offline with a docker stub logging calls; drive `start-redis` and the success guard directly:
   ```bash
   d="$CLAUDE_JOB_DIR/tmp"; export probe="$d/calls.log"
   # a) create path uses --restart no
   : > "$probe"
   ( source ./assume-role; export ASSUME_ROLE_REDIS=1
     docker(){ echo "docker $*" >> "$probe"; [ "$1 $2" = "container inspect" ] && return 1; return 0; }
     start-redis )
   grep -q -- '--restart no' "$probe" && ! grep -q -- '--restart always' "$probe" && echo "OK: --restart no"
   # b) existing container is started
   : > "$probe"
   ( source ./assume-role; export ASSUME_ROLE_REDIS=1
     docker(){ echo "docker $*" >> "$probe"; return 0; }
     start-redis )
   grep -q '^docker start assume-role-redis' "$probe" && echo "OK: starts existing container"
   # c) failed assume -> redis never touched
   : > "$probe"
   ( source ./assume-role; export ASSUME_ROLE_REDIS=1; ROLE_SESSION=fail
     docker(){ echo "docker $*" >> "$probe"; return 0; }
     [ "$ROLE_SESSION" != "fail" ] && start-redis
     [ -s "$probe" ] && echo "FAIL: docker called" || echo "OK: no docker on failed assume" )
   ```
3. End-to-end: `ASSUME_ROLE_REDIS=1 assume-role <acct>` → container created with
   `docker inspect -f '{{.HostConfig.RestartPolicy.Name}}' assume-role-redis` = `no`; after a WSL/Docker
   restart it is `exited` and stays down; the next `ASSUME_ROLE_REDIS=1 assume-role <acct>` starts it. A
   run that fails to assume leaves no redis container.
