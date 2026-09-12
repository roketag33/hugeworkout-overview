# HugeWorkout

A platform connecting a strength coach with their athletes.

The coach builds a training program structured in weeks, days and exercise blocks, assigns it to clients or sells it from their own storefront, and follows what each person actually does. The athlete gets the program on their phone, works through the session set by set, logs loads and sees progression. Both sides talk through messaging with a weekly check-in. A social feed lets athletes publish sessions, follow each other and join challenges.

In production. The source is private; this page documents the architecture and the decisions behind it.

## Why it exists

Coaching tools and training tools tend to be two separate products. What is built around the coach relationship is weak once you are on the gym floor, and what is built for logging sets leaves the coach outside. HugeWorkout targets both at once.

The second constraint is the network. Gyms sit in basements, and an app that stalls on a spinner between two sets is unusable at exactly the moment it is needed. So the training path, and only the training path, runs off a real database on the phone: WatermelonDB in JSI mode with its own schema, its own migrations and the history queries, through a custom Expo prebuild plugin. Nine screens out of fifty-seven touch it. The rest of the app is online, deliberately, and the reasoning is below.

## Status

Live since August 2026 on three public domains: web, API and object storage. The stack sits behind a shared Traefik and is driven through Portainer. A push on the default branch runs the full verification suite, builds two images tagged by commit hash, pushes the compose file to the Portainer API and waits for the health endpoint to return the expected version. The server compiles nothing.

Android only, on the Play Store internal testing track. The first install from the store by someone outside the code was on 27 August 2026. No iOS build.

Remote push notifications are not wired yet: the mobile service only schedules local notifications for rest timers.

## Architecture

A Yarn workspaces monorepo, no Nx despite what the first commit message says.

```
apps/
  api/          NestJS shell. Entry point, root module, health probe,
                a handful of controllers. 3500 lines.
  web/          Next.js App Router, 65 routes.
  mobile/       Expo and React Native, 57 screens, React Navigation.
packages/
  core/         Pure TypeScript. Use cases, ports, domain services.
                No production dependency beyond tslib.
  shared/       DTOs, Zod schemas, the HTTP client shared by web and
                mobile, the common translation catalogue.
  ui/           Design system. Carries no text.
  data-access/  The mobile WatermelonDB layer.
  api/          Ten NestJS modules by domain, including the Prisma
                repositories.
```

The ratio between `apps/api` at 3500 lines and `packages/api` at 39 855 measures the rule: no business logic in the applications.

The browser never talks to the API directly. The Next middleware rewrites every `/api/*` call to the internal container address, so the API is served under the web origin, which is also what makes the OAuth callback work with a single public origin. Mobile calls the API host directly, baked into the build.

PostgreSQL is the only durable source of truth. Redis counts requests per client, with no persistence. MinIO stores media and is the only data service reachable publicly, because file URLs have to be openable by a browser and a phone. Neither Postgres nor Redis leaves the internal network, and no port is published on the host.

### One request, end to end

An athlete opens the social feed:

1. Traefik routes to the Next.js container. The page calls its own origin; the middleware rewrites to the API.
2. The throttler counts the call in Redis. The JWT guard validates the token and re-reads the account token generation from the database, one read per authenticated request, accepted so that signing out actually revokes sessions.
3. The controller validates the body with a Zod schema imported from `packages/shared`, the same schema the client used to validate its own form.
4. It calls a use case in `packages/core`, which knows only ports, meaning interfaces. A Prisma repository in `packages/api` implements the port.
5. The use case returns a DTO. Where a label will be displayed, the DTO carries a catalogue key, the numeric values separately, and the French sentence as a fallback. The client writes the words.

## Decisions

### The domain knows nothing about the database, the framework or the screen, and tooling enforces it

Three clients consume the same logic. A business rule written inside a NestJS controller is testable only inside NestJS and reusable nowhere.

So `packages/core` is framework-free TypeScript declaring ports, and the Prisma package implements them. That rule existed in the project documentation from the start and **nothing checked it**. The comment on the lint config says it plainly: the only thing stopping `core` from importing Prisma was that nobody had done it yet.

The ratchet was closed at the moment none of the three boundaries were being violated, which is the moment it cost nothing to close. Holding the boundary by convention and review had been tried for months, and the argument against it is not that it failed, it is that it could not say *when* it failed.

