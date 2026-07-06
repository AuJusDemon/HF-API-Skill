# HF API Skill

A reference file for building against the [HackForums API v2](https://apidocs.hackforums.net/), meant to be dropped into an AI coding agent's context (Claude, or anything else that reads a `SKILL.md`/`AGENT.md`-style file) so it can help you build bots, dashboards, trading tools, or anything else on top of the API without guessing.

## Why this exists

The official docs cover the basics: OAuth, scopes, the `/read` and `/write` model. They leave out a lot of the stuff you only find by actually building something. Which fields silently return nothing without the right scope. Which endpoints throw an undocumented error under certain call shapes. How Cloudflare's bot-check actually behaves. What the write-only contract actions even are, since they're not in the official docs at all.

This file collects everything found the hard way while building [HFToolbox](https://github.com/AuJusDemon/HFToolbox), an open-source, self-hosted, all-in-one HF dashboard, tested against live traffic, not guessed. The goal is that the next person building on the HF API doesn't have to rediscover any of it.

## Using it

Drop `SKILL.md` into your project, or point your AI agent at this repo directly. It covers:

- OAuth2 flow
- Every read/write endpoint, fields, and known quirks
- Hard safety rules for write actions that cost bytes or lock in a contract (sending bytes, bumping threads, approving contracts)
- Rate limits and error handling
- Batching rules and reusable call patterns

## Contributing

If you find something new while building against the live API, a field that behaves differently, an error case not covered here, open a PR. State what you tested (endpoint, scope, request shape) so it's reproducible, not just a guess.

## License

MIT.
