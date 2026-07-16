# no-nonsense-backend

Backend concepts explained the way I'd teach them to a colleague — no fluff, real code, real trade-offs.

---

## What this is

This isn't API documentation, and it isn't a copy-paste of my project READMEs. It's the
explanation I'd give someone sitting next to me, walking through why a piece of backend
code looks the way it does — the problem it solves, the naive approach that doesn't
scale, and the trade-offs I made instead of pretending there's one right answer.

Every chapter is built from a real, working project in my
[java-backend-portfolio](https://github.com/emmanuelgilmont/java-backend-portfolio) repo — not toy examples invented for the sake of a tutorial.

If you're preparing for a backend interview, ramping up on Spring Boot, or just curious
how someone with 25 years of Java under their belt thinks through these problems, this
is for you.

## How to read this

Two ways in, pick whichever fits what you're after:

**[`by-project/`](./by-project)** — one chapter per project, in a rough learning order
(simplest concepts first). Good if you want the full story behind a specific piece of
code, start to finish.

**`by-concept/`** *(coming as patterns repeat across projects)* — cross-cutting topics
like testing without Docker, or the cache-aside pattern, pulled together from wherever
they show up. Good if you already know which idea you want to understand, regardless of
which project it lives in.

Each chapter links directly to the exact source files it discusses, so you're never
reading about code you can't also go look at.

## Chapters

### By project

| # | Chapter | Status |
|---|---|---|
| 1 | [Why every log line should tell a story](./by-project/01-transaction-token-starter.md) | ✅ |
| 2 | [Why you cache, and what can go wrong when you do](./by-project/02-fx-rate-service.md) | ✅ |
| 3 | [What happens between two threads when you call a service](./by-project/03-grpc-price-service.md) | ✅ |
| 4 | Processing yesterday's trades, one chunk at a time | 🚧 |
| 5 | Finding a needle in a NAS full of documents | 🚧 |
| 6 | The anchor: why event-driven matters in finance | 🚧 |

### Bonus: side projects

| # | Chapter | Status |
|---|---|---|
| 7 | Talking to Discord from Java, for fun and automation | 🚧 |
| 8 | Consuming a public API without overthinking it | 🚧 |

### By concept

*Not started yet — these will emerge once enough project chapters exist to show real,
repeated patterns rather than an invented list.*

## A word on how this is written

Every chapter starts from the problem, not the solution. I try to show what I'd have
tried first, why it falls short, and only then land on the actual design — because
that's usually how understanding actually happens, not the other way around.

I also try to be honest about what I'm not sure of. If a design choice is debatable, or
I only found the issue after the fact, I'll say so rather than presenting the finished
code as if it was obvious from the start.

## Source code

The code behind every chapter lives in
[java-backend-portfolio](https://github.com/emmanuelgilmont/java-backend-portfolio),
a separate repo — this one is documentation only, no build files, no dependencies.

## License

This repository is licensed under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — reuse it, adapt it, share
it, just credit the source. See [LICENSE](./LICENSE) for the full text.

## About me

**Emmanuel Gilmont** — Freelance Java Backend Developer, Belgium.
25+ years of backend experience, currently available for contracts.

[LinkedIn](https://www.linkedin.com/in/emmanuelgilmont) |
[gate25.be](https://gate25.be) |
[code@gate25.be](mailto:code@gate25.be)