### Offline is a scope, not a mode

There is no synchronisation engine. `synchronize()` from WatermelonDB is called nowhere, and no server route speaks the sync protocol. What exists instead is two hand-written one-way mechanisms over a subset of the data.

**Sessions go up.** A finished session with an empty `pushed_at` is waiting in the queue; that column *is* the queue, there is no second table. The client sends each session whole to one endpoint, in sequence, and stops at the first failure. The server answers with its own id, which the client stores. Nothing comes back down into the local tables.

**The program comes down.** The server sends the full snapshot and the client diffs it locally against the six program tables in a single batch. Rows the snapshot keeps are updated, rows it drops are destroyed leaf-first so nothing is orphaned. No delta is requested from the server.

**The invariant that makes this work: every local table has exactly one writer.** The program is server-owned and the athlete never writes to it, so the local copy is a read cache that gets overwritten without condition. Sessions are client-owned and no code path ever pulls a session down. Routines sit in a read cache that only answers when the network has refused, and a successful read replaces it.

That is why there is no conflict resolution anywhere in this codebase. Not a merge policy, a structural property. Writing a generic bidirectional engine would have meant inventing conflicts in order to then resolve them.

Idempotency is the one place the two sides do meet, and it is handled at the seam. Each session carries a client-generated id, the server looks it up before inserting, and a replay returns the stored session without re-detecting personal records. The uniqueness key is the pair of athlete and client id rather than the client id alone, because two phones can draw the same local id.

A real bidirectional engine did exist as a spike, and it was removed in August 2026 for a security defect rather than a change of heart: the pull returned the whole table, and the push accepted a row id supplied by the client, so any authenticated caller could overwrite any row. A test was written at the moment of the removal so the endpoint cannot come back by accident. One Prisma model from that spike is still declared, with a comment saying why: dropping it would make Prisma generate a destructive migration against a production database.

### The server names the case, the client writes the sentence

Two clients, two i18n libraries, two languages to keep complete, and a server that was composing inflected French sentences. Take "3 sessions completed over 28 days": the plural and the word order are French rules, and no client can translate that sentence, it can only copy it.

A DTO now carries three fields instead: a key, the values, and the server's own sentence as a fallback. The fallback is what makes the migration safe, because a client that does not know the key yet displays exactly what it displayed before, so an already installed mobile app keeps working unchanged. Six guard suites enforce it, including one that renders keys in both languages to catch number formatting errors that key-existence checks cannot see.

The assumed exception: anything the server writes whole and nobody downstream can re-translate, an email or a CSV export, ships in a language the server picks.

### The deployed compose comes from the repository

The first automated deployment only pushed the image tag, out of caution, so as not to ship unrequested changes. The price was measured: three variables fixed in the repository, including the environment mode and the payment provider, never reached production. The workflow comment draws the conclusion. **A compose file fixed in the repo and never deployed is worse than a wrong one, because you believe it is applied.**

The workflow now pushes the whole file and rewrites only the tag. The stack is found by name rather than by a hard-coded id, because an id shifts the day a stack is recreated and would then point at a neighbour on the same server. Deployment is not considered successful until the health endpoint returns the expected version, because the old container answers "ok" until its last second.

### In CI, no fixed port and no fixed container name

The runner shares the Docker daemon with other stacks. Two real outages settled this. The end-to-end suite died on a port already allocated by a neighbouring stack. The next day it talked to a database container that was not its own, and was saved only by a password that did not match: with the right credentials, migrations and seeding would have run against another product's database.

The CI compose resets container names back to Compose and asks Docker for a free port, then the test code reads the assigned ports back and composes its connection strings from there. Every run uses a unique project name, so containers, network and volume are new and torn down at the end. Picking a different fixed port number was rejected in one line: it only moves the appointment, since the problem is not *which* port, it is imposing one at all.

### End-to-end tests also run against the built binaries

The dev e2e suite starts the frameworks in watch mode. Those are not the programs that get deployed. An i18n config file once passed a catalogue path as a parameter: the build succeeded, because all forty routes are dynamic and none is rendered at build time, and the production server returned a 500 on every page.

A second pass now builds both apps and replays the same Playwright journeys against them. It was added rather than substituted, because both modes have their own traps and only one of them ships. The dev pass had caught that particular defect, but only by luck: the server failed to start at all, where in production the same error would have surfaced as forty broken pages.

