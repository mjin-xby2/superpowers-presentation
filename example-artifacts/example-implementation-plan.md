# SDK Manifest CLI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a dependency-free `automation-sdk manifest` command that imports explicit compiled ESM definitions, infers their Git source, and writes portable manifest JSON files while isolating failures.

**Architecture:** Keep `src/cli.ts` as a shebang-bearing executable shell. Put parsing, safe ESM loading, per-file manifest generation, and command orchestration in focused modules under `src/cli/`; the command processes resolved inputs sequentially and converts each failure into stderr plus a final non-zero status. Exercise the real command in child processes against compiled `.mjs` definitions and temporary Git repositories.

**Tech Stack:** Node.js 22 built-ins, TypeScript 6, Vitest 4, existing SDK manifest APIs, Git CLI

**Spec:** `jira/TSS-491-sdk-manifest-cli/spec.md`

## Global Constraints

- Add no runtime or development dependencies; use Node 22 built-ins.
- Support compiled ESM JavaScript modules only; inspect only `default` and make no CommonJS guarantee.
- Never invoke a workflow `run`, Playwright, `executeAutomation`, or a test suite.
- Do not pre-validate, lint, or classify authored `input` or `output` schemas.
- Resolve definition paths and `--out-dir` from the command's current working directory.
- Process inputs sequentially in argument order, preserve successful outputs, and return non-zero if any input fails.
- Preserve `sideEffects: false`; do not export CLI code from `src/index.ts`.
- Keep named exports in the public SDK API and arrow functions in all new TypeScript.
- Before review, run both `npm run format` and `npm run verify`.

## Completed prerequisites

These approved-scope changes are already implemented and should only be adjusted if a later integration test exposes a defect:

- `ManifestSource`, `AutomationManifest.repositoryUrl`, and `AutomationManifest.gitCommit` in `src/types.ts`.
- `toManifest(definition, source)` in `src/manifest.ts` and its public export from `src/index.ts`.
- `getDefaultDefinition(...)` and its non-schema shape checks in `src/cli/definition.ts`.
- `inferManifestSource(...)` and `runGitCommand(...)` in `src/cli/git.ts` and `src/cli/run-git-command.ts`.

---

### Task 1: Parse the manifest command without a CLI dependency

**Files:**

- Create: `src/cli/arguments.ts`
- Create: `test/cli-arguments.test.ts`

**Interfaces:**

- Consumes: raw `process.argv.slice(2)` values.
- Produces: `ManifestCliOptions` and `parseArguments(args: string[]): ManifestCliOptions`.

- [ ] **Step 1: Write failing argument-parser tests**

```ts
import { describe, expect, it } from 'vitest';
import { parseArguments } from '../src/cli/arguments.js';

describe('parseArguments', () => {
	it('parses only the required command and definition file', () => {
		expect(parseArguments(['manifest', 'a.js'])).toEqual({
			definitionFiles: ['a.js'],
			outDir: undefined,
			quiet: false,
		});
	});

	it('parses manifest inputs and flags', () => {
		expect(
			parseArguments(['manifest', '--quiet', '--out-dir', 'generated', 'a.js', 'b.js'])
		).toEqual({
			definitionFiles: ['a.js', 'b.js'],
			outDir: 'generated',
			quiet: true,
		});
	});

	it('accepts optional fields in any order', () => {
		expect(
			parseArguments(['manifest', 'a.js', '--out-dir', 'generated', 'b.js', '--quiet'])
		).toEqual({
			definitionFiles: ['a.js', 'b.js'],
			outDir: 'generated',
			quiet: true,
		});
	});

	it.each([
		[[], 'expected a command'],
		[['unknown'], "unsupported command 'unknown'"],
		[['manifest'], 'manifest requires at least one definition file'],
		[['manifest', '--out-dir'], '--out-dir requires a directory'],
		[['manifest', '--wat', 'a.js'], "unknown option '--wat'"],
	] as const)('rejects invalid arguments %#', (args, message) => {
		expect(() => parseArguments([...args])).toThrow(message);
	});
});
```

- [ ] **Step 2: Run the focused test and verify it fails because the module is missing**

Run: `npm test -- test/cli-arguments.test.ts`

Expected: FAIL with an import error for `src/cli/arguments.js`.

- [ ] **Step 3: Implement the minimal parser**

