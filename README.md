# CSMix

A competitive Counter-Strike platform, in the shape of GamersClub or FACEIT:
players sign in with Steam, queue alone or as a party, get matched into two teams
of five, veto maps, play on a server the platform provisions for them, and come
out with a rating that moved.

This repository holds no application code. Each service lives in its own
repository and is pinned here as a submodule, so that one clone gets the whole
platform at a set of revisions that were known to go together.

## The services

| Path | Repository | Owns | Stack |
|---|---|---|---|
| [`accounts`](accounts) | [CSMix.Accounts](https://github.com/tiago-bitten/CSMix.Accounts) | identity, Steam sign-in, tokens, bans | .NET 10 |
| [`bff`](bff) | [CSMix.BFF](https://github.com/tiago-bitten/CSMix.BFF) | the only service a browser talks to | Go |
| [`matchmaking`](matchmaking) | [CSMix.Matchmaking](https://github.com/tiago-bitten/CSMix.Matchmaking) | parties, queues, forming a match, getting it accepted | Go |
| [`matches`](matches) | [CSMix.Matches](https://github.com/tiago-bitten/CSMix.Matches) | the match itself, from accepted to finished | Go |
| [`gameservers`](gameservers) | [CSMix.GameServers](https://github.com/tiago-bitten/CSMix.GameServers) | the machines and the server processes on them | Go |
| [`rating`](rating) | [CSMix.Rating](https://github.com/tiago-bitten/CSMix.Rating) | rating, seasons, leaderboards | Go |

## How they fit together

```
                    browser
                       │  cookie, never a token
                       ▼
                      BFF ──────────────┐
                       │                │
      X-Api-Key + Bearer                │
                       ▼                ▼
                   Accounts        Matchmaking
                                        │
                                        │ MatchReadyToCreate
                                        ▼
                                     Matches ──HTTP──▶ GameServers
                                        │
                                        │ MatchFinished
                                        ▼
                                      Rating
```

Two rules hold the seams together:

- **Accounts is the only issuer of tokens.** It signs with a P-256 private key
  that never leaves it; everyone else verifies with the public half from its
  JWKS. Handing a service the means to verify a token does not hand it the means
  to mint one.
- **Every service-to-service call carries `X-Api-Key`**, and a bearer token as
  well when the call is on a player's behalf. A service that calls another on
  behalf of a player forwards the token it was given — it never mints a new one,
  because it cannot.

The arrows are not all the same kind. `Matches` asks `GameServers` for a server
over **HTTP**, because a match blocks on the answer. `MatchReadyToCreate` and
`MatchFinished` go over **Kafka**, because nothing blocks on them and both sides
should survive the other being down.

## How every service is put together

The Go services share one layout, taken from `api-adm` and kept deliberately
plain:

- **`internal/<slice>/`** — a feature is a folder, not a layer. Inside one, four
  folders appear as they earn it: `api`, `app`, `domain`, `infra`. They are not
  created empty.
- **`internal/shared/`** — config, logging, the inbound edge. Duplicated per
  service rather than shared as a library: `internal/` could not export it
  anyway, and a common module would couple the deploys together.
- **No `pkg/` and no framework.** `net/http` has had method and path routing
  since Go 1.22.

`CSMix.Accounts` is .NET and follows its own rules, written down in
[its `docs/`](https://github.com/tiago-bitten/CSMix.Accounts/tree/main/docs) —
hexagonal, CQRS without MediatR, and a numbered decision record explaining why
each rule exists.

## Cloning

```bash
git clone --recurse-submodules https://github.com/tiago-bitten/CSMix.git
```

Already cloned without them:

```bash
git submodule update --init --recursive
```

## Working with the submodules

Each submodule tracks the `main` branch of its repository. Pull the latest commit
of every service:

```bash
git submodule update --remote
```

That moves the pointers in the working tree; commit them here so this repository
records the new revisions:

```bash
git commit -am "chore: bump service pointers"
```

Changes to a service are made and pushed in its own repository. The pointer here
is updated afterwards, and that is the point: this repository answers "which
revisions were known to work together", which is a question no individual
service can answer about itself.

## Adding a service

```bash
git submodule add -b main https://github.com/tiago-bitten/<repo>.git <path>
```

Then add a row to the table above.

## Where this is

`Accounts` is built and has no consumer yet. `BFF` is built against it and
verified end to end against a stub. The other four are skeletons: layout,
configuration that refuses to start when something is missing, a health endpoint
and a graceful shutdown, and nothing else.

The order they get built in is `Matchmaking`, `Matches`, `GameServers`,
`Rating` — and `Rating` can arrive last, because nothing blocks on it and a
platform where nobody has a rating yet still runs matches.
