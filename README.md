# Avon — Generate and Deploy Configs with Code

**Avon** is a programming language AND deployment system for configuration files. Write one program, deploy hundreds of files.

**The language is only half the story—the `@` syntax deployment system is what makes Avon different.**

[![Rust](https://img.shields.io/badge/Rust-1.70%2B-orange)](https://www.rust-lang.org/)
[![License: MIT OR Apache-2.0](https://img.shields.io/badge/license-MIT%20OR%20Apache--2.0-blue.svg)](#license)

## The Problem

Stop copy-pasting configs. Stop maintaining 50 nearly-identical YAML files. Stop forgetting to update that one config when you change a port.

**Traditional approach:** Generate → Save → Copy → Repeat 100 times  
**Avon:** Write once → `avon --deploy` → 100 files appear

## What Makes Avon Different

**Two integrated systems:**

1. **Functional Language** — Variables, functions, lists, conditionals, type checking
2. **Deployment System** — `@/path/to/file.yml {"content"}` syntax writes files directly

**Result:** One command generates and deploys everything. No intermediate steps.

## Real Example

10 services × 3 environments = 30 Kubernetes manifests. **One Avon program:**

```avon
let services = ["auth", "api", "frontend", "worker", "cache"] in
let environments = ["dev", "staging", "prod"] in

let make_k8s = \svc \env @/k8s/{env}/{svc}-deployment.yaml {"
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: {svc}
    namespace: {env}
  spec:
    replicas: {if env == "prod" then "3" else "1"}
    containers:
    - name: {svc}
      image: myapp/{svc}:latest
      env:
      - name: LOG_LEVEL
        value: {if env == "prod" then "warn" else "debug"}
"} in

flatmap (\env map (\svc make_k8s svc env) services) environments
```

Run: `avon k8s.av --deploy --root ./manifests`  
Result: 15 files created instantly. Change one line, redeploy all.

## Quick Start

**Install:**
```bash
cargo build --release
./target/release/avon --version
```

**Create `docker-configs.av`:**
```avon
let services = ["web", "api", "db"] in
let make_compose = \svc @/docker-compose-{svc}.yml {"
  version: '3.8'
  services:
    {svc}:
      image: myapp/{svc}:latest
      ports:
        - {if svc == "web" then "80:80" else "8080:8080"}
"} in
map make_compose services
```

**Deploy:**
```bash
avon docker-configs.av --deploy --root ./generated
```

**Result:** 3 files created in `./generated/`. Scale this to 100 services just as easily.

## Use Cases

**Multi-environment configs:**
```avon
let envs = ["dev", "staging", "prod"] in
map (\env @/.env.{env} {"
  DATABASE_URL=postgres://db-{env}.company.com
  API_KEY={if env == "prod" then "secret" else "dev-key"}
"}) envs
```

**CI/CD pipelines:**
```avon
let repos = ["frontend", "backend", "mobile"] in
map (\repo @/.github/workflows/{repo}-ci.yml {"
  name: {repo} CI
  on: [push, pull_request]
  jobs:
    test:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v2
        - run: npm test
"}) repos
```

**From JSON data:**
```avon
let data = json_parse "services.json" in
let services = get data "services" in
map (\svc 
  let name = get svc "name" in
  @/nginx-{name}.conf {"
    server {{
      listen 80;
      server_name {name}.example.com;
    }}
  "}) services
```

## Key Features

**`@` Deployment Syntax** — The game changer
```avon
@/path/to/file.yml {"content goes here"}
```
Files know where they belong. `avon --deploy` writes everything at once.

**Functional Programming** — Variables, functions, map/filter/fold, conditionals

**Type Safety** — `typeof`, `is_*`, `assert_*` functions catch errors early

**80+ Builtins** — String ops, list ops, formatting, JSON, file I/O, HTML/Markdown helpers

**Any Text Format** — YAML, JSON, TOML, HCL, shell scripts, code, configs, docs

**Excellent Errors** — Clickable file:line:column, shows variable definitions, helpful hints

Run `avon --doc` for complete function reference.

## Commands

```bash
# Test (no files written)
avon eval program.av

# Deploy files
avon program.av --deploy --root ./output

# Pass variables
avon program.av --deploy -env prod -region us-east-1

# Overwrite existing
avon program.av --deploy --force

# Quick eval from command line
avon --eval-input 'map (\x x * 2) [1, 2, 3]'
```

**Workflow:** Write → Test with `eval` → Deploy to test dir → Deploy to production

## Examples

**77 working examples in `examples/` directory:**
- Docker Compose, Kubernetes, Terraform, GitHub Actions
- Nginx configs, environment files, CI/CD pipelines
- Neovim/Emacs configs, static sites, markdown docs

```bash
ls examples/                    # See all examples
avon eval examples/nginx_gen.av # Try one
```

## Learn Avon in 5 Minutes

```avon
# Variables
let port = 8080 in
let host = "localhost" in

# Functions
let make_url = \svc \p "http://{svc}:{p}" in

# Lists and map
let services = ["auth", "api", "web"] in
map (\s make_url s port) services

# Conditionals
let env = "prod" in
if env == "prod" then "3 replicas" else "1 replica"

# Generate files with @ syntax
@/config.yml {"
  port: {port}
  debug: {if env == "prod" then "false" else "true"}
"}
```

See [TUTORIAL.md](./tutorial/TUTORIAL.md) for complete guide or run `avon --doc`.

## Documentation

- **[Tutorial](./tutorial/TUTORIAL.md)** | **[Reference](./tutorial/FEATURES.md)** | **[Style Guide](./tutorial/STYLE_GUIDE.md)**
- **77 working examples** in `./examples/`
- **`avon --doc`** for built-in help

## Quality

- **338+ tests passing** (109 unit + 228+ integration)
- **Excellent error messages** with clickable locations
- Type-safe, single binary, no dependencies

## Installation

```bash
git clone https://github.com/pyrotek45/avon
cd avon
cargo build --release
./target/release/avon --version
```

## License

MIT OR Apache-2.0 — Use it however you want.

---

## Get Started in 3 Commands

```bash
# 1. Build
cargo build --release

# 2. Try an example
./target/release/avon eval examples/docker_compose_gen.av

# 3. Generate your first configs
./target/release/avon examples/docker_compose_gen.av --deploy --root ./my-configs
```

**Stop maintaining 50 config files. Maintain 1 Avon program.**
