# Bunny Catalog

Package manifests for [bunny](https://github.com/cristatus/bunny), a toolchain manager for Java + Node developers.

The catalog is split out from the bunny binary so adding or bumping a package doesn't require shipping a new release.

## Scope

This catalog is **deliberately bounded** to the Java and Node developer workstation. The rule of thumb: we ship packages whose canonical distribution is a **standalone binary or tarball**. If upstream's install instructions are `npm install -g X` or `pip install X`, that's not us: install it inside the project that needs it.

We curate:

- **JVM ecosystem.** JDK distros (Temurin, Corretto, GraalVM, Liberica, Zulu, OpenJ9/Semeru, JetBrains Runtime), build tools (Maven, Maven Daemon, Gradle, sbt, Ant, JBang), language compilers (Kotlin, Scala), framework CLIs (Micronaut, Quarkus, Spring Boot), servlet containers (Tomcat), and diagnostics tools (JMC, VisualVM, async-profiler, Arthas).
- **JavaScript runtimes.** Node LTS lines, Bun, and Deno.
- **Node ecosystem tools.** pnpm's standalone executable. Yarn remains available through Corepack on Node releases that bundle it.
- **Editors and IDEs that target the above.** JetBrains IDEs (via the Toolbox app), Eclipse, NetBeans, VS Code, Cursor, Zed, Neovim.
- **AI coding agents.** The terminal agents (Claude Code, Codex CLI, opencode, pi, Antigravity) and the desktop apps that drive them (Claude Desktop, ChatGPT, GitHub Copilot, Antigravity). They earn a slot on the same grounds an editor gets one: a tool pointed at the project, not a chat client that happens to be installed. Several also publish an npm package; we take the official standalone binary instead, per the rule of thumb above.
- **General-purpose CLI tools that show up in nearly every dev workflow.** ripgrep, fd, bat, fzf, jq, yq, gh, glab, lazygit, delta, difftastic, eza, zoxide, starship, atuin, zellij, broot, dust, duf, bottom, procs, sd, hyperfine, just, tealdeer, ShellCheck, shfmt.

We do **not** accept:

- **npm-installed JS tooling.** Prettier, ESLint, TypeScript, Biome, Vite, webpack, etc. These are project dependencies; pin them in `package.json` and run via `npx` or a script.
- **Yarn.** Modern setups use Corepack (which ships with Node).
- Browsers, media players, productivity software, and chat apps that aren't a coding assistant (use Flatpak or your distro).
- Compiled-from-source language toolchains outside the JVM/Node families (use mise/asdf).
- System-level packages (use `apt`/`dnf`/`pacman`).
- Forks, beta channels, or one-off vendor builds (fork the catalog instead, see "Forking" below).

PRs outside this scope will be politely closed with a pointer to fork.

## Layout

```
.
├── index.json                  # Top-level catalog index (summary metadata per id)
├── tags.yaml                   # The tag vocabulary manifests may draw from
└── packages/{id}/manifest.yaml # One directory per package id
```

Where a manifest sits does not decide where the package lands on disk: `kind:` does, and bunny reads it from the index.

`index.json` is the entry point: bunny fetches it first to learn what packages exist and where to find their manifests. Keep `provides` and `requires` in each summary when the manifest declares them; capability-aware list, search, completion, and reverse-dependency views use this metadata without fetching every manifest.

## Adding a package

1. Confirm it fits the scope above.
2. Create `packages/{id}/manifest.yaml`. Copy the closest existing manifest as a template.
3. Append a corresponding entry to `index.json`, and declare any tag it uses in `tags.yaml`.
4. Point `catalog.local` at your checkout and run `bunny dev validate`.
5. Open a PR. CI runs the same check: the manifest schema, the tag vocabulary, and that `index.json` matches the on-disk manifests.

## Manifest shape

```yaml
id: jdk-21
name: Eclipse Temurin JDK 21
description: Production-ready OpenJDK 21 LTS distribution
version: "21.0.5+11"
kind: sdk
homepage: https://adoptium.net/
license: GPL-2.0-with-classpath-exception

provides: jdk

sources:
    - url: "https://github.com/adoptium/temurin21-binaries/releases/download/jdk-{version}/OpenJDK21U-jdk_x64_linux_hotspot.tar.gz"
      file: jdk.tar.gz
      sha256: "..."
      update:
        type: foojay
        distribution: temurin

prepare:
    - "tar xf jdk.tar.gz -C {pkg} --strip-components=1"

bin:
    - { name: java,    path: "{app}/bin/java" }
    - { name: javac,   path: "{app}/bin/javac" }
    - { name: jshell,  path: "{app}/bin/jshell" }
    - { name: jar,     path: "{app}/bin/jar" }
    - { name: keytool, path: "{app}/bin/keytool" }

env:
    JAVA_HOME: "{app}"
```

Key fields:

- `provides:`: the capability slot. Multiple packages can `provides: jdk` (Temurin, Corretto, GraalVM); `bunny use` and `.bunny-version` operate on the capability.
- `requires:`: capabilities this package needs at install + run time. A bare capability (`jdk`) needs any provider; a constraint (`jdk>=17`) needs a provider of at least that major. Satisfied providers' `env:` is merged into this package's launch.
- `sources[*].update`: per-source update backend (`github`, `html`, `json`, `foojay`, `debian`). Drives the daily auto-update cron. `sources[0]` is primary; bumping it bumps `version:`. JDKs use `foojay` with a `distribution:` (e.g. `temurin`, `corretto`, `zulu`, `graalvm_community`).
- `sources[*].update.hash-url`: an upstream checksum document. For nonstandard documents, `hash-pattern` is a regular expression whose first capture group must be a SHA-256 or SHA-512 digest.
- `prepare:`: install-time shell commands run inside an `--unshare-all` bwrap whose writable area is the staging root: `{src}` (download cache, and the working directory) and `{pkg}` (the tree that becomes `{app}`). `{data}` is writable there too, shadowed from staging and merged into the real data dir once the install commits, copying only what is not already there. That is how a package seeds a default config the user then owns: Tomcat lays down `{data}/conf` this way, and an edited `server.xml` survives the next upgrade.
- `dirs:`: directories created before launch, for what a package needs to exist but does not create itself (`{data}/logs`, `{data}/work`).
- `bin:`: what shows up in `~/.bunny/bin/` after install. `args:` injects launcher flags, `path:` is the absolute binary path inside the install.
- `global-bins:`: directories under `{data}` where this package's package manager drops runtime-installed global executables (e.g. `npm install -g`). `bunny reshim` scans them and shims what it finds.
- `env:`: env vars set when this package's binary runs; consumed by tools whose data dir is configurable via env (Maven, Gradle, Node).
- `toolchains:`: `gradle` or `maven`; bunny generates JDK-toolchain config (gradle.properties / toolchains.xml) for this build tool from the installed JDKs.

### Formatting conventions

- 4-space indent.
- An empty line between top-level keys (sources, prepare, bin, env, dirs, desktop, icons, completions).
- Inline-style `{ key: value }` for small homogeneous lists (bin entries, icon entries); block style for multi-line content.
- Quote version strings to keep YAML from interpreting `1.0` as a float.

## Automatic version bumps

A daily GitHub Actions cron runs `bunny dev update` against this catalog and pushes any new versions it finds straight to `main` (sha256 and size are refreshed alongside; a separate CI job validates every push). Don't bump `version:` manually unless upstream's archive layout also changed and you need to adjust `prepare:` to match.

For internal-only packages where you want manual review, omit the `update:` block from the source: the cron will skip it.

## Local override

Drop a manifest into `~/.bunny/catalog/{category}/{id}/manifest.yaml` to override the remote copy on your own machine. Useful for pinning, testing a manifest change before opening a PR, or hand-editing a `prepare:` step for a one-off platform issue.

## Forking

If you need packages outside the scope above (internal tools, your team's vendored JDK with corp certs), fork the catalog:

```yaml
# ~/.bunny/config.yaml
catalog:
  remote: https://raw.githubusercontent.com/your-org/bunny-catalog/main
```

See [team deployment](https://github.com/cristatus/bunny/blob/main/docs/teams.md) in the bunny repo for the full pattern.

## License

MIT
