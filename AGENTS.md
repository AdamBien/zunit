# AGENTS.md

Guidance for AI agents working on this repository or integrating zunit into other projects.

## What this is

zunit is a zero-dependency test runner for Java 25+, shipped as a **single executable source file**: [`zunit`](zunit). There is no build step, no JAR, no dependencies — the script *is* the product. It discovers `*Test.java` (unit) and `*IT.java` (integration) source files and runs each in its own JVM via `java --source 25`, with assertions (`-ea`) enabled.

## Repository layout

```
zunit               # the runner — a Java 25 source-mode script with a shebang, no .java extension
test/               # self-tests: run ./zunit inside a sample project to exercise them
README.md           # human-facing usage
```

## Working on the runner

- Everything lives in the single `zunit` file; keep it that way — no extra source files, no external libraries.
- The script maintains `String version = "YYYY-MM-DD.N";` — on every change, set the date to today and bump `N` (reset to 1 on a new day).
- The shebang is `#!/usr/bin/env -S java --source 25` — never add `--enable-preview` or classpath entries.
- Colored output follows [zcl](https://github.com/AdamBien/zcl) conventions.

## Using zunit in a project (for agents writing tests)

- A test is a self-contained source script with `void main()` (or `void main(String... args)`), failing via any thrown exception or non-zero exit — plain `assert condition : message;` is the idiom.
- Naming: `SomethingTest.java` for unit tests, `SomethingIT.java` for integration tests (run after unit tests; skip with `-skip-it`).
- Test sources auto-detected from `src/test/java/`, then `test/`, then `.`. Classpath auto-detected from `.zb` (`jar.dir` + `jar.file.name`), then `zbo/app.jar`, `target/classes`, `classes/`, `.`. Override with `-tp:<path>` / `-cp:<path>`.
- **Exit code contract**: `0` when all tests pass, non-zero when any test fails. This is the machine-readable result — parse nothing else.

## CI / CD integration

zunit is fetched straight from this repository's `main` branch — single file, stable URL:

```
https://raw.githubusercontent.com/AdamBien/zunit/main/zunit
```

GitHub Actions steps for a [zb](https://github.com/AdamBien/zb) project (requires `actions/setup-java` with Java 25+, e.g. temurin):

```yaml
- name: Fetch zunit
  run: |
    curl -fL -o zunit https://raw.githubusercontent.com/AdamBien/zunit/main/zunit
    chmod +x zunit
    echo "$PWD" >> "$GITHUB_PATH"

- name: Build
  run: java -jar zb.jar

- name: Test
  run: zunit
```

Two integration rules agents get wrong:

1. **zb's post-build hook does not gate.** Projects typically declare `post.build.hook=zunit` in `.zb`; zb runs the hook but exits `0` even when the hook fails (or is missing). Putting zunit on `GITHUB_PATH` makes the hook work in CI, but a release/deploy job **must run `zunit` as its own step** so a red suite fails the pipeline.
2. **Fetch before build.** Add the runner to the PATH before invoking zb, otherwise the hook logs `zunit: command not found` (a warning, easily mistaken for green).

Local equivalent: `zb && zunit` — same gating semantics, the `&&` does what the explicit CI step does.
