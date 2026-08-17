# CSMix

Umbrella repository for the CSMix platform. It holds no application code of its own — each
service lives in its own repository and is linked here as a git submodule, pinned to a commit.

## Services

| Path | Repository | What it does |
|---|---|---|
| [`accounts`](accounts) | [tiago-bitten/CSMix.Accounts](https://github.com/tiago-bitten/CSMix.Accounts) | Accounts service — Steam-first identity, users and their linked Steam profiles. .NET 10, hexagonal, CQRS. |

Shared building blocks (typed ids, `Result`, CQRS contracts, domain events, specifications,
cursor pagination) come from [i26](https://github.com/tiago-bitten/i26), consumed as NuGet
packages rather than as a submodule.

## Cloning

```bash
git clone --recurse-submodules https://github.com/tiago-bitten/CSMix.git
```

Already cloned without the submodules:

```bash
git submodule update --init --recursive
```

## Working with the submodules

Each submodule tracks the `main` branch of its repository. Pull the latest commit of every
service:

```bash
git submodule update --remote
```

That moves the pointers in the working tree; commit them here so the umbrella records the new
revisions:

```bash
git commit -am "chore: bump service pointers"
```

Changes to a service are made and pushed inside its own repository — the pointer in this
repository is updated afterwards.

## Adding a service

```bash
git submodule add -b main https://github.com/tiago-bitten/<repo>.git <path>
```

Then add a row to the table above.