```ts
export interface ManifestCliOptions {
	definitionFiles: string[];
	outDir: string | undefined;
	quiet: boolean;
}

export const parseArguments = (args: string[]): ManifestCliOptions => {
	const [command, ...commandArgs] = args;

	if (command === undefined) {
		throw new Error('expected a command');
	}

	if (command !== 'manifest') {
		throw new Error(`unsupported command '${command}'`);
	}

	const definitionFiles: string[] = [];
	let outDir: string | undefined;
	let quiet = false;

	for (let index = 0; index < commandArgs.length; index += 1) {
		const argument = commandArgs[index];
		if (argument === '--quiet') {
			quiet = true;
		} else if (argument === '--out-dir') {
			outDir = commandArgs[index + 1];
			if (outDir === undefined || outDir.startsWith('--')) {
				throw new Error('--out-dir requires a directory');
			}
			index += 1;
		} else if (argument?.startsWith('--')) {
			throw new Error(`unknown option '${argument}'`);
		} else if (argument !== undefined) {
			definitionFiles.push(argument);
		}
	}

	if (definitionFiles.length === 0) {
		throw new Error('manifest requires at least one definition file');
	}

	return { definitionFiles, outDir, quiet };
};
```

- [ ] **Step 4: Run the focused test**

Run: `npm test -- test/cli-arguments.test.ts`

Expected: PASS.

- [ ] **Step 5: Commit the parser**

```bash
git add src/cli/arguments.ts test/cli-arguments.test.ts
git commit -m "TSS-491 - parse manifest CLI arguments"
```

### Task 2: Import only the default ESM definition and suppress import stdout

**Files:**

- Create: `src/cli/load-definition.ts`
- Create: `test/fixtures/cli/noisy-workflow.mjs`
- Create: `test/fixtures/cli/quiet-workflow.mjs`
- Create: `test/fixtures/cli/aliased-default-workflow.mjs`
- Create: `test/fixtures/cli/throwing-module.mjs`
- Create: `test/cli-load-definition.test.ts`

**Interfaces:**

- Consumes: an absolute module path and `quiet` boolean.
- Produces: `loadDefinition(modulePath: string, quiet: boolean): Promise<AutomationDefinition>`.

- [ ] **Step 1: Add compiled ESM fixtures**

```js
// test/fixtures/cli/noisy-workflow.mjs
process.stdout.write('fixture stdout\n');

export const namedDefinition = { id: 'ignored.named' };
export default {
	entryPoint: 'src/noisy.ts',
	id: 'fixture.noisy',
	name: 'Noisy workflow',
	run: () => {
		throw new Error('run must not be called');
	},
};
```

```js
// test/fixtures/cli/throwing-module.mjs
throw new Error('fixture import failed');
```

Create `quiet-workflow.mjs` with the same contents as `noisy-workflow.mjs` but change the ID to
`fixture.quiet`. The distinct URL ensures its top-level stdout has not already been consumed by
Node's ESM module cache.

Create `aliased-default-workflow.mjs` using a local definition followed by an aliased default export
and at least one additional named export. This proves discovery uses the ESM module namespace's
`default` property and does not depend on the `export default ...` spelling:

```js
const definition = {
	entryPoint: 'src/aliased.ts',
	id: 'fixture.aliased-default',
	name: 'Aliased default workflow',
	run: () => {
		throw new Error('run must not be called');
	},
};

const metadata = 'ignored named export';

export { definition as default, metadata };
```

- [ ] **Step 2: Write failing loader tests**

```ts
import { resolve } from 'node:path';
import { describe, expect, it, vi } from 'vitest';
import { loadDefinition } from '../src/cli/load-definition.js';

describe('loadDefinition', () => {
	it('imports the default definition without invoking run', async () => {
		const definition = await loadDefinition(
			resolve('test/fixtures/cli/noisy-workflow.mjs'),
			false
		);
		expect(definition.id).toBe('fixture.noisy');
	});

	it('imports a definition exported through an aliased default export', async () => {
		const definition = await loadDefinition(
			resolve('test/fixtures/cli/aliased-default-workflow.mjs'),
			false
		);

		expect(definition.id).toBe('fixture.aliased-default');
	});

	it('suppresses stdout only while a quiet module is imported', async () => {
		const write = vi.spyOn(process.stdout, 'write').mockImplementation(() => true);
		await loadDefinition(resolve('test/fixtures/cli/quiet-workflow.mjs'), true);
		expect(write).not.toHaveBeenCalled();
		write.mockRestore();
	});

	it('preserves the import failure cause', async () => {
		await expect(
			loadDefinition(resolve('test/fixtures/cli/throwing-module.mjs'), false)
		).rejects.toThrow('fixture import failed');
	});
});
```

- [ ] **Step 3: Run the focused test and verify it fails because the loader is missing**

