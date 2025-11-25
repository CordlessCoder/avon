# Avon — Stop Copy-Pasting Config Files

**Avon** is a programming language for generating configuration files, boilerplate code, and entire project structures. Instead of maintaining 50 nearly-identical Kubernetes manifests or copying Docker Compose files between projects, you write one Avon program that generates them all.

[![Rust](https://img.shields.io/badge/Rust-1.70%2B-orange)](https://www.rust-lang.org/)
[![License: MIT OR Apache-2.0](https://img.shields.io/badge/license-MIT%20OR%20Apache--2.0-blue.svg)](#license)

## The Problem Avon Solves

You're tired of:
- **Copying** the same nginx config 20 times, changing just the server name
- **Maintaining** 15 near-identical Kubernetes manifests that differ only in environment variables
- **Forgetting** to update all your Docker Compose files when you change a port number
- **Wrestling** with jinja2, mustache, or sed/awk when you need even basic logic
- **Debugging** shell script template generators that grew out of control

## The Avon Solution

Write your configuration **once** as code, with:
- ✅ **Variables and functions** — DRY principle for configs
- ✅ **Lists and loops** — Generate 100 files as easily as 1
- ✅ **Conditionals** — Different output for dev/staging/prod
- ✅ **Type safety** — Catch errors before deploying
- ✅ **Real programming** — Not just text substitution

**One Avon program replaces hundreds of config files.**

## Real-World Example

**Before Avon:** You have 10 microservices, each needs a Kubernetes deployment, service, and ingress. That's 30 YAML files you copy-paste and manually edit. When you need to add a new environment variable, you edit 10 files.

**With Avon:**

```avon
let services = ["auth", "api", "frontend", "worker", "cache", "db", "queue", "analytics", "logger", "monitor"] in
let environments = ["dev", "staging", "prod"] in

let make_k8s_manifest = \service \env @/k8s/{env}/{service}-deployment.yaml {"
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: {service}
    namespace: {env}
  spec:
    replicas: {if env == "prod" then "3" else "1"}
    template:
      spec:
        containers:
        - name: {service}
          image: mycompany/{service}:latest
          env:
          - name: ENVIRONMENT
            value: {env}
          - name: LOG_LEVEL
            value: {if env == "prod" then "warn" else "debug"}
"} in

# Generate 30 files with one expression
flatmap (\env map (\svc make_k8s_manifest svc env) services) environments
```

Run once: `avon deploy k8s.av --root ./manifests`
Result: 30 perfectly consistent Kubernetes manifests, instantly.

Need to add a health check? Change one line, regenerate all 30 files.

## Why Avon Instead of...?

| Tool | Problem | Avon Solution |
|------|---------|---------------|
| **Shell scripts + sed/awk** | Unreadable after 50 lines, error-prone | Real programming language |
| **Jinja2/Mustache** | Limited logic, requires Python/Node, verbose | Full functional programming |
| **Helm** | Kubernetes-only, steep learning curve | Works for ANY text format |
| **Terraform** | Great for infrastructure, overkill for configs | Lightweight, fast, general-purpose |
| **Copy-paste** | Maintenance nightmare, inconsistency | Single source of truth |

## Quick Start — See It Work in 60 Seconds

### 1. Install

```bash
cargo build --release
./target/release/avon --version
```

### 2. Generate Your First Config

Create `docker-configs.av`:

```avon
let services = ["web", "api", "db"] in
let make_compose = \service @/docker-compose-{service}.yml {"
  version: '3.8'
  services:
    {service}:
      image: myapp/{service}:latest
      ports:
        - {if service == "web" then "80:80" else "8080:8080"}
      environment:
        SERVICE_NAME: {service}
"} in

map make_compose services
```

### 3. Generate Three Files at Once

```bash
avon docker-configs.av --deploy --root ./generated
```

**Result:** Three perfectly consistent Docker Compose files appear in `./generated/`:
- `docker-compose-web.yml` (port 80)
- `docker-compose-api.yml` (port 8080)
- `docker-compose-db.yml` (port 8080)

**That's it.** You just replaced 3 config files with 1 program. Now imagine scaling this to 100 services or 10 environments.

## Common Use Cases

### 1. Multi-Environment Configs
```avon
let environments = ["dev", "staging", "prod"] in
let make_env_config = \env @/.env.{env} {"
  DATABASE_URL=postgres://db-{env}.company.com
  API_KEY={if env == "prod" then "secret-key" else "dev-key"}
  DEBUG={if env == "prod" then "false" else "true"}
"} in
map make_env_config environments
```
**Generates:** `.env.dev`, `.env.staging`, `.env.prod`

### 2. Microservices Infrastructure
```avon
let services = ["auth", "payments", "notifications"] in
let make_k8s = \svc @/k8s/{svc}.yaml {"
  apiVersion: v1
  kind: Service
  metadata:
    name: {svc}-service
  spec:
    selector:
      app: {svc}
    ports:
    - port: 80
      targetPort: 8080
"} in
map make_k8s services
```
**Generates:** Complete Kubernetes manifests for each service

### 3. CI/CD Pipelines
```avon
let repos = ["frontend", "backend", "mobile"] in
let make_github_action = \repo @/.github/workflows/{repo}-ci.yml {"
  name: {repo} CI
  on: [push, pull_request]
  jobs:
    test:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v2
        - name: Test {repo}
          run: npm test
"} in
map make_github_action repos
```
**Generates:** GitHub Actions workflows for all repositories

### 4. Configuration from JSON
```avon
let data = json_parse "services.json" in
let services = get data "services" in
map (\svc 
  let name = get svc "name" in
  let port = get svc "port" in
  @/nginx-{name}.conf {"
    server {{
      listen 80;
      server_name {name}.example.com;
      location / {{
        proxy_pass http://localhost:{port};
      }}
    }}
  )"}) services
```
**Reads:** `services.json` with your service definitions
**Generates:** Nginx configs for every service (note: `{{` `}}` escape literal braces)

## Key Features That Make Config Generation Easy

### 1. **Variables and Functions** — Stop Copy-Pasting
```avon
let common_labels = "app=myapp,env=prod" in
let make_service = \name \port @/{name}-service.yml {"
  metadata:
    labels: {common_labels}
  spec:
    port: {port}
"} in
make_service "api" 8080
```
Change `common_labels` once → all files update

### 2. **Lists and Loops** — Generate Many Files at Once
```avon
let databases = ["users", "products", "orders"] in
map (\db @/migrations/{db}_init.sql {"
  CREATE TABLE {db} (
      id SERIAL PRIMARY KEY,
      created_at TIMESTAMP DEFAULT NOW()
  );
"}) databases
```
3 databases → 3 migration files, automatically

### 3. **Conditionals** — Different Configs Per Environment
```avon
let env = "prod" in
@/config.json {"
  {{
    "debug": {if env == "prod" then "false" else "true"},
    "replicas": {if env == "prod" then "5" else "1"},
    "log_level": "{if env == "prod" then "warn" else "debug"}"
  }}
"}
```
One program → correct config for dev, staging, or prod

### 4. **JSON Integration** — Use Existing Data
```avon
# Read from services.json: {"services": [{"name": "api", "port": 8080}, ...]}
let data = json_parse "services.json" in
let services = get data "services" in
map (\svc 
  let name = get svc "name" in
  let port = get svc "port" in
  @/docker-compose-{name}.yml {"
    services:
      {name}:
        ports: [{port}:8080]
  "}) services
```
Already have service definitions? Reuse them.

### 5. **Template Interpolation** — Simple and Powerful
```avon
@/nginx.conf {"
  server {{
    server_name {domain};
    listen {port};
    root {app_root}/public;
    
    location / {{
      proxy_pass http://localhost:{backend_port};
    }}
  }}
"}
```
Clear, readable templates with `{{` `}}` for literal braces in configs

### 6. **Works with ANY Text Format**
- ✅ YAML (Kubernetes, Docker Compose, Ansible)
- ✅ JSON (package.json, tsconfig.json, configs)
- ✅ TOML (Cargo.toml, pyproject.toml)
- ✅ HCL (Terraform)
- ✅ Shell scripts (.sh, .bash)
- ✅ Lua/Python/JavaScript code
- ✅ Nginx/Apache configs
- ✅ Markdown/HTML documentation
- ✅ **Literally anything that's text**

### 7. **Builtin Functions for Common Tasks**

**String Operations:** `upper`, `lower`, `split`, `join`, `replace`, `trim`
**List Operations:** `map`, `filter`, `fold`, `length`, `flatmap`
**File Operations:** `readfile`, `readlines`, `exists`, `basename`
**Formatting:** `format_json`, `format_hex`, `format_currency`, `pad_left`
**HTML/Markdown:** `html_escape`, `md_heading`, `md_link`, `md_code`

```avon
# Example: Generate markdown docs from a list
let features = ["Fast", "Simple", "Powerful"] in
@/README.md {"
# Features
{md_list features}
"}
```

## How to Use Avon

### Basic Commands

```bash
# Test your program (prints output, doesn't create files)
avon eval my-configs.av

# Generate files for real
avon my-configs.av --deploy --root ./output

# Pass variables from command line
avon my-configs.av --deploy -env prod -region us-east-1

# Overwrite existing files
avon my-configs.av --deploy --root ./output --force

# Only create files that don't exist yet
avon my-configs.av --deploy --if-not-exists
```

### Real-World Workflow

```bash
# 1. Write your Avon program
vim k8s-generator.av

# 2. Test it (no files created)
avon eval k8s-generator.av -env staging

# 3. Generate files to a test directory
avon k8s-generator.av --deploy -env staging --root ./test-output

# 4. Check the generated files
ls -la ./test-output/

# 5. Deploy to real directory
avon k8s-generator.av --deploy -env prod --root ./kubernetes --force
```

### Useful Flags

| Flag | What It Does | When To Use |
|------|-------------|-------------|
| `eval` | Print output, don't create files | Testing and debugging |
| `--deploy` | Actually create files | When you're ready |
| `--root <dir>` | Put all files in this directory | Organize output |
| `--force` | Overwrite existing files | Update configs |
| `--if-not-exists` | Only create new files | Don't touch existing |
| `-name value` | Pass variables | Customize generation |

### Pro Tips

**Tip 1: Always test with `eval` first**
```bash
# See what will be generated without creating files
avon eval my-program.av
```

**Tip 2: Use `--root` to keep things organized**
```bash
# All files go into ./generated/ instead of current directory
avon program.av --deploy --root ./generated
```

**Tip 3: Version control your Avon programs, not the output**
```bash
# ✅ Commit this
git add k8s-generator.av

# ❌ Don't commit this (it's generated)
echo "kubernetes/" >> .gitignore
```

**Tip 4: Share programs via GitHub**
```bash
# Run someone else's generator directly from GitHub
avon --git pyrotek45/avon/examples/docker_compose_gen.av --root ./my-docker
```

## Real Examples You Can Run Right Now

Avon includes **77 example files** that demonstrate real-world use cases:

### Infrastructure & DevOps
```bash
# Generate complete Docker Compose stack
avon examples/docker_compose_gen.av --deploy --root ./my-stack

# Generate Kubernetes manifests for 3 environments
avon examples/kubernetes_gen.av --deploy -env prod

# Generate GitHub Actions CI/CD pipelines
avon examples/github_actions_gen.av --deploy --root ./.github/workflows

# Generate Terraform configurations
avon examples/terraform_gen.av --deploy --root ./infrastructure

# Generate Nginx server configs
avon examples/nginx_gen.av --deploy --root ./nginx/conf.d
```

### Development Configs
```bash
# Generate package.json for multiple projects
avon examples/package_json_gen.av --deploy

# Generate VS Code/Neovim/Emacs configs
avon examples/neovim_config_gen.av --deploy --root ~/.config/nvim
avon examples/vim_simple.av --deploy --root ~
avon examples/emacs_init.av --deploy --root ~/.emacs.d

# Generate multi-environment .env files
avon examples/multi_file_deploy.av --deploy
```

### Static Sites & Documentation
```bash
# Generate a complete static website
avon examples/site_generator.av --deploy --root ./website

# Generate markdown documentation
avon examples/markdown_readme_gen.av --deploy

# Generate HTML pages
avon examples/html_page_gen.av --deploy --root ./public
```

### See All Examples
```bash
# List all 77 example files
ls examples/

# Try any example
avon eval examples/<name>.av
avon examples/<name>.av --deploy --root ./output
```

**Every single example is tested and works.** Pick one that's close to what you need and modify it.

## Learning Avon (It Takes 10 Minutes)

### 1. Variables
```avon
let port = 8080 in
let host = "localhost" in {"
  Server: {host}:{port}
"}
```

### 2. Functions
```avon
let make_url = \service \port "http://{service}:{port}" in
make_url "api" 8080
# Output: "http://api:8080"
```

### 3. Lists and map
```avon
let services = ["auth", "api", "web"] in
map (\s "Service: {s}") services
# Output: ["Service: auth", "Service: api", "Service: web"]
```

### 4. Conditionals
```avon
let env = "prod" in
if env == "prod" then "3 replicas" else "1 replica"
```

### 5. Generate Files
```avon
@/config.yml {"
  port: 8080
  debug: false
"}
# Creates config.yml with that content
```

### 6. Generate Multiple Files
```avon
let envs = ["dev", "prod"] in
map (\e @/config-{e}.yml {"
  env: {e}
"}) envs
# Creates config-dev.yml and config-prod.yml
```

**That's 90% of what you need to know.** The rest is just builtin functions for string manipulation, list processing, and formatting.

### Complete Syntax Reference

See [FEATURES.md](./tutorial/FEATURES.md) for every function and operator, or run:
```bash
avon --doc
```

## Who Should Use Avon?

### ✅ You Should Use Avon If You...

- **Maintain multiple similar configs** (Kubernetes, Docker, nginx, etc.)
- **Copy-paste configs** between projects or environments
- **Need to generate code** (boilerplate, migrations, etc.)
- **Manage infrastructure** across multiple environments
- **Want to version-control your config logic** instead of 100 YAML files
- **Are tired of jinja2/mustache limitations**
- **Write shell scripts** that generate files and wish they were cleaner

### ❌ You Probably Don't Need Avon If...

- You have 1-2 config files that never change
- You're happy with your current template system
- You don't mind copy-pasting and editing configs manually
- Your configs are truly one-off and never need updating

## Frequently Asked Questions

**Q: Why not just use Jinja2/Mustache/Handlebars?**
A: Those are great for simple substitution, but terrible for logic. Need to loop over a list? Generate different output based on conditions? Read from JSON? Jinja2 gets ugly fast. Avon is a real programming language designed for this.

**Q: Why not just use Python/JavaScript/Ruby scripts?**
A: You can! But you'll end up writing a lot of string concatenation and file I/O boilerplate. Avon's entire purpose is file generation, so it's optimized for that. Plus, it's a single binary with no dependencies.

**Q: Isn't this like Terraform?**
A: Terraform is amazing for infrastructure-as-code, but it's: 1) Specific to infrastructure, 2) HCL is not a general-purpose language, 3) Overkill if you just need to generate some config files. Avon works for ANY text file.

**Q: Can I use this in CI/CD?**
A: Yes! It's a single binary. Just add it to your build step:
```bash
./avon k8s-generator.av --deploy -env $ENVIRONMENT --root ./manifests --force
kubectl apply -f ./manifests/
```

**Q: Can I generate code, not just configs?**
A: Absolutely. Generate migrations, API routes, test fixtures, documentation, anything that's text.

**Q: Is it production-ready?**
A: Yes. 77 example files, 228+ automated tests, all passing. Type-safe, fast, stable.

## Documentation & Help

- **[Quick Tutorial](./tutorial/TUTORIAL.md)** — Learn Avon in 20 minutes
- **[Complete Reference](./tutorial/FEATURES.md)** — Every function, operator, and feature
- **[Style Guide](./tutorial/STYLE_GUIDE.md)** — Best practices and formatting conventions
- **[77 Working Examples](./examples/)** — Copy and modify for your needs
- **Built-in Help:** `avon --doc` shows all functions with examples

## Testing & Quality

Avon has **three layers of automated testing**:

1. **77 example files** (76 tested, 1 is an import library) — All tested examples work
2. **152 output validation tests** — Verify correct output
3. **Full integration tests** — Deployment, file creation, edge cases

```bash
# Run all tests
./scripts/test_all_examples.sh        # Quick smoke test
./scripts/test_example_outputs.sh     # Output validation  
./scripts/run_examples.sh             # Full integration

# Result: 228+ tests, all passing ✅
```

## Contributing & Development

```bash
# Build
cargo build --release

# Run tests
./scripts/test_all_examples.sh

# Check your changes
cargo test
cargo clippy
```

## Installation

```bash
# Build from source
git clone https://github.com/pyrotek45/avon
cd avon
cargo build --release

# Binary is at ./target/release/avon
./target/release/avon --version

# Optional: Add to PATH
cp ./target/release/avon ~/.local/bin/
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
