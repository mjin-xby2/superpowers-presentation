# TSS-491 manifest CLI manual test report

Date: 2026-09-03

This report records a manual build and manifest-generation demonstration using the real workflow
and test-suite examples. The commands imported the compiled definitions only; they did not invoke
workflow execution or Playwright Test execution.

## Environment and Git provenance

Command:

```powershell
Get-Date -Format 'yyyy-MM-dd'; node --version; npm --version; git --version; git config --get remote.origin.url; git rev-parse HEAD
```

Exit code: `0`

stdout:

```text
2026-09-03
v22.21.0
10.9.4
git version 2.54.0.windows.1
https://bitbucket.aoins.com/scm/tss/automation-hub-sdk-npm.git
22000f261eb9c770179ec166d5b1720ae6cc22b6
```

stderr: empty

The manifest commit is the full 40-character `HEAD`.

## Build the SDK

Command:

```powershell
npm run build
```

Exit code: `0`

stdout:

```text
> @aoins/automation-sdk@0.1.1 build
> rimraf dist && tsc -p tsconfig.json
```

stderr: empty

## Compile the real examples to ESM JavaScript

Command:

```powershell
npx tsc -p tsconfig.examples.json --noEmit false --outDir .tmp/tss-491-examples --declaration false --declarationMap false
```

Exit code: `0`

stdout: empty

stderr: empty

The configured TypeScript program preserves the `examples/` source directory beneath `outDir`.
The generated workflow and suite modules were therefore:

- `.tmp/tss-491-examples/examples/create-homeowners-policy.js`
- `.tmp/tss-491-examples/examples/rate-match.automation.js`

## Local binary form: workflow

The command used the ignored, repository-local `.tmp/tss-491-npm-cache`.

Command:

```powershell
$env:npm_config_cache = (Join-Path (Get-Location) '.tmp\tss-491-npm-cache'); npx --no-install automation-sdk manifest .tmp/tss-491-examples/examples/create-homeowners-policy.js
```

Exit code: `0`

stdout:

```text
C:\Users\e032253\Code\automation-hub-sdk-npm\.worktrees\TSS-491-sdk-manifest-cli\.tmp\tss-491-examples\examples\homeowners.create-policy.manifest.json
```

stderr:

```text
npm notice
npm notice New major version of npm available! 10.9.4 -> 12.0.2
npm notice Changelog: https://github.com/npm/cli/releases/tag/v12.0.2
npm notice To update run: npm install -g npm@12.0.2
npm notice
```

## Scoped package form and approved fallback: test suite

The scoped self-package form was checked offline so that the check could not contact a registry.

Command:

```powershell
$env:npm_config_cache = (Join-Path (Get-Location) '.tmp\tss-491-npm-cache'); $env:npm_config_offline = 'true'; npx --no-install @aoins/automation-sdk manifest .tmp/tss-491-examples/examples/rate-match.automation.js
```

Exit code: `1`

stdout: empty

stderr:

```text
node:internal/modules/cjs/loader:1386
  throw err;
  ^

Error: Cannot find module 'C:\Users\e032253\AppData\Roaming\npm\node_modules\@aoins\automation-sdk\dist\cli.js'
    at Function._resolveFilename (node:internal/modules/cjs/loader:1383:15)
    at defaultResolveImpl (node:internal/modules/cjs/loader:1025:19)
    at resolveForCJSWithHooks (node:internal/modules/cjs/loader:1030:22)
    at Function._load (node:internal/modules/cjs/loader:1192:37)
    at TracingChannel.traceSync (node:diagnostics_channel:328:14)
    at wrapModuleLoad (node:internal/modules/cjs/loader:237:24)
    at Function.executeUserEntryPoint [as runMain] (node:internal/modules/run_main:171:5)
    at node:internal/main/run_main_module:36:49 {
  code: 'MODULE_NOT_FOUND',
  requireStack: []
}

Node.js v22.21.0
```

The environment selected a stale user-level scoped shim whose package files are absent. No registry
request was made. The package metadata confirms that the scoped package and local binary map to the
same built entry point.

Command:

```powershell
node --input-type=module -e "import { readFile } from 'node:fs/promises'; const { name, bin } = JSON.parse(await readFile('package.json', 'utf8')); console.log(JSON.stringify({ name, bin }, null, 2));"
```

Exit code: `0`

stdout:

```text
{
  "name": "@aoins/automation-sdk",
  "bin": {
    "automation-sdk": "./dist/cli.js"
  }
}
```

stderr: empty

The allowed direct-dist fallback then generated the suite manifest.

Command:

```powershell
node dist/cli.js manifest .tmp/tss-491-examples/examples/rate-match.automation.js
```

Exit code: `0`

stdout:

