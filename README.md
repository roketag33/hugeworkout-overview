# HugeWorkout

Coaching platform that closes the gap between coaches and athletes. Coaches build training
programs, athletes run them from a mobile app that works with no network.

This repository holds the architecture write-up. **The source code is private.**

## The constraint that shaped everything

An athlete trains in a basement gym with no signal. The app has to be fully usable offline
and reconcile itself when the connection comes back. That single constraint drove most of
the decisions below.

## Architecture

A **TypeScript monorepo**, organised into scope-based libraries so dependency rules are
enforced rather than agreed upon.

```
apps/
  api/                 NestJS backend
  mobile/              Expo React Native app
  web/                 Next.js front
libs/
  api/                 backend modules (auth, users…)
  mobile/              mobile features
  shared/              DTOs, Zod schemas, utilities
  data-access-db/      Prisma schema and client
```

**API** — NestJS, PostgreSQL, Prisma, JWT with Passport strategies, dockerised services.

**Mobile** — React Native with Expo, **WatermelonDB** for the local database, a custom
design system (themes and tokens), React Navigation.

**Web** — Next.js 16, React 19, Tailwind, Zustand, Stripe for payments, Sentry, next-intl
for internationalisation.

## Three decisions worth explaining

**Shared contracts, single source of truth.** API contracts are Zod schemas living in
`libs/shared`. The API uses them in its validation pipes, the mobile app uses the same
objects for form validation. A contract change breaks both sides at compile time instead
of failing silently in production.

**Offline-first, not offline-tolerant.** The mobile app reads and writes to WatermelonDB
and never waits on the network. Synchronisation is a separate concern that reconciles
later. It is harder to build than online-first with a cache, and it is the only honest
answer to the basement gym.

**Boundaries enforced by tooling.** ESLint module boundary rules encode who may import
what: `shared` is importable by anyone, `api` code can never be imported by `mobile`.
Architecture that depends on discipline erodes; architecture that fails the build holds.

## Current state

The sync engine is a work in progress.

<!-- À COMPLÉTER PAR ALEXANDRE : où en est le produit, ce qui tourne déjà,
ce qui reste. Et une ou deux captures d'écran feraient beaucoup de bien ici. -->

---

Built by [Alexandre Sarrazin](https://www.alexandresarrazin.fr/fr).