Run: `npm test -- test/cli-load-definition.test.ts`

Expected: FAIL with an import error for `src/cli/load-definition.js`.

- [ ] **Step 4: Implement file-URL import with `try/finally` stdout restoration**

```ts
import { pathToFileURL } from 'node:url';
import type { AutomationDefinition } from '../types.js';
import { getDefaultDefinition } from './definition.js';

const importModule = async (
	modulePath: string,
	quiet: boolean
): Promise<Record<string, unknown>> => {
	const moduleUrl = pathToFileURL(modulePath).href;
	const originalWrite = process.stdout.write;
	if (quiet) {
		process.stdout.write = (() => true) as typeof process.stdout.write;
	}
	try {
		return (await import(moduleUrl)) as Record<string, unknown>;
	} catch (error) {
		const cause = error instanceof Error ? error.message : String(error);
		throw new Error(`${modulePath}: cannot import definition module: ${cause}`);
	} finally {
		process.stdout.write = originalWrite;
	}
};

export const loadDefinition = async (
	modulePath: string,
	quiet: boolean
): Promise<AutomationDefinition> =>
	getDefaultDefinition(modulePath, await importModule(modulePath, quiet));
```

- [ ] **Step 5: Run loader and existing definition tests**

Run: `npm test -- test/cli-load-definition.test.ts test/cli-definition.test.ts`

Expected: PASS.

- [ ] **Step 6: Commit safe module loading**

```bash
git add src/cli/load-definition.ts test/fixtures/cli test/cli-load-definition.test.ts
git commit -m "TSS-491 - load manifest definitions safely"
```

### Task 3: Generate one manifest file from one definition

**Files:**

- Create: `src/cli/generate-manifest.ts`
- Create: `test/cli-test-helpers.ts`
- Create: `test/cli-generate-manifest.test.ts`

**Interfaces:**

- Consumes: `definitionFile` plus `{ cwd, outDir, quiet }`.
- Produces: `generateManifest(...): Promise<string>` returning the absolute output path.
- Uses: `loadDefinition`, `inferManifestSource`, and `toManifest`.

- [ ] **Step 1: Create a temporary Git-repository test helper**

Define these shared types in `test/cli-test-helpers.ts`:

```ts
export interface DefinitionFixture {
	file?: string;
	id: string;
	kind?: 'test-suite' | 'workflow';
	name?: string;
	source?: string;
}

export interface GitDefinitionFixture {
	directory: string;
	git: (args: string[]) => Promise<string>;
	gitCommit: string;
	modulePath: string;
	repositoryUrl: string;
}

export type SingleDefinitionFixture = DefinitionFixture & { repositoryUrl?: string };
```

Implement `createGitDefinitionFiles(definitions, repositoryUrl)` by using `mkdir`, `mkdtemp`,
`writeFile`, `dirname`, and promisified `execFile`. For a fixture without a custom `source`, write
this compiled ESM text, substituting its fields:

```js
export default {
	entryPoint: 'src/fixture.ts',
	id: 'fixture.workflow',
	name: 'Fixture workflow',
	run: () => {
		throw new Error('run must not be called');
	},
};
```

For `kind: 'test-suite'`, emit `kind: 'test-suite'` and omit `run`. Initialize and commit the files
with these exact operations:

```ts
await git(['init']);
await git(['config', 'user.email', 'automation-sdk@example.test']);
await git(['config', 'user.name', 'Automation SDK Test']);
await git(['remote', 'add', 'origin', repositoryUrl]);
await mkdir(dirname(modulePath), { recursive: true });
await writeFile(modulePath, moduleSource, 'utf8');
await git(['add', '.']);
await git(['commit', '-m', 'fixture']);
const gitCommit = (await git(['rev-parse', 'HEAD'])).trim();
return { directory, git, gitCommit, modulePath, repositoryUrl };
```

Use `join(tmpdir(), ...)` paths and an arrow-function `git(args: string[])` bound to the temporary
directory. Give `createGitDefinitionFiles(definitions, repositoryUrl)` a default repository URL of
`ssh://git@bitbucket.example/team/fixture.git`. Export
`createGitDefinition(options: SingleDefinitionFixture)` as a one-file wrapper that separates
`options.repositoryUrl` before calling `createGitDefinitionFiles([definition], repositoryUrl)`.
`modulePath` is the absolute path of the first definition file.

- [ ] **Step 2: Write failing per-file generation tests**

Import `mkdir` and `join` for the nested invocation-directory case in addition to the helpers used
by the existing test cases.