```text
C:\Users\e032253\Code\automation-hub-sdk-npm\.worktrees\TSS-491-sdk-manifest-cli\.tmp\tss-491-examples\examples\concord.pldc-rate-match.manifest.json
```

stderr: empty

## Manifest assertions

Command:

```powershell
$origin = (git config --get remote.origin.url).Trim(); $head = (git rev-parse HEAD).Trim(); if ($head -notmatch '^[0-9a-f]{40}$') { throw "HEAD is not a 40-character hash: $head" }; $draft = 'https://json-schema.org/draft/2020-12/schema'; $cases = @(@{ Label = 'workflow'; Path = '.tmp\tss-491-examples\examples\homeowners.create-policy.manifest.json'; Id = 'homeowners.create-policy'; Kind = 'workflow' }, @{ Label = 'suite'; Path = '.tmp\tss-491-examples\examples\concord.pldc-rate-match.manifest.json'; Id = 'concord.pldc-rate-match'; Kind = 'test-suite' }); foreach ($case in $cases) { $manifest = Get-Content -Raw -LiteralPath $case.Path | ConvertFrom-Json; if (-not [string]::Equals($manifest.id, $case.Id, [System.StringComparison]::Ordinal)) { throw "$($case.Label) id mismatch" }; if (-not [string]::Equals($manifest.kind, $case.Kind, [System.StringComparison]::Ordinal)) { throw "$($case.Label) kind mismatch" }; if (-not [string]::Equals($manifest.repositoryUrl, $origin, [System.StringComparison]::Ordinal)) { throw "$($case.Label) repositoryUrl mismatch" }; if (-not [string]::Equals($manifest.gitCommit, $head, [System.StringComparison]::Ordinal)) { throw "$($case.Label) gitCommit mismatch" }; foreach ($name in @('inputSchema', 'outputSchema')) { $schema = $manifest.$name; if ($null -ne $schema -and -not [string]::Equals($schema.'$schema', $draft, [System.StringComparison]::Ordinal)) { throw "$($case.Label) $name draft mismatch" } }; Write-Output "$($case.Label): id=$($manifest.id); kind=$($manifest.kind); repositoryUrl=$($manifest.repositoryUrl); gitCommit=$($manifest.gitCommit); inputSchema=$($manifest.inputSchema.'$schema'); outputSchema=$($manifest.outputSchema.'$schema')" }; Write-Output "HEAD length: $($head.Length)"; Write-Output 'All manifest assertions passed.'
```

Exit code: `0`

stdout:

```text
workflow: id=homeowners.create-policy; kind=workflow; repositoryUrl=https://bitbucket.aoins.com/scm/tss/automation-hub-sdk-npm.git; gitCommit=22000f261eb9c770179ec166d5b1720ae6cc22b6; inputSchema=https://json-schema.org/draft/2020-12/schema; outputSchema=https://json-schema.org/draft/2020-12/schema
suite: id=concord.pldc-rate-match; kind=test-suite; repositoryUrl=https://bitbucket.aoins.com/scm/tss/automation-hub-sdk-npm.git; gitCommit=22000f261eb9c770179ec166d5b1720ae6cc22b6; inputSchema=https://json-schema.org/draft/2020-12/schema; outputSchema=https://json-schema.org/draft/2020-12/schema
HEAD length: 40
All manifest assertions passed.
```

stderr: empty

Assertions established:

- workflow ID `homeowners.create-policy` and kind `workflow`;
- suite ID `concord.pldc-rate-match` and kind `test-suite`;
- both `repositoryUrl` values exactly equal
  `https://bitbucket.aoins.com/scm/tss/automation-hub-sdk-npm.git`;
- both `gitCommit` values exactly equal
  `22000f261eb9c770179ec166d5b1720ae6cc22b6`, the 40-character `HEAD`;
- all four non-null input/output schemas declare
  `https://json-schema.org/draft/2020-12/schema`.

## Execution guard check

Command:

```powershell
Test-Path -LiteralPath 'examples\run-output.json'; Test-Path -LiteralPath 'examples\artifacts'; Test-Path -LiteralPath '.tmp\tss-491-examples\examples\homeowners.create-policy.manifest.json'; Test-Path -LiteralPath '.tmp\tss-491-examples\examples\concord.pldc-rate-match.manifest.json'
```

Exit code: `0`

stdout:

```text
False
False
True
True
```

stderr: empty

No workflow output or workflow artifact directory was created. No Playwright Test command was
invoked. Both generated manifests remained present.

## ESLint generated-output regression

The generated `.tmp/tss-491-examples` tree and both manifests remained present for the RED and
GREEN commands.

### RED: full verification linted configured scratch output

The original pre-fix full verification failure is retained here. The focused pre-fix
`npm run lint` reproduction produced the same 19-problem diagnostic.

Command:

```powershell
npm run verify
```

Exit code: `1`

stdout:

