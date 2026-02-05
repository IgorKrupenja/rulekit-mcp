---
appliesTo:
  techs:
    - nestjs
description: NestJS-specific code quality rules and best practices
---

## General rules

### Package manager and scripts

- Use `pnpm`, not `npm` or `yarn`.
- Script naming (non-watch focus): use `*:check` for read-only checks (e.g. `format:check`, `lint:check`); use `test:run` and `test:e2e:run` for non-watch test runs; plain `test` is watch by default.

### Imports

- For imports, `.js` extensions should be appended, e.g.: `import { config } from './config.js';`
- Avoid relative imports that traverse directories; only same-folder relative imports are allowed (use path aliases otherwise).

### Config

- Always inject config via `@Inject(configFactory.KEY)` and `ConfigType<typeof configFactory>` instead of accessing config directly.

### Swagger

- Always use specific Swagger decorators like `ApiOkResponse` and `ApiNotFoundResponse` instead of `ApiResponse`.
- When using polymorphic response DTOs in Swagger (`oneOf`, `anyOf`, custom decorators), register all referenced models with `@ApiExtraModels(...)` so they appear in the schema.

### Prisma and database

- For atomic operations, wrap related Prisma writes in a transaction.
- When adding new SELECT-heavy queries, add or adjust DB indexes if needed, but first check existing indexes to avoid duplicates.
- After making changes in Prisma, suggest to run `prisma:generate`, `prisma:seed` and/or `prisma:migrate` scripts. Depending on what is necessary.

### DTOs and serialization

- DTO and interface properties should be marked as `readonly` if they are not intended to be modified.
- With `ClassSerializerInterceptor` and `excludeExtraneousValues` enabled, DTOs should explicitly use `@Expose()` on properties meant to be returned.
- When creating DTOs that are subsets of existing DTOs, use `PickType` (or similar mapped types) instead of duplicating fields.

### Testing

- Use `vitest`, not `jest`.
- `e2e` tests: if you write a test that actually modifies anything in DB, it makes sense to check if DB was actually modified. **Not** only checking what is e.g. returned by endpoint.

## Bootstrapping with nestjs-starter

- When asked to bootstrap or create a NestJS project, use the starter repo: `https://github.com/IgorKrupenja/nestjs-starter`.
- Before cloning, check the current project structure:
  - If the workspace is a monorepo and the target folder is empty, clone the starter into that folder and remove its `.git` directory.
  - If the workspace is not empty or the repo is not clearly a monorepo, suggest a subfolder (default to `/api`). If the user agrees, clone into that subfolder and remove the starter `.git`.
- GitHub Actions handling:
  - Single-project repo: keep the starter `.github` as-is.
  - Monorepo (API in subfolder): move the starter `.github` contents to the repo root and adapt actions to be monorepo-aware. Example composite action (store as `.github/actions/pnpm-install/action.yml`):
    - `inputs.working-directory` is required and used by `package_json_file`, `node-version-file`, `cache-dependency-path`, and `working-directory`.
    - Example snippet:

      ```yml
      name: pnpm-install
      inputs:
        working-directory:
          required: true
      runs:
        using: composite
        steps:
          - uses: pnpm/action-setup@v4
            with:
              package_json_file: ${{ inputs.working-directory }}/package.json
          - uses: actions/setup-node@v4
            with:
              node-version-file: ${{ inputs.working-directory }}/package.json
              cache: pnpm
              cache-dependency-path: ${{ inputs.working-directory }}/pnpm-lock.yaml
          - run: pnpm install --frozen-lockfile
            working-directory: ${{ inputs.working-directory }}
            shell: bash
      ```

    - Example workflow usage (`.github/workflows/api-run-checks.yml`):

      ```yml
      jobs:
        api-checks:
          runs-on: ubuntu-latest
          steps:
            - uses: actions/checkout@v4
            - uses: ./.github/actions/pnpm-install
              with:
                working-directory: api
            - run: pnpm lint
              working-directory: api
      ```

    - Update all steps to use the API subfolder for `node-version-file`, `cache-dependency-path`, install, lint, typecheck, test, etc.

  - If the monorepo already has shared GitHub Actions and workflows (e.g., front-end checks), only add the API-specific workflow and/or update existing composite actions to support the API subfolder.
  - Final step: ask if the user wants to remove the example Post module/service/controller. If yes, delete everything under `src/post/` and remove the related e2e test `test/post.e2e-spec.ts`.
