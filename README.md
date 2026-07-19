# NumberHive Documentation

This repository is the **central knowledge base** for the NumberHive product ecosystem — the single place where architecture, cross-repo processes, and shared conventions are documented so they stay in sync across every repo that makes up NumberHive.

## Why this repo exists

NumberHive is not a single application — it's a **collection of repositories** that together form one product, served across multiple subdomains. As the number of repos grows, so does the risk of architectural knowledge living only in one person's head, or in a Slack thread, or scattered across READMEs that drift out of date with each other.

This repo solves that by being the **one place** that documents:

- How the various NumberHive repos relate to and depend on each other
- Cross-repo events, contracts, and processes (e.g. things that happen in one repo that other repos need to know about or react to)
- Shared architectural decisions that apply across the ecosystem
- Which subdomain each part of the product is served from, and how they fit together

If it affects more than one repo, or needs to stay consistent across repos, it belongs here.

## The NumberHive ecosystem

NumberHive currently includes (and will continue to grow):

- **Educational game** — the learning-focused product experience
- **Public game** — the public-facing game experience
- **Administrative facilities** *(upcoming)* — internal/admin tooling for managing the platform

Each of these is expected to be served from its own subdomain under the NumberHive domain, while sharing underlying data, architecture, and processes. Other supporting and prototype repositories exist alongside these in the [NumberHive GitHub org](https://github.com/NumberHive) — this document will be expanded to map out which are active production services, which are prototypes, and how each one fits into the whole as that picture is confirmed.

## What belongs in this repo

- **Architecture** — system diagrams, service boundaries, data ownership, integration points between repos
- **Cross-repo processes** — workflows, events, or triggers that span more than one repo (e.g. "when X happens in the game repo, the admin repo needs to know about it")
- **Subdomain map** — which repo serves which subdomain, and how routing/deployment ties them together
- **Shared conventions** — naming, data formats, auth patterns, or other standards multiple repos need to agree on
- **Decision records** — why architectural choices were made, so the reasoning isn't lost

## What does *not* belong here

- Implementation detail specific to a single repo (that stays in that repo's own README/docs)
- Code — this is a documentation-only repository

## Status

This repo has just been established as the central documentation store. Content is being built out incrementally — expect this README and the structure around it to expand as architecture and cross-repo processes are captured.

## Contributing

When you make a change in any NumberHive repo that affects another repo, or that other developers need to understand to work across the ecosystem, document it here rather than (or in addition to) the originating repo.
