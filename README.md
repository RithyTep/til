# Today I Learned (TIL)

<div align="center">

<img src="https://upload.wikimedia.org/wikipedia/commons/thumb/9/91/Octicons-mark-github.svg/2048px-Octicons-mark-github.svg.png" width="100" alt="TIL">

![TIL](https://img.shields.io/badge/TIL-Today%20I%20Learned-6366F1?style=for-the-badge)
![Articles](https://img.shields.io/badge/Articles-80+-success?style=for-the-badge)
![Updated](https://img.shields.io/badge/Updated-December%202025-blue?style=for-the-badge)

**A curated collection of concise write-ups on technologies I learn daily.**

[![Portfolio](https://img.shields.io/badge/Portfolio-rithytep.online-blue?style=flat-square)](https://portfolio.rithytep.online/)
[![GitHub](https://img.shields.io/badge/GitHub-RithyTep-black?style=flat-square&logo=github)](https://github.com/RithyTep)

</div>

---

## Quick Navigation

| Frontend | Backend | DevOps | AI/ML | Game Dev |
|:--------:|:-------:|:------:|:-----:|:--------:|
| [React](#react) | [API](#api) | [DevOps](#devops--platform) | [AI/ML](#aiml-engineering) | [WebGL](#game-development) |
| [Next.js](#nextjs) | [Database](#database-engineering) | [Monorepo](#monorepo) | [LLM](#aiml-engineering) | [Physics](#game-development) |
| [TypeScript](#typescript) | [Auth](#authentication) | [Testing](#testing) | [RAG](#aiml-engineering) | [Multiplayer](#game-development) |
| [CSS](#css) | [State](#state-management) | [Bundlers](#bundlers) | [Agents](#aiml-engineering) | [Neural AI](#game-development) |

---

## Categories

### Frontend

<details>
<summary><b>React</b> (2 articles)</summary>

- [React Compiler - Automatic Optimization](react/react-compiler.md) - No more useMemo, useCallback
- [React 19 Features - Compiler, Actions, and New Hooks](nextjs/react-19-features.md)

</details>

<details>
<summary><b>Next.js</b> (17 articles)</summary>

**Core Concepts**
- [Server Components - Zero-bundle React](nextjs/server-components.md)
- [Server Actions - Form handling without API routes](nextjs/server-actions.md)
- [Route Handlers - API endpoints in App Router](nextjs/route-handlers.md)
- [Middleware - Request/response interception](nextjs/middleware.md)
- [Turbopack - Rust-Powered Bundler](nextjs/turbopack.md)

**Advanced Routing**
- [Parallel Routes - Multiple pages in one layout](nextjs/parallel-routes.md)
- [Intercepting Routes - Modal patterns](nextjs/intercepting-routes.md)

**Performance**
- [Streaming & Suspense - Progressive rendering](nextjs/streaming-suspense.md)
- [Caching Strategies - Full Stack caching](nextjs/caching-strategies.md)
- [Partial Prerendering (PPR) - Static + Dynamic](nextjs/partial-prerendering.md)
- [Dynamic Imports - Code splitting](nextjs/dynamic-imports.md)
- [Image Optimization - next/image deep dive](nextjs/image-optimization.md)

**Features**
- [Data Fetching Patterns - Server-first approach](nextjs/data-fetching-patterns.md)
- [Metadata API - SEO optimization](nextjs/metadata-seo.md)
- [Authentication Patterns - Auth in App Router](nextjs/authentication-patterns.md)
- [Internationalization - Multi-language apps](nextjs/internationalization.md)
- [Error Handling - error.tsx and recovery](nextjs/error-handling.md)

</details>

<details>
<summary><b>TypeScript</b> (6 articles)</summary>

- [Zod v4 - 14x Faster Schema Validation](typescript/zod-v4.md)
- [Satisfies Operator - Better type inference](typescript/satisfies-operator.md)
- [Branded Types - Type-safe IDs](typescript/branded-types.md)
- [Template Literal Types - String patterns](typescript/template-literal-types.md)
- [Infer Keyword - Conditional types](typescript/infer-keyword.md)
- [Const Assertions - `as const`](typescript/const-assertions.md)

</details>

<details>
<summary><b>CSS</b> (4 articles)</summary>

- [Tailwind CSS v4 - CSS-First Configuration](css/tailwind-v4.md)
- [View Transitions API - Smooth Page Animations](css/view-transitions.md)
- [Container Queries - Component-based styling](css/container-queries.md)
- [:has() Selector - Parent selection](css/has-selector.md)

</details>

<details>
<summary><b>JavaScript</b> (3 articles)</summary>

- [Nullish Coalescing - `??` vs `||`](javascript/nullish-coalescing.md)
- [Object.groupBy - Native array grouping](javascript/object-groupby.md)
- [AbortController - Fetch cancellation](javascript/abort-controller.md)

</details>

<details>
<summary><b>Angular</b> (3 articles)</summary>

- [Standalone Components - Without NgModules](angular/standalone-components.md)
- [Signals - Reactive state management](angular/signals.md)
- [Defer Blocks - Lazy loading](angular/defer-blocks.md)

</details>

---

### Backend & Data

<details>
<summary><b>API</b> (1 article)</summary>

- [tRPC v11 - End-to-End Type-Safe APIs](api/trpc-v11.md)

</details>

<details>
<summary><b>Database Engineering</b> (5 articles)</summary>

- [Drizzle ORM - Type-Safe SQL](database/drizzle-orm.md)
- [Turso - SQLite at the Edge](database/turso.md)
- [Supabase Realtime - Live Data Subscriptions](database/supabase-realtime.md)
- [Query Optimization - EXPLAIN, Indexes, CTEs](database/query-optimization.md)
- [Distributed Transactions - 2PC, Sagas, Outbox](database/distributed-transactions.md)

</details>

<details>
<summary><b>Authentication</b> (2 articles)</summary>

- [Clerk - Modern Authentication for React](auth/clerk.md)
- [Passkeys - Passwordless WebAuthn](auth/passkeys.md)

</details>

<details>
<summary><b>State Management</b> (2 articles)</summary>

- [Zustand - Lightweight State Management](state/zustand.md)
- [TanStack Query v5 - Server State Management](state/tanstack-query.md)

</details>

<details>
<summary><b>Frameworks</b> (2 articles)</summary>

- [Hono - Ultrafast Web Framework for Edge](frameworks/hono.md)
- [TanStack Router - Type-Safe Routing](tanstack/tanstack-router.md)

</details>

---

### DevOps & Tools

<details>
<summary><b>DevOps & Platform</b> (5 articles)</summary>

- [Kubernetes Patterns - Deployments, HPA, Service Mesh](devops/kubernetes-patterns.md)
- [GitOps with ArgoCD - ApplicationSets](devops/gitops-argocd.md)
- [Observability Stack - OpenTelemetry, Prometheus](devops/observability-stack.md)
- [JavaScript Runtimes 2025 - Node.js vs Deno vs Bun](devops/javascript-runtimes-2025.md)
- [Edge Runtime 2025 - Cloudflare vs Vercel](devops/edge-runtime-2025.md)

</details>

<details>
<summary><b>Runtimes</b> (1 article)</summary>

- [Bun - All-in-One JavaScript Runtime](runtimes/bun.md)

</details>

<details>
<summary><b>Bundlers</b> (3 articles)</summary>

- [Rspack - 23x Faster Webpack Alternative](bundlers/rspack.md)
- [Vite 6 - Environment API](bundlers/vite6.md)
- [Turbopack - Next.js Rust Bundler](nextjs/turbopack.md)

</details>

<details>
<summary><b>Testing</b> (2 articles)</summary>

- [Playwright - End-to-End Testing](testing/playwright.md)
- [Vitest - Blazing Fast Unit Testing](testing/vitest.md)

</details>

<details>
<summary><b>Monorepo</b> (1 article)</summary>

- [Turborepo - High-Performance Monorepo](monorepo/turborepo.md)

</details>

<details>
<summary><b>Tools</b> (1 article)</summary>

- [Biome - Fast Linter & Formatter](tools/biome.md)

</details>

<details>
<summary><b>Git</b> (2 articles)</summary>

- [Interactive Rebase - Cleaner history](git/interactive-rebase.md)
- [Git Worktrees - Parallel development](git/worktrees.md)

</details>

---

### Architecture & Systems

<details>
<summary><b>Architecture</b> (3 articles)</summary>

- [Event-Driven Architecture - CQRS, Sagas, Outbox](architecture/event-driven-architecture.md)
- [Domain-Driven Design - Aggregates, Bounded Contexts](architecture/domain-driven-design.md)
- [Hexagonal Architecture - Ports & Adapters](architecture/hexagonal-architecture.md)

</details>

<details>
<summary><b>Distributed Systems</b> (2 articles)</summary>

- [Consensus Protocols - Raft, Paxos, PBFT](distributed-systems/consensus-protocols.md)
- [CAP Theorem & PACELC - Consistency models](distributed-systems/cap-theorem.md)

</details>

<details>
<summary><b>Security</b> (2 articles)</summary>

- [API Security - JWT, Rate Limiting, Validation](security/api-security.md)
- [Zero Trust Architecture - mTLS, SPIFFE](security/zero-trust.md)

</details>

<details>
<summary><b>Performance</b> (1 article)</summary>

- [System Design & Scaling - Sharding, Caching, Queues](performance/system-design-scaling.md)

</details>

<details>
<summary><b>Observability</b> (1 article)</summary>

- [Distributed Tracing - OpenTelemetry](observability/distributed-tracing.md)

</details>

---

### Game Development

<details>
<summary><b>Game Dev</b> (1 article)</summary>

- [WebGL Game Architecture - Building Browser Games](gamedev/webgl-game-architecture.md) - WebGL, Physics, Multiplayer, Neural AI

**Topics Covered:**
- WebGL 2.0 rendering & isometric projection
- Car physics (Marco Monster approach)
- Quad-tree collision detection
- Neural network AI with genetic algorithms
- Client-side prediction for multiplayer
- Bezier curve track generation

</details>

---

### AI/ML Engineering

<details>
<summary><b>AI/ML</b> (10 articles)</summary>

**APIs & SDKs**
- [Claude API - Building with Anthropic's AI](ai-ml/claude-api.md)
- [Vercel AI SDK 4.1 - Production AI Apps](ai-ml/vercel-ai-sdk.md)

**Patterns**
- [RAG Patterns - Retrieval Augmented Generation](ai-ml/rag-patterns.md)
- [LLM Integration - Function Calling, Streaming](ai-ml/llm-integration.md)
- [Prompt Engineering - CoT, Few-Shot](ai-ml/prompt-engineering.md)

**Tools**
- [MCP - Model Context Protocol](ai-ml/mcp-model-context-protocol.md)
- [Context7 MCP - Real-Time Documentation](ai-ml/context7-mcp.md)
- [AI Coding Agents 2025 - Cursor vs Claude Code](ai-ml/ai-coding-agents-2025.md)
- [Cursor Rules - System Prompts for AI](ai-ml/cursor-rules.md)
- [Development Conventions - Team Guidelines](ai-ml/development-conventions.md)

</details>

---

## Recent Additions (December 2025)

| Article | Category | Description |
|---------|----------|-------------|
| [WebGL Game Architecture](gamedev/webgl-game-architecture.md) | Game Dev | Browser games with WebGL |
| [React Compiler](react/react-compiler.md) | React | Automatic memoization |
| [Tailwind v4](css/tailwind-v4.md) | CSS | CSS-first configuration |
| [View Transitions](css/view-transitions.md) | CSS | Native page animations |
| [tRPC v11](api/trpc-v11.md) | API | Type-safe APIs |
| [Drizzle ORM](database/drizzle-orm.md) | Database | Type-safe SQL |
| [Turso](database/turso.md) | Database | SQLite at edge |
| [Clerk](auth/clerk.md) | Auth | Modern authentication |
| [Passkeys](auth/passkeys.md) | Auth | WebAuthn |
| [Zustand](state/zustand.md) | State | Lightweight state |
| [TanStack Query](state/tanstack-query.md) | State | Server state |
| [Playwright](testing/playwright.md) | Testing | E2E testing |
| [Vitest](testing/vitest.md) | Testing | Unit testing |
| [Turborepo](monorepo/turborepo.md) | Monorepo | Build system |
| [Bun](runtimes/bun.md) | Runtime | All-in-one JS |
| [Rspack](bundlers/rspack.md) | Bundler | Rust webpack |
| [Vite 6](bundlers/vite6.md) | Bundler | Environment API |
| [Hono](frameworks/hono.md) | Framework | Edge framework |
| [TanStack Router](tanstack/tanstack-router.md) | Framework | Type-safe routing |
| [Biome](tools/biome.md) | Tools | Fast linter |
| [Claude API](ai-ml/claude-api.md) | AI/ML | Anthropic API |
| [RAG Patterns](ai-ml/rag-patterns.md) | AI/ML | Vector search |

---

## About

I'm **Rithy Tep**, a **Full Stack Developer** from Cambodia. I architect and build scalable systems using TypeScript, Next.js, and cloud-native technologies.

**Focus Areas:**
- System Architecture & Distributed Systems
- AI/ML Integration & LLM Applications
- Platform Engineering & DevOps

**Connect:**
- [Portfolio](https://portfolio.rithytep.online/)
- [GitHub](https://github.com/RithyTep)

---

## License

MIT - Feel free to use these learnings in your own projects!

<div align="center">

**Star this repo if you find it helpful!**

![Stars](https://img.shields.io/github/stars/RithyTep/til?style=social)

</div>