```ts
it('writes a source-complete manifest beside the definition', async () => {
	const fixture = await createGitDefinition({
		id: 'fixture.workflow',
		repositoryUrl: 'ssh://git@bitbucket.example/team/fixture.git',
	});
	const outputPath = await generateManifest('definition.mjs', {
		cwd: fixture.directory,
		outDir: undefined,
		quiet: true,
	});
	const text = await readFile(outputPath, 'utf8');
	const manifest = JSON.parse(text) as AutomationManifest;
	expect(outputPath).toBe(join(fixture.directory, 'fixture.workflow.manifest.json'));
	expect(text.endsWith('\n')).toBe(true);
	expect(manifest).toMatchObject({
		gitCommit: fixture.gitCommit,
		id: 'fixture.workflow',
		repositoryUrl: fixture.repositoryUrl,
	});
});

it('creates and resolves an out directory from cwd', async () => {
	const fixture = await createGitDefinition({ id: 'fixture.out-dir' });
	await expect(
		generateManifest('definition.mjs', {
			cwd: fixture.directory,
			outDir: 'nested/manifests',
			quiet: false,
		})
	).resolves.toBe(join(fixture.directory, 'nested/manifests/fixture.out-dir.manifest.json'));
});

it('resolves a definition path relative to the invocation cwd', async () => {
	const fixture = await createGitDefinition({
		file: 'dist/definition.mjs',
		id: 'fixture.relative-path',
	});
	const invocationDirectory = join(fixture.directory, 'tools/scripts');

	await mkdir(invocationDirectory, { recursive: true });

	await expect(
		generateManifest('../../dist/definition.mjs', {
			cwd: invocationDirectory,
			outDir: undefined,
			quiet: true,
		})
	).resolves.toBe(join(fixture.directory, 'dist/fixture.relative-path.manifest.json'));
});
```

This test owns the relative-path requirement: `generateManifest` is the boundary that combines the
user-supplied definition path with the invocation cwd. It must pass the resulting absolute path to
both `loadDefinition` and Git-source inference. `loadDefinition` itself continues to consume a
pre-resolved module path and converts it to a file URL for ESM import.

- [ ] **Step 3: Run the focused test and verify it fails because generation is missing**

Run: `npm test -- test/cli-generate-manifest.test.ts`

Expected: FAIL with an import error for `src/cli/generate-manifest.js`.

- [ ] **Step 4: Implement per-file generation**

```ts
import { mkdir, writeFile } from 'node:fs/promises';
import { dirname, join, resolve } from 'node:path';
import type { AutomationManifest } from '../types.js';
import { toManifest } from '../manifest.js';
import { inferManifestSource } from './git.js';
import { loadDefinition } from './load-definition.js';

export interface GenerateManifestOptions {
	cwd: string;
	outDir: string | undefined;
	quiet: boolean;
}

export const generateManifest = async (
	definitionFile: string,
	options: GenerateManifestOptions
): Promise<string> => {
	const modulePath = resolve(options.cwd, definitionFile);

	const definition = await loadDefinition(modulePath, options.quiet);
	const source = await inferManifestSource(modulePath);

	const outputDirectory = options.outDir
		? resolve(options.cwd, options.outDir)
		: dirname(modulePath);

	const outputPath = join(outputDirectory, `${definition.id}.manifest.json`);

	let manifest: AutomationManifest;

	try {
		manifest = toManifest(definition, source);
	} catch (error) {
		const cause = error instanceof Error ? error.message : String(error);

		throw new Error(`${modulePath}: cannot generate manifest: ${cause}`);
	}

	try {
		if (options.outDir !== undefined) {
			await mkdir(outputDirectory, { recursive: true });
		}

		await writeFile(outputPath, `${JSON.stringify(manifest, null, 2)}\n`, 'utf8');
	} catch (error) {
		const cause = error instanceof Error ? error.message : String(error);

		throw new Error(`${modulePath}: cannot write manifest '${outputPath}': ${cause}`);
	}

	return outputPath;
};
```

- [ ] **Step 5: Run per-file, Git, definition, and manifest tests**

Run: `npm test -- test/cli-generate-manifest.test.ts test/cli-git.test.ts test/cli-definition.test.ts test/manifest.test.ts`

Expected: PASS.

- [ ] **Step 6: Commit manifest file generation**

```bash
git add src/cli/generate-manifest.ts test/cli-test-helpers.ts test/cli-generate-manifest.test.ts
git commit -m "TSS-491 - generate manifest files"
```

### Task 4: Add the executable shell and resilient sequential orchestration

**Files:**

