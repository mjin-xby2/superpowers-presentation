# TSS-491 - Spec

> Status: Approved at Gate 1 on 2026-08-28.

## Problem

The SDK's `toManifest(definition)` returns an in-memory representation of an automation definition.
Authors have no supported command that imports compiled definitions, associates them with their Git
source, and writes portable manifest files. Each automation repository would otherwise need to
build its own import, Git-inference, and serialization script.

## Approach

Expose an `automation-sdk` binary with a `manifest` subcommand:

```text
automation-sdk manifest [--out-dir <dir>] [--quiet] <definition-file...>
```![alt text](image.png)

The command imports each explicit compiled ESM JavaScript module, examines its default export,
infers source coordinates from the Git checkout containing that module, calls `toManifest`, and
writes formatted JSON. Use Node 22 built-ins and add no dependencies.

## Package and SDK contract changes

- Add `"bin": { "automation-sdk": "./dist/cli.js" }` to `package.json`. The existing `dist` files
  entry includes the CLI output.
- Add `src/cli.ts` with a Node shebang. TypeScript must preserve the shebang in `dist/cli.js`;
  `dist/cli.d.ts` is acceptable.
- Preserve `"sideEffects": false`; do not export or execute CLI entry-point code from the package's
  public module entry point.
- Keep `AutomationManifest` definition-only, matching Automation Hub's manifest-definition type.
  It contains metadata and schemas, but no Git source coordinates.
- Keep Git source coordinates in `ManifestSource`, and return the portable artifact envelope from
  `toManifest`. The intended contract is:

  ```ts
  interface ManifestSource {
      repositoryUrl: string;
      gitCommit: string;
  }

  const toManifest = (
      definition: AutomationDefinition,
      source: ManifestSource
  ): AutomationManifestArtifact;
  ```

  where `AutomationManifestArtifact` is `{ manifest: AutomationManifest; source: ManifestSource }`.
  `entryPoint` remains under `manifest`; `source` contains only `repositoryUrl` and `gitCommit`.

  Update existing SDK callers, tests, examples, and documentation for this signature.

Changes to `automation-hub-node`, including `automationManifestSchema` and related Hub types, are
outside TSS-491.

## Shared CLI behavior

- A locally installed `automation-sdk` binary and `npx @aoins/automation-sdk` must execute the same
  entry point and behave identically.
- Resolve definition paths and `--out-dir` from the current working directory.
- Convert resolved module paths with `pathToFileURL(resolvedPath).href` before dynamic import so ESM
  imports work with Windows paths.
- Process inputs sequentially in argument order and attempt every input after a failure. Preserve
  successful outputs and return a non-zero final exit code if any input fails.
- Report errors on stderr. `--quiet` suppresses all stdout, including output written while author
  modules are imported, without suppressing stderr.

## Module format and definition discovery

TSS-491 supports compiled ESM JavaScript modules as its documented and tested path. The CLI does
not need to identify and reject CommonJS explicitly, but this story makes no compatibility promise
for CommonJS export behavior.

Importing an author module evaluates its top-level code on the author's machine or in CI. The CLI
must not call a workflow `run` function, invoke `executeAutomation` or Playwright, or run a test
suite. Documentation must advise authors to avoid top-level side effects.

Inspect only the module's default export:

1. If the module has no default export, report an error identifying the module.
2. If the default export does not have the basic runtime shape of a supported definition, report an
   error identifying the module and invalid field.
3. Ignore all named exports. Do not search them for definitions.

Named-export discovery and CommonJS support are potential future work only if concrete authoring
use cases justify them.

## Definition shape checks

The CLI checks the non-schema fields needed to safely identify the definition, generate its
manifest, and name the output file:

| Definition | Required fields | Optional fields checked when present |
|---|---|---|
| Workflow | `id`, `name`, and `entryPoint`: non-empty strings; `run`: function | `kind`: absent or `workflow`; `browser`: boolean |
| Test suite | `kind`: `test-suite`; `id`, `name`, and `entryPoint`: non-empty strings | `browserRequired`: boolean; `run` is ignored and never called |

For both kinds:

