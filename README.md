# TypeScript Skills

A focused collection of reusable TypeScript development skills for AI coding agents, covering APIs, testing, tooling, Fastify, databases, Effect, and functional patterns.

## Install

```bash
npx skills add martinffx/typescript-skills
```

Install a specific skill:

```bash
npx skills add martinffx/typescript-skills@typescript-fastify
```

Install a released version:

```bash
npx skills add 'martinffx/typescript-skills#v0.1.0'
```

Install one skill from a released version:

```bash
npx skills add 'martinffx/typescript-skills#v0.1.0@typescript-fastify'
```

List the available skills:

```bash
npx skills add martinffx/typescript-skills --list
```

## Skills

| Skill | Focus |
| --- | --- |
| `typescript-api-design` | REST API conventions, errors, and pagination |
| `typescript-build-tools` | Bun or pnpm, TypeScript 7, Vitest, Biome or Oxc, and Turborepo |
| `typescript-dynamodb-toolbox` | DynamoDB Toolbox entities and repositories |
| `typescript-drizzle-orm` | Drizzle schemas, queries, and repositories |
| `typescript-effect-ts` | Effect services, schemas, errors, and resources |
| `typescript-fastify` | Fastify routes, validation, and plugins |
| `typescript-functional-patterns` | ADTs, branded types, and Option/Result |
| `typescript-testing` | Mocking, MSW, and snapshot testing |

Each skill lives in `skills/<skill-name>/SKILL.md` with supporting material in its `references/` directory.