```text
> @aoins/automation-sdk@0.1.1 verify
> npm run typecheck && npm run lint && npm run test && npm run build && npm run pack:dry-run


> @aoins/automation-sdk@0.1.1 typecheck
> tsc -p tsconfig.json --noEmit && tsc -p tsconfig.examples.json --noEmit


> @aoins/automation-sdk@0.1.1 lint
> eslint . --max-warnings 0


C:\Users\e032253\Code\automation-hub-sdk-npm\.worktrees\TSS-491-sdk-manifest-cli\.tmp\tss-491-examples\examples\create-homeowners-policy.js
  41:13  warning  Unused eslint-disable directive (no problems were reported from 'no-param-reassign')

C:\Users\e032253\Code\automation-hub-sdk-npm\.worktrees\TSS-491-sdk-manifest-cli\.tmp\tss-491-examples\src\io.js
  10:56  error  'process' is not defined  no-undef
  15:65  error  'process' is not defined  no-undef
  18:44  error  'process' is not defined  no-undef

C:\Users\e032253\Code\automation-hub-sdk-npm\.worktrees\TSS-491-sdk-manifest-cli\.tmp\tss-491-examples\src\logger.js
  10:40  error  'process' is not defined  no-undef
  11:39  error  'process' is not defined  no-undef
  12:39  error  'process' is not defined  no-undef

C:\Users\e032253\Code\automation-hub-sdk-npm\.worktrees\TSS-491-sdk-manifest-cli\.tmp\tss-491-examples\src\runtime.js
   12:31  error  'process' is not defined  no-undef
   15:32  error  'process' is not defined  no-undef
   17:55  error  'process' is not defined  no-undef
   39:60  error  'process' is not defined  no-undef
   51:26  error  'process' is not defined  no-undef
   61:55  error  'process' is not defined  no-undef
   71:26  error  'process' is not defined  no-undef
  104:25  error  'process' is not defined  no-undef
  109:9   error  'process' is not defined  no-undef
  110:9   error  'process' is not defined  no-undef

C:\Users\e032253\Code\automation-hub-sdk-npm\.worktrees\TSS-491-sdk-manifest-cli\.tmp\tss-491-examples\src\slack.js
  3:28  error  'process' is not defined  no-undef
  7:32  error  'fetch' is not defined    no-undef

✖ 19 problems (18 errors, 1 warning)
  0 errors and 1 warning potentially fixable with the `--fix` option.
```

stderr: empty

### GREEN: intentional scratch output is ignored

Change: add only `.tmp/**` to the top-level `ignores` array in `eslint.config.mjs`.

Command:

```powershell
npm run lint
```

Exit code: `0`

stdout:

```text
> @aoins/automation-sdk@0.1.1 lint
> eslint . --max-warnings 0
```

stderr: empty

## Pre-review verification

### Formatting

Command:

```powershell
npm run format -- --log-level silent
```

Exit code: `0`

stdout:

```text
> @aoins/automation-sdk@0.1.1 format
> prettier --write . --log-level silent
```

stderr: empty

Command:

```powershell
npm run format:check
```

Exit code: `0`

stdout:

```text
> @aoins/automation-sdk@0.1.1 format:check
> prettier --check .

Checking formatting...
All matched files use Prettier code style!
```

stderr: empty

### Full verification with generated output retained

Command:

```powershell
npm run verify
```

Exit code: `0`

stdout:

```text
> @aoins/automation-sdk@0.1.1 verify
> npm run typecheck && npm run lint && npm run test && npm run build && npm run pack:dry-run


> @aoins/automation-sdk@0.1.1 typecheck
> tsc -p tsconfig.json --noEmit && tsc -p tsconfig.examples.json --noEmit


> @aoins/automation-sdk@0.1.1 lint
> eslint . --max-warnings 0


> @aoins/automation-sdk@0.1.1 test
> vitest run


 RUN  v4.1.10 C:/Users/e032253/Code/automation-hub-sdk-npm/.worktrees/TSS-491-sdk-manifest-cli

fixture stdout
{"details":{"automationId":"example.no-contracts"},"level":"INFO","message":"Automation started.","timestamp":"2026-09-03T12:49:06.214Z"}
{"details":{"automationId":"example.no-contracts"},"level":"INFO","message":"Automation completed.","timestamp":"2026-09-03T12:49:06.217Z"}
{"details":{"automationId":"example.run-if-main"},"level":"INFO","message":"Automation started.","timestamp":"2026-09-03T12:49:06.221Z"}
{"details":{"automationId":"example.run-if-main"},"level":"INFO","message":"Automation completed.","timestamp":"2026-09-03T12:49:06.221Z"}

 Test Files  11 passed (11)
      Tests  68 passed (68)
   Start at  08:49:02
   Duration  49.73s (transform 727ms, setup 0ms, import 4.91s, tests 54.69s, environment 2ms)


> @aoins/automation-sdk@0.1.1 build
> rimraf dist && tsc -p tsconfig.json


> @aoins/automation-sdk@0.1.1 pack:dry-run
> npm pack --dry-run

aoins-automation-sdk-0.1.1.tgz
```