- Reject IDs containing `/`, `\`, or `:` because the ID becomes part of the filename.
- When present, `description` and `owner` must be strings.
- When present, `tags` must be an array of strings.
- Do not inspect, pre-validate, lint, or classify `input` or `output` schema values. The author is
  responsible for writing them correctly.

The command passes authored schemas to normal SDK manifest generation. TSS-491 does not promise
special diagnostics for malformed schemas and does not add invalid-schema fixtures or schema
validation tests. A dedicated schema validator is potential future work and is outside this story.

## Git source inference

Infer source coordinates independently for each definition module from the Git checkout containing
that module. Run Git with the resolved definition module's directory as its working directory:

- `git config --get remote.origin.url` supplies `repositoryUrl`.
- `git rev-parse HEAD` supplies the full commit hash for `gitCommit`.

Trim command output and require both values to be non-empty. Preserve the repository URL exactly as
configured; SSH and HTTPS remote forms are both valid. Do not infer or include a branch or Git ref.

- Git runs with the developer's inherited environment and effective configuration unchanged. A
  parseable HTTP(S) `remote.origin.url` with a non-empty URL password component is rejected before a
  manifest is written and is treated as a per-module failure. The diagnostic does not echo the
  rejected URL or secret. Username-only HTTP(S), SSH, and scp-style SSH origins remain valid and are
  preserved exactly.

Detached HEAD is valid because `git rev-parse HEAD` still identifies the commit. A dirty working
tree is also valid; the command records `HEAD` and does not inspect or report worktree status.

If Git is unavailable, the module is outside a checkout, `remote.origin.url` is absent, or `HEAD`
cannot be resolved, report the module path and failed inference clearly. Treat that module as
failed, continue with later inputs, and do not write a manifest for the failed module.

Tokens in usernames or query parameters, provider-specific secret scanning, URL rewriting,
Git-environment policy, and Git subprocess timeouts are outside this MVP.

## Manifest output

Each generated manifest artifact has exactly two top-level keys, `manifest` and `source`:

```json
{
  "manifest": {
    "manifestVersion": 1,
    "id": "homeowners.create-policy",
    "name": "Create Homeowners Policy",
    "description": "",
    "kind": "workflow",
    "browserRequired": true,
    "entryPoint": "src/create-homeowners-policy.ts",
    "owner": "",
    "tags": [],
    "inputSchema": null,
    "outputSchema": null
  },
  "source": {
    "repositoryUrl": "ssh://git@bitbucket.example/team/automations.git",
    "gitCommit": "0123456789abcdef0123456789abcdef01234567"
  }
}
```

Output rules:

- Name each file `<definition.id>.manifest.json`.
- Without `--out-dir`, write beside the resolved definition module.
- With `--out-dir`, write every manifest to that directory and create it recursively if needed. If
  the path exists as a file, report each affected write as an error.
- Overwrite existing files. When inputs resolve to the same output path, the later successful input
  wins.
- Write pretty-printed JSON with a trailing newline.
- Without `--quiet`, stdout may show generated paths and output from imported modules. With
  `--quiet`, suppress both while preserving errors on stderr.

## Failure behavior

Report the affected module path and underlying cause when a module:

- is missing, unreadable, invalid ESM, or throws during evaluation;
- cannot resolve an imported dependency;
- has no default export or an invalid default definition;
- has a parseable HTTP(S) `remote.origin.url` with a non-empty URL password component; reject it
  before writing a manifest, treat it as a per-module failure, and do not echo the rejected URL or
  secret;
- cannot provide `repositoryUrl` or `gitCommit` through Git inference; or
- cannot write its output file.

Continue after every per-file failure. Do not delete or roll back successful outputs.

## Tests

Run CLI tests as child processes against compiled ESM fixture modules. Cover:

- valid default-exported workflow and test-suite definitions;
- missing and invalid default exports while ignoring named exports;
- invalid required non-schema fields and unsafe ID characters;
- ESM import failures and partial success across multiple inputs;
- source inference from the checkout containing each definition module;
- exact `remote.origin.url` preservation and full `HEAD` commit output;
- detached HEAD and dirty working tree behavior;
- missing checkout, missing origin, and unresolved `HEAD` failures;
- overwrite order, default output location, `--out-dir`, trailing newline, and `--quiet`;
- proof that workflow `run`, Playwright, and test-suite execution are never invoked;
- package metadata and equivalent local/scoped binary entry points;
- exact `{ manifest, source }` output shape, including `entryPoint` under `manifest` and only Git
  coordinates under `source`; and
- After the SDK is built and packed, a clean temporary npm consumer can install the tarball and
  successfully run both `npx --no-install automation-sdk manifest ...` and
  `npx --no-install @aoins/automation-sdk manifest ...` against a real Git-backed ESM fixture.
  The smoke asserts the generated `{ manifest, source }` shape, installed CLI shebang/shim,
  repository URL, full commit, and cleanup of the exact temporary directory.

Do not add CommonJS compatibility, named-definition discovery, or invalid-schema validation tests.

Include the tests in `npm run verify`. Record a manual transcript in `test-report.md` that builds the
JavaScript outputs corresponding to `examples/create-homeowners-policy.ts` and
`examples/rate-match.automation.ts`, generates both manifests, and confirms their IDs, kinds,
repository URLs, full commit hashes, and schema draft identifiers.

## Documentation

Document generation of the separated `{ manifest, source }` artifact in:

- SDK README section "Generating manifests";
- `docs/contract-authoring.md` publication-artifact section; and
- `examples/README.md` command samples.

Document that authors must build ESM JavaScript first, default-export one definition per module,
provide correct schemas, run from a Git checkout with `remote.origin.url`, and avoid top-level side
effects. Mention publication only as future work that can be added as another CLI subcommand.

## Out of scope

- The `publish` CLI command, Hub URL configuration, authentication, and HTTP registration
  (TSS-522).
- All `automation-hub-node` changes, including `automationManifestSchema`, related Hub types,
  ingestion, validation, registration, and Dashboard behavior (for example, TSS-492).
- Schema validation, schema linting, invalid-schema diagnostics, and schema-dialect compatibility
  checks (TSS-525).
- Named-export definition discovery and CommonJS compatibility. Either may be considered in the
  future if a concrete use case justifies it (Automation SDK ADR 0004).
- Repository scanning or CLI-managed globbing; inputs are explicit files for the MVP. This may be
  reconsidered if discovery is requested.
- Native TypeScript execution as the primary documented flow (TSS-528).
- Git source overrides (override source inference) are not included in the MVP and may be
  reconsidered if requested.
- Automatic publication, Bitbucket webhooks, and CI-push automation (TSS-529).