- Create: `src/cli/main.ts`
- Create: `src/cli.ts`
- Create: `test/cli.test.ts`

**Interfaces:**

- Produces: `runCli(args: string[], cwd?: string): Promise<number>`.
- Executable: `src/cli.ts` assigns the returned status to `process.exitCode`.

- [ ] **Step 1: Add a child-process runner to `test/cli-test-helpers.ts`**

```ts
export interface CliProcessResult {
	exitCode: number | null;
	stderr: string;
	stdout: string;
}

const cliPath = resolve('src/cli.ts');
const tsxLoader = import.meta.resolve('tsx');

export const runCliProcess = (args: string[], cwd: string): Promise<CliProcessResult> =>
	new Promise((resolveProcess, rejectProcess) => {
		const child = spawn(process.execPath, ['--import', tsxLoader, cliPath, ...args], { cwd });
		let stderr = '';
		let stdout = '';
		child.stderr.setEncoding('utf8');
		child.stdout.setEncoding('utf8');
		child.stderr.on('data', chunk => {
			stderr += String(chunk);
		});
		child.stdout.on('data', chunk => {
			stdout += String(chunk);
		});
		child.on('error', rejectProcess);
		child.on('close', exitCode => {
			resolveProcess({ exitCode, stderr, stdout });
		});
	});
```

Import `spawn` from `node:child_process` and `resolve` from `node:path`. The absolute CLI path must
be calculated before spawning because the child `cwd` is the temporary automation repository.

- [ ] **Step 2: Write failing orchestration tests**

```ts
it('processes every input and returns non-zero after a partial failure', async () => {
	const fixture = await createGitDefinition({ id: 'fixture.success' });
	const result = await runCliProcess(
		['manifest', 'missing.mjs', 'definition.mjs'],
		fixture.directory
	);
	expect(result.exitCode).toBe(1);
	expect(result.stderr).toContain(resolve(fixture.directory, 'missing.mjs'));
	expect(result.stdout).toContain('fixture.success.manifest.json');
	await expect(
		readFile(join(fixture.directory, 'fixture.success.manifest.json'), 'utf8')
	).resolves.toContain('"id": "fixture.success"');
});

it('writes later successful inputs over the same output path', async () => {
	const fixture = await createGitDefinitionFiles([
		{ file: 'first.mjs', id: 'fixture.same', name: 'First' },
		{ file: 'second.mjs', id: 'fixture.same', name: 'Second' },
	]);
	const result = await runCliProcess(
		['manifest', '--out-dir', 'out', 'first.mjs', 'second.mjs'],
		fixture.directory
	);
	expect(result.exitCode).toBe(0);
	const manifest = JSON.parse(
		await readFile(join(fixture.directory, 'out/fixture.same.manifest.json'), 'utf8')
	) as AutomationManifest;
	expect(manifest.name).toBe('Second');
});
```

Also assert an unsupported command returns `1`, prints the parser message on stderr, and writes nothing to stdout.

Add a real subprocess case in which the temporary repository contains `dist/definition.mjs`, the
CLI is invoked with cwd set to a nested `tools/scripts` directory, and the argument is
`../../dist/definition.mjs`. Assert the command succeeds and writes the manifest beside the module.
Repeat with `--out-dir generated` and assert that output is resolved beneath `tools/scripts`, proving
both user-supplied paths are relative to the directory from which the manifest command is invoked.

- [ ] **Step 3: Run the CLI integration test and verify it fails because the executable is missing**

Run: `npm test -- test/cli.test.ts`

Expected: FAIL because `src/cli.ts` cannot be loaded.

- [ ] **Step 4: Implement sequential orchestration**

```ts
// src/cli/main.ts
import type { ManifestCliOptions } from './arguments.js';
import { parseArguments } from './arguments.js';
import { generateManifest } from './generate-manifest.js';

const errorMessage = (error: unknown): string =>
	error instanceof Error ? error.message : String(error);

export const runCli = async (args: string[], cwd = process.cwd()): Promise<number> => {
	let options: ManifestCliOptions;
	try {
		options = parseArguments(args);
	} catch (error) {
		process.stderr.write(`${errorMessage(error)}\n`);
		return 1;
	}

	let failed = false;
	for (const definitionFile of options.definitionFiles) {
		try {
			const outputPath = await generateManifest(definitionFile, {
				cwd,
				outDir: options.outDir,
				quiet: options.quiet,
			});
			if (!options.quiet) {
				process.stdout.write(`${outputPath}\n`);
			}
		} catch (error) {
			failed = true;
			process.stderr.write(`${errorMessage(error)}\n`);
		}
	}
	return failed ? 1 : 0;
};
```

