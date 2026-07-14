# wtdev

One dev server per git worktree. No port collisions, stable URLs, and a live dashboard of every checkout.

If you run coding agents (Claude Code, Cursor, etc.) in parallel git worktrees, you've hit this: every worktree's dev server wants port 3000, agents kill each other's servers or drift to random ports, and you can never remember which branch is on which port.

`wtdev` fixes it with one rule: **the port is a pure function of the worktree's path.**

- main checkout → `3000`
- every linked worktree → a stable port in `3001–3999`, hashed from its path

Same worktree, same port, every time — across restarts, across agents, with zero coordination and zero state. In the rare case two worktrees hash to the same port, the later one (in `git worktree list` order) simply takes the next free port, so ports stay unique without anything to configure.

## Install

```sh
curl -fsSL https://raw.githubusercontent.com/Dave-56/wtdev/main/wtdev -o /usr/local/bin/wtdev
chmod +x /usr/local/bin/wtdev
```

It's a single dependency-free POSIX shell script; you can also just copy it into your repo.

## Usage

```sh
wtdev run                 # start the dev server on this checkout's port
wtdev run pnpm dev        # ...with an explicit command ({port} is substituted)
wtdev up                  # start it in the background if not already serving
wtdev port                # print this checkout's port
wtdev list                # every worktree with its branch + port
wtdev dashboard           # generate a live-status HTML dashboard
```

The easiest setup is to route your dev script through it, so agents and humans alike get the right port without thinking:

```json
{ "scripts": { "dev": "wtdev run next dev -p {port}" } }
```

`wtdev run` exports `PORT` (which Next.js, Vite via config, CRA, Remix, and most Node servers respect) and also substitutes `{port}` into the command for tools that want a flag.

It also copies gitignored env files (`.env`, `.env.local` by default) from the main checkout into fresh worktrees, since those never come along with `git worktree add`.

`wtdev up` is `run` for automation: it starts the server in the background (logging to `.wtdev.log`) only if the checkout's port isn't already serving, and prints the URL either way. Wire it into an agent's session-start hook and every session begins with its dev URL ready:

```json
{ "hooks": { "SessionStart": [ { "hooks": [ { "type": "command", "command": "sh scripts/wtdev up" } ] } ] } }
```

## Dashboard

```sh
wtdev dashboard
```

Writes `worktrees.html` (into `public/` if you have one, so your dev server serves it) listing every worktree's branch, last commit, and URL, with a green/red live-status dot probed from the page every 5 seconds. Re-run it when you add or remove worktrees.

## Pretty URLs

If [localias](https://github.com/peterldowns/localias) is installed and running (`brew install peterldowns/localias/localias && localias start`), `wtdev run` also registers `http://<worktree-name>.localhost` for each worktree. `my-feature.localhost` beats `localhost:3417`.

## Config

| Env var | Default | |
|---|---|---|
| `PORT` | — | overrides everything |
| `WTDEV_BASE_PORT` | `3000` | port for the main checkout |
| `WTDEV_PORT_RANGE` | `999` | worktree ports span `base+1 … base+range` |
| `WTDEV_CMD` | `npm run dev` | dev command template for `wtdev run` |
| `WTDEV_ENV_FILES` | `.env .env.local` | env files borrowed from the main checkout |

## How it works

There's no daemon, no lockfile, no registry. The worktree path is hashed with `cksum` and mapped into the port range. Worktrees are discovered with `git worktree list`, so any layout works (including Claude Code's `.claude/worktrees/`). Collisions are possible in principle (999 slots), astronomically unlikely with a handful of worktrees — and if you ever hit one, `PORT` wins.

## License

MIT