## What is solid, and what is not

Solid: the domain layer. 241 source files, 195 test suites, a coverage floor enforced in CI at 93 percent of lines and 84 percent of branches, and mutation testing on its services with a break threshold at 85.1 percent. It is the one layer where no framework catches a mistake, so it is the one that gets measured.

The CI gates: formatting, types, lint with zero warnings tolerated, the full Jest suite, dead code detection, the coverage floor, two Playwright passes, and incremental mutation testing. A repository that agrees to fail on dead code carries little silent debt.

The offline path is tested where it lives, at service level: 17 cases on the session queue covering the mark after acceptance, the stop at the first failure and two concurrent runs serialised, and 11 integration cases for the program replacement against a real local database, including one named after the failure it prevents, "does not destroy the offline program when the creation fails".

The repository also tests its own rules, not just behaviour: that a declared guard is actually attached, that a route with no caller is registered with a written reason, that no displayed text is hard-coded, that no key is missing from a language, that no `new Date()` sits in domain code. Several of those guards were born from a real defect, which is readable in their comments.

Not solid, and known:

- **The shared HTTP client is the knot of the project.** 2907 lines, about 197 methods, twelve domains in one class. It is the most connected node in the dependency graph at 115 edges, against 40 for the second. Any change to it touches web and mobile at once. The split is written up and costed, and not done.
- **The social Prisma repository is very large**, 6125 lines, more than twice the next file. It is filed as "watch if the feature keeps growing" rather than "split now", which is a decision rather than an oversight.
- **The mobile local database is not encrypted.** WatermelonDB in JSI mode opens a plain SQLite file and exposes no encryption option. This is worth stating because the project documentation claimed the opposite for months: it named SQLCipher, and it was wrong. The line was corrected, with an instruction not to write code that assumes the file is protected, and the real options are costed in a separate note. Still open.
- **The queue never fires on its own.** No network state listener, no timer, no background task. A session recorded with no signal waits until the athlete finishes another one or returns to the home screen. Connectivity can come back for hours with nothing leaving the phone.
- **Only sessions go up.** Routines, posts, comments, reactions, chat and check-ins all require the network at write time, with no queue and no replay. A second queue was considered and turned down as a second write path to maintain; the gap was closed for reading instead.
- **Three edges are known and open.** A discarded session never leaves the phone and is never purged, the local database only grows, and switching accounts wipes it including sessions still waiting.
- **The social feed does nothing offline.** No cache of any kind, so a post read five minutes earlier is gone.
- **No end-to-end offline test.** Nothing in the Playwright suites cuts the network; offline is verified at service level and never as a journey.
- **The coverage floor only covers the domain layer.** Web, mobile and the API modules have many tests and no threshold protecting them from a regression.
- **No iOS.** No build, no submission config, no Apple account.

## Numbers

182 865 lines of production code across 1064 files, 123 113 lines of tests across 679 files, 4988 declared test blocks. 61 Prisma models, 32 enums, 72 migrations. 47 controllers, 138 use case files, 44 Prisma repositories, 2 WebSocket gateways, 65 web routes, 57 mobile screens, 21 Playwright suites. 5508 translation keys per language, French and English at exact parity. 1261 commits.

## Stack

TypeScript 5.9 strict, Node 22, Yarn workspaces. Jest 30, Playwright 1.62, Stryker 9, ESLint 9, knip.

**API**: NestJS 11 on Express 5, Prisma 5 on PostgreSQL 15, JWT with Passport, Zod 4 validated per route through a custom pipe, helmet, throttler backed by ioredis, Stripe 20 with Connect, nodemailer, S3 SDK against MinIO, socket.io, Sentry.

**Web**: Next.js 16 App Router, React 19, Tailwind 4, next-intl with a cookie-carried locale, Zustand, framer-motion, dnd-kit for the program builder, Stripe React and Connect.

**Mobile**: Expo 54, React Native 0.81 new architecture, React Navigation 7, WatermelonDB 0.28 in JSI mode through a custom prebuild plugin, use-intl with FormatJS polyfills, FlashList 2, Reanimated 4, Detox.

---

Built by [Alexandre Sarrazin](https://www.alexandresarrazin.fr/fr).