```ts
#!/usr/bin/env node
// src/cli.ts
import { runCli } from './cli/main.js';

process.exitCode = await runCli(process.argv.slice(2));
```

- [ ] **Step 5: Run parser, generator, and child-process tests**

Run: `npm test -- test/cli-arguments.test.ts test/cli-generate-manifest.test.ts test/cli.test.ts`

Expected: PASS.

- [ ] **Step 6: Build and verify the shebang is preserved**

Run: `npm run build`

Expected: PASS and the first line of `dist/cli.js` is `#!/usr/bin/env node`.

- [ ] **Step 7: Commit the executable command**

```bash
git add src/cli.ts src/cli/main.ts test/cli.test.ts test/cli-test-helpers.ts
git commit -m "TSS-491 - add manifest CLI entry point"
```

### Task 5: Complete ESM, Git, quiet-mode, and failure integration coverage

**Files:**

- Modify: `test/cli.test.ts`
- Modify: `test/cli-test-helpers.ts`
- Create: `test/fixtures/cli/named-only.mjs`
- Create: `test/fixtures/cli/invalid-default.mjs`
- Create: `test/fixtures/cli/noisy-stderr-workflow.mjs`

**Interfaces:** No new production API; this task proves the command-level acceptance criteria and permits focused fixes in `src/cli/**` if a test exposes a defect.

- [ ] **Step 1: Add the remaining compiled ESM fixtures**

```js
// test/fixtures/cli/named-only.mjs
export const definition = {
	entryPoint: 'src/named.ts',
	id: 'fixture.named',
	name: 'Named only',
	run: () => undefined,
};
```

```js
// test/fixtures/cli/invalid-default.mjs
export default {};
```

```js
// test/fixtures/cli/noisy-stderr-workflow.mjs
import { writeFileSync } from 'node:fs';

process.stdout.write('fixture stdout\n');
process.stderr.write('fixture stderr\n');
export default {
	entryPoint: 'src/noisy.ts',
	id: 'fixture.noisy-stderr',
	name: 'Noisy stderr workflow',
	run: () => writeFileSync('run-sentinel', 'called'),
};
```

- [ ] **Step 2: Add table-driven module discovery and import failure tests**

Cover these exact cases in child processes:

```ts
it.each([
	['named-only.mjs', 'module must default-export one automation definition'],
	['invalid-default.mjs', 'default export id, name, and entryPoint must be non-empty strings'],
	['throwing-module.mjs', 'fixture import failed'],
])('reports %s without stopping later inputs', async (fixtureName, message) => {
	const compiledSourceCode = await readFile(
		resolve('test/fixtures/cli', fixtureName),
		'utf8'
	);
	const fixture = await createGitDefinitionFiles([
		{ file: fixtureName, id: 'unused.invalid', source: compiledSourceCode },
		{ file: 'valid.mjs', id: 'fixture.success' },
	]);
	const result = await runCliProcess(['manifest', fixtureName, 'valid.mjs'], fixture.directory);
	expect(result.exitCode).toBe(1);
	expect(result.stderr).toContain(join(fixture.directory, fixtureName));
	expect(result.stderr).toContain(message);
	await expect(
		readFile(join(fixture.directory, 'fixture.success.manifest.json'), 'utf8')
	).resolves.toContain('"id": "fixture.success"');
});
```

Add the same partial-success assertions for two dynamically written compiled-module failures so an
invalid fixture does not enter ESLint's source scan:

```ts
it.each([
	['syntax-error.mjs', 'export default {', 'cannot import definition module'],
	[
		'missing-dependency.mjs',
		"import './dependency-that-does-not-exist.mjs';",
		'dependency-that-does-not-exist.mjs',
	],
] as const)('reports ESM failure for %s', async (file, compiledSourceCode, message) => {
	const fixture = await createGitDefinitionFiles([
		{ file, id: 'unused.invalid', source: compiledSourceCode },
		{ file: 'valid.mjs', id: 'fixture.success' },
	]);
	const result = await runCliProcess(['manifest', file, 'valid.mjs'], fixture.directory);
	expect(result.exitCode).toBe(1);
	expect(result.stderr).toContain(join(fixture.directory, file));
	expect(result.stderr).toContain(message);
	await expect(
		readFile(join(fixture.directory, 'fixture.success.manifest.json'), 'utf8')
	).resolves.toContain('"id": "fixture.success"');
});
```

Retain the existing unit matrix for invalid `kind`, required fields, optional metadata, booleans, tags, and unsafe ID characters rather than duplicating every shape case through a child process.