stderr:

```text
npm notice
npm notice 📦  @aoins/automation-sdk@0.1.1
npm notice Tarball Contents
npm notice 5.8kB README.md
npm notice 171B dist/artifacts.d.ts
npm notice 210B dist/artifacts.d.ts.map
npm notice 521B dist/artifacts.js
npm notice 709B dist/artifacts.js.map
npm notice 64B dist/cli.d.ts
npm notice 100B dist/cli.d.ts.map
npm notice 147B dist/cli.js
npm notice 229B dist/cli.js.map
npm notice 239B dist/cli/arguments.d.ts
npm notice 296B dist/cli/arguments.d.ts.map
npm notice 1.2kB dist/cli/arguments.js
npm notice 1.3kB dist/cli/arguments.js.map
npm notice 228B dist/cli/definition.d.ts
npm notice 267B dist/cli/definition.d.ts.map
npm notice 2.4kB dist/cli/definition.js
npm notice 2.2kB dist/cli/definition.js.map
npm notice 279B dist/cli/generate-manifest.d.ts
npm notice 339B dist/cli/generate-manifest.d.ts.map
npm notice 1.5kB dist/cli/generate-manifest.js
npm notice 1.7kB dist/cli/generate-manifest.js.map
npm notice 175B dist/cli/git.d.ts
npm notice 221B dist/cli/git.d.ts.map
npm notice 1.1kB dist/cli/git.js
npm notice 1.2kB dist/cli/git.js.map
npm notice 210B dist/cli/load-definition.d.ts
npm notice 264B dist/cli/load-definition.d.ts.map
npm notice 915B dist/cli/load-definition.js
npm notice 995B dist/cli/load-definition.js.map
npm notice 114B dist/cli/main.d.ts
npm notice 191B dist/cli/main.d.ts.map
npm notice 1.0kB dist/cli/main.js
npm notice 1.2kB dist/cli/main.js.map
npm notice 166B dist/cli/run-git-command.d.ts
npm notice 267B dist/cli/run-git-command.d.ts.map
npm notice 384B dist/cli/run-git-command.js
npm notice 581B dist/cli/run-git-command.js.map
npm notice 912B dist/index.d.ts
npm notice 790B dist/index.d.ts.map
npm notice 473B dist/index.js
npm notice 479B dist/index.js.map
npm notice 792B dist/io.d.ts
npm notice 624B dist/io.d.ts.map
npm notice 1.1kB dist/io.js
npm notice 1.4kB dist/io.js.map
npm notice 896B dist/jsonSchema.d.ts
npm notice 1.0kB dist/jsonSchema.d.ts.map
npm notice 49B dist/jsonSchema.js
npm notice 112B dist/jsonSchema.js.map
npm notice 147B dist/logger.d.ts
npm notice 191B dist/logger.d.ts.map
npm notice 537B dist/logger.js
npm notice 767B dist/logger.js.map
npm notice 614B dist/manifest.d.ts
npm notice 553B dist/manifest.d.ts.map
npm notice 973B dist/manifest.js
npm notice 1.1kB dist/manifest.js.map
npm notice 799B dist/runtime.d.ts
npm notice 706B dist/runtime.d.ts.map
npm notice 4.5kB dist/runtime.js
npm notice 4.6kB dist/runtime.js.map
npm notice 1.7kB dist/schema.d.ts
npm notice 1.1kB dist/schema.d.ts.map
npm notice 2.3kB dist/schema.js
npm notice 2.9kB dist/schema.js.map
npm notice 182B dist/slack.d.ts
npm notice 219B dist/slack.d.ts.map
npm notice 700B dist/slack.js
npm notice 739B dist/slack.js.map
npm notice 3.9kB dist/types.d.ts
npm notice 4.0kB dist/types.d.ts.map
npm notice 44B dist/types.js
npm notice 102B dist/types.js.map
npm notice 2.0kB package.json
npm notice Tarball Details
npm notice name: @aoins/automation-sdk
npm notice version: 0.1.1
npm notice filename: aoins-automation-sdk-0.1.1.tgz
npm notice package size: 18.5 kB
npm notice unpacked size: 73.1 kB
npm notice shasum: 16ff8521174244acafeafcfb819fa1ef9582fd25
npm notice integrity: sha512-VpBn5nxjPehyq[...]sUn/65tE4q7gg==
npm notice total files: 74
npm notice
```

The `.tmp/tss-491-examples` compiled modules and both generated manifests remained present after
the successful gate.
