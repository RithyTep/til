# Today I Learned (TIL)

<div align="center">

<img src="https://raw.githubusercontent.com/sindresorhus/awesome/main/media/logo.svg" width="100" alt="TIL">

![TIL](https://img.shields.io/badge/TIL-Today%20I%20Learned-6366F1?style=for-the-badge)
![Articles](https://img.shields.io/badge/Articles-56-success?style=for-the-badge)

![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white)
![Next.js](https://img.shields.io/badge/Next.js-000000?style=for-the-badge&logo=next.js&logoColor=white)
![React](https://img.shields.io/badge/React-61DAFB?style=for-the-badge&logo=react&logoColor=black)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![AI](https://img.shields.io/badge/AI%2FML-FF6F00?style=for-the-badge&logo=tensorflow&logoColor=white)

**Advanced engineering concepts for senior developers and tech leads.**

[![Portfolio](https://img.shields.io/badge/Portfolio-rithytep.online-blue?style=flat-square)](https://portfolio.rithytep.online/)
[![GitHub](https://img.shields.io/badge/GitHub-RithyTep-black?style=flat-square&logo=github)](https://github.com/RithyTep)

</div>

---

## Categories

| Category | Count |
|----------|-------|
| [Architecture](#architecture) | 3 |
| [Distributed Systems](#distributed-systems) | 2 |
| [DevOps & Platform](#devops--platform) | 5 |
| [AI/ML Engineering](#aiml-engineering) | 8 |
| [Database Engineering](#database-engineering) | 2 |
| [Security Engineering](#security-engineering) | 2 |
| [Performance & Scaling](#performance--scaling) | 1 |
| [Observability](#observability) | 1 |
| [Next.js & React](#nextjs--react) | 17 |
| [TypeScript](#typescript) | 5 |
| [Angular](#angular) | 3 |
| [JavaScript](#javascript) | 3 |
| [Git](#git) | 2 |
| [CSS](#css) | 2 |

**Total: 56 articles**

---

## Architecture

System design patterns for large-scale applications.

- [Event-Driven Architecture - CQRS, Sagas, Outbox Pattern](architecture/event-driven-architecture.md)
- [Domain-Driven Design - Aggregates, Bounded Contexts, ACL](architecture/domain-driven-design.md)
- [Hexagonal Architecture - Ports & Adapters, Clean Architecture](architecture/hexagonal-architecture.md)

## Distributed Systems

Building reliable systems at scale.

- [Consensus Protocols - Raft, Paxos, PBFT implementation](distributed-systems/consensus-protocols.md)
- [CAP Theorem & PACELC - Consistency models, Quorums, CRDTs](distributed-systems/cap-theorem.md)

## DevOps & Platform

Production-grade infrastructure and deployment.

- [Kubernetes Patterns - Deployments, HPA, Service Mesh, Secrets](devops/kubernetes-patterns.md)
- [GitOps with ArgoCD - ApplicationSets, Progressive Delivery](devops/gitops-argocd.md)
- [Observability Stack - OpenTelemetry, Prometheus, Grafana](devops/observability-stack.md)
- [JavaScript Runtimes 2025 - Node.js vs Deno vs Bun](devops/javascript-runtimes-2025.md)
- [Edge Runtime 2025 - Cloudflare Workers vs Vercel Edge](devops/edge-runtime-2025.md)

## AI/ML Engineering

Building production AI systems.

- [LLM Integration - RAG, Function Calling, Streaming, Caching](ai-ml/llm-integration.md)
- [Prompt Engineering - Chain of Thought, Few-Shot, Constitutional AI](ai-ml/prompt-engineering.md)
- [MCP - Model Context Protocol for AI Tools](ai-ml/mcp-model-context-protocol.md)
- [Context7 MCP - Real-Time Documentation for AI Coding](ai-ml/context7-mcp.md)
- [AI Coding Agents 2025 - Cursor vs Claude Code vs Windsurf](ai-ml/ai-coding-agents-2025.md)
- [Vercel AI SDK 4.1 - Building Production AI Applications](ai-ml/vercel-ai-sdk.md)
- [Cursor Rules - System Prompts for AI Coding Assistants](ai-ml/cursor-rules.md)
- [Development Conventions - Team Guidelines and Standards](ai-ml/development-conventions.md)

## Database Engineering

Advanced data management patterns.

- [Query Optimization - EXPLAIN, Indexes, Window Functions, CTEs](database/query-optimization.md)
- [Distributed Transactions - 2PC, Sagas, Outbox, CRDTs](database/distributed-transactions.md)

## Security Engineering

Zero-trust and API security.

- [API Security - JWT, Rate Limiting, Input Validation, Secrets](security/api-security.md)
- [Zero Trust Architecture - mTLS, SPIFFE, Policy-Based Access](security/zero-trust.md)

## Performance & Scaling

System design for massive scale.

- [System Design & Scaling - Load Balancing, Sharding, Caching, Queues](performance/system-design-scaling.md)

## Observability

End-to-end system visibility.

- [Distributed Tracing - OpenTelemetry, Sampling, Trace Analysis](observability/distributed-tracing.md)

---

## Next.js & React

Deep dive into Next.js 15+ and React 19 features and patterns.

### React 19
- [React 19 Features - Compiler, Actions, and New Hooks](nextjs/react-19-features.md)

### Core Concepts
- [Server Components - Zero-bundle React](nextjs/server-components.md)
- [Server Actions - Form handling without API routes](nextjs/server-actions.md)
- [Route Handlers - API endpoints in App Router](nextjs/route-handlers.md)
- [Middleware - Request/response interception](nextjs/middleware.md)

### Advanced Routing
- [Parallel Routes - Multiple pages in one layout](nextjs/parallel-routes.md)
- [Intercepting Routes - Modal patterns](nextjs/intercepting-routes.md)

### Performance & Optimization
- [Streaming & Suspense - Progressive rendering](nextjs/streaming-suspense.md)
- [Caching Strategies - Full Stack caching](nextjs/caching-strategies.md)
- [Partial Prerendering (PPR) - Static + Dynamic](nextjs/partial-prerendering.md)
- [Dynamic Imports - Code splitting](nextjs/dynamic-imports.md)
- [Image Optimization - next/image deep dive](nextjs/image-optimization.md)

### Data & Fetching
- [Data Fetching Patterns - Server-first approach](nextjs/data-fetching-patterns.md)

### SEO & Metadata
- [Metadata API - SEO optimization](nextjs/metadata-seo.md)

### Authentication & i18n
- [Authentication Patterns - Auth in App Router](nextjs/authentication-patterns.md)
- [Internationalization - Multi-language apps](nextjs/internationalization.md)

### Error Handling
- [Error Handling - error.tsx and recovery](nextjs/error-handling.md)

## TypeScript

- [Use `satisfies` for better type inference](typescript/satisfies-operator.md)
- [Branded types for type-safe IDs](typescript/branded-types.md)
- [Template literal types for string patterns](typescript/template-literal-types.md)
- [Using `infer` in conditional types](typescript/infer-keyword.md)
- [Const assertions with `as const`](typescript/const-assertions.md)

## Angular

- [Standalone components without NgModules](angular/standalone-components.md)
- [Signals for reactive state management](angular/signals.md)
- [Defer blocks for lazy loading](angular/defer-blocks.md)

## JavaScript

- [Nullish coalescing vs OR operator](javascript/nullish-coalescing.md)
- [Object.groupBy for array grouping](javascript/object-groupby.md)
- [Using AbortController for fetch cancellation](javascript/abort-controller.md)

## Git

- [Interactive rebase for cleaner history](git/interactive-rebase.md)
- [Git worktrees for parallel development](git/worktrees.md)

## CSS

- [Container queries for component-based styling](css/container-queries.md)
- [The `:has()` selector for parent selection](css/has-selector.md)

---

## About

I'm **Rithy Tep**, a **Full Stack Team Lead** from Cambodia. I architect and build scalable systems using TypeScript, Next.js, Angular, and cloud-native technologies.

**Expertise:**
- System Architecture & Design
- Distributed Systems
- Platform Engineering
- AI/ML Integration
- Team Leadership

**Connect:**
- [Portfolio](https://portfolio.rithytep.online/)
- [GitHub](https://github.com/RithyTep)
- [Techbodia](https://github.com/Techbodia)

## License

MIT - Feel free to use these learnings in your own projects!

---

<div align="center">

**Star this repo** if you find it helpful!

</div>