- [ ] **Step 3: Add quiet and non-execution tests**

Use a compiled fixture that writes one line to stdout and one to stderr at top level and whose workflow `run` would create a sentinel file. Assert:

- without `--quiet`, stdout contains the fixture line and generated path;
- with `--quiet`, stdout is exactly empty;
- stderr remains visible with `--quiet`;
- the sentinel file is absent;
- a test-suite definition containing a throwing `run` property is accepted without calling it.

- [ ] **Step 4: Add real Git-state tests**

Using `createGitDefinition(...)`, cover:

```ts
await fixture.git(['checkout', '--detach', fixture.gitCommit]);
await writeFile(join(fixture.directory, 'dirty.txt'), 'dirty', 'utf8');
```

Generate the manifest and assert the recorded commit remains the exact 40-character `fixture.gitCommit`. Separately assert exact preservation of both SSH and HTTPS origin URLs.

- [ ] **Step 5: Add Git failure tests**

Create separate temporary directories for:

- a definition outside any Git checkout;
- an initialized repository without `remote.origin.url`;
- an initialized repository with an origin but no commit, so `HEAD` is unresolved.

For each, assert exit `1`, stderr includes the absolute module path and either `cannot infer repositoryUrl` or `cannot infer gitCommit`, and no manifest was written. Include a later valid input to prove processing continues.

- [ ] **Step 6: Add output failure and overwrite tests**

Create a regular file at the requested `--out-dir` path, pass two valid inputs, and assert each affected write is reported, final exit is `1`, and neither manifest exists. Pre-create a valid manifest output with stale content and assert a successful run overwrites it with formatted JSON ending in one newline.

- [ ] **Step 7: Run the full CLI test group and fix only exposed defects**

Run: `npm test -- test/cli-*.test.ts test/cli.test.ts`

Expected: PASS with child processes proving partial success, Git states, import safety, output behavior, and quiet mode.

- [ ] **Step 8: Commit the completed behavioral coverage**

```bash
git add src/cli test/cli.test.ts test/cli-test-helpers.ts test/fixtures/cli
git commit -m "TSS-491 - cover manifest CLI behavior"
```

### Task 6: Expose the packaged binary and verify package metadata

**Files:**

- Modify: `package.json`
- Regenerate through npm only: `package-lock.json`
- Create: `test/package.test.ts`

**Interfaces:**

- Produces package binary mapping `automation-sdk -> ./dist/cli.js`.
- Does not add the CLI to the public `exports` map or `src/index.ts`.

- [ ] **Step 1: Write the failing package metadata test**

```ts
import { readFile } from 'node:fs/promises';
import { describe, expect, it } from 'vitest';

describe('package manifest CLI', () => {
	it('maps local and scoped npm invocation to the same executable', async () => {
		const packageJson = JSON.parse(await readFile('package.json', 'utf8')) as {
			bin?: Record<string, string>;
			name?: string;
			sideEffects?: boolean;
		};
		expect(packageJson.name).toBe('@aoins/automation-sdk');
		expect(packageJson.bin).toEqual({ 'automation-sdk': './dist/cli.js' });
		expect(packageJson.sideEffects).toBe(false);
	});
});
```

- [ ] **Step 2: Run the package test and verify the missing `bin` failure**

Run: `npm test -- test/package.test.ts`

Expected: FAIL because `packageJson.bin` is undefined.

- [ ] **Step 3: Add the binary mapping**

Add this top-level property to `package.json` without changing `exports` or `sideEffects`:

```json
"bin": {
	"automation-sdk": "./dist/cli.js"
}
```

Do not edit `package-lock.json` directly. Ask the user for approval to run `npm i`, then run it so
npm synchronizes the root package metadata into the lockfile. Inspect the resulting lockfile diff and
confirm it contains the expected root `bin` metadata change and no unintended dependency-version
changes.

- [ ] **Step 4: Run the package test, build, and dry-run pack**

Run: `npm test -- test/package.test.ts`

Expected: PASS.

Run: `npm run build`

Expected: PASS; `dist/cli.js` and `dist/cli.d.ts` exist.

Run: `npm run pack:dry-run`

Expected: PASS; the file list includes `dist/cli.js`, and no test fixtures or CLI source modules outside `dist` are packed.

- [ ] **Step 5: Commit package exposure**

```bash
git add package.json package-lock.json test/package.test.ts
git commit -m "TSS-491 - expose automation SDK binary"
```

### Task 7: Document, manually demonstrate, and verify manifest generation

**Files:**

- Modify: `README.md`
- Modify: `docs/contract-authoring.md`
- Modify: `examples/README.md`
- Modify: `.gitignore`
- Create: `jira/TSS-491-sdk-manifest-cli/test-report.md`

**Interfaces:** Documentation for SDK authors and the Gate 2 evidence record.

- [ ] **Step 1: Update the SDK README**

Replace the stale `toManifest(definition)` text with `toManifest(definition, source)` and add a `Generating manifests` section containing:

```bash
npm run build
automation-sdk manifest dist/automations/create-homeowners-policy.js
npx @aoins/automation-sdk manifest --out-dir manifests dist/automations/create-homeowners-policy.js
```

State explicitly that inputs must be compiled ESM JavaScript, each module must default-export one definition, Git must provide `remote.origin.url` and `HEAD`, schemas remain author-owned, and top-level module code executes during import.

- [ ] **Step 2: Add publication guidance to contract authoring**

Add a `Publication artifact` section to `docs/contract-authoring.md` explaining that the definition remains the single source for static inference, runtime validation, and generated JSON Schema; the CLI does not separately validate schemas; and the manifest records the exact configured repository URL and full commit hash, including detached or dirty checkouts.

- [ ] **Step 3: Add workflow and test-suite commands to the examples README**

Document building the example TypeScript to ESM JavaScript before invoking:

```bash
automation-sdk manifest .tmp/tss-491-examples/create-homeowners-policy.js
automation-sdk manifest .tmp/tss-491-examples/rate-match.automation.js
```

Warn that `runIfMain` is safe because the file is imported rather than invoked as the main entry point, but arbitrary top-level side effects still execute.

- [ ] **Step 4: Prepare ignored manual-output space**

Add `.tmp/` to `.gitignore`, then build SDK and example JavaScript:

```bash
npm run build
npx tsc -p tsconfig.examples.json --noEmit false --outDir .tmp/tss-491-examples --declaration false --declarationMap false
```

- [ ] **Step 5: Run both supported command forms manually**

Use the local binary shim for one example and the scoped npm form for the other, targeting the same built package entry point:

```bash
npx --no-install automation-sdk manifest .tmp/tss-491-examples/create-homeowners-policy.js
npx --no-install @aoins/automation-sdk manifest .tmp/tss-491-examples/rate-match.automation.js
```

If npm does not resolve the scoped self-package form without registry access, document that environmental limitation and demonstrate equivalence by recording the package `name` and `bin` mapping plus a second direct execution of `dist/cli.js`; do not contact a registry merely to satisfy this local check.

- [ ] **Step 6: Record exact evidence in `test-report.md`**

Record the date, Node/npm/Git versions, commands, exit codes, stdout/stderr, and assertions that:

- workflow ID is `homeowners.create-policy` and kind is `workflow`;
- suite ID is `concord.pldc-rate-match` and kind is `test-suite`;
- both `repositoryUrl` values exactly equal `git config --get remote.origin.url`;
- both `gitCommit` values exactly equal the 40-character `git rev-parse HEAD` result;
- every non-null schema has `$schema` equal to `https://json-schema.org/draft/2020-12/schema`.

Do not write expected placeholders into the report: paste the actual values observed during execution.

- [ ] **Step 7: Run formatting and focused documentation checks**

Run: `npm run format`

Expected: PASS and only intended files are reformatted.

Run: `npm run format:check`

Expected: PASS.

- [ ] **Step 8: Run final verification before review**

Run: `npm run verify`

Expected: typecheck, lint, every Vitest test, build, and package dry-run all pass; the packed file list contains the CLI.

The two mandatory pre-review commands are therefore `npm run format` in Step 7 and
`npm run verify` in this step. Re-run either command if later edits could invalidate its result.

- [ ] **Step 9: Review the final diff against the accepted scope**

Run: `git diff --check`

Expected: no whitespace errors.

Run: `git status --short`

Expected: only TSS-491 implementation, test, documentation, plan, and report files are changed; `.tmp/`, generated manifests, `dist/`, and `node_modules/` are absent from status.

- [ ] **Step 10: Commit documentation and verification evidence**

```bash
git add .gitignore README.md docs/contract-authoring.md examples/README.md docs/superpowers/plans/2026-09-02-sdk-manifest-cli.md
git add -f jira/TSS-491-sdk-manifest-cli/test-report.md
git commit -m "TSS-491 - document manifest generation"
```

The JIRA folder is intentionally ignored, so use `git add -f` only if this repository's review convention requires the test report in source control; otherwise attach the report to the JIRA and omit it from the commit.
