# Learning Notes — Data Engineering Zoomcamp

> One file for every topic covered. Add a short section per new topic (bullets, tables, abbreviations expanded).
> Instructions and current environment live in [../CLAUDE.md](../CLAUDE.md).

---

## 1. Docker

### Core concepts
- **Image** = read-only blueprint (`python:3.13.10-slim`). **Container** = running instance of an image
- **Containers are ephemeral** — changes are lost on removal unless you use volumes
- Building an image creates a file holding OS + Python + packages + code. Nothing is installed on the host
- **Images are per-environment** — an image built in Codespaces doesn't exist locally
- **One container = one purpose** — Postgres in one, pipeline code in another, connected via a network
- **Docker Desktop** (Mac/Windows) must be running before any `docker` command. Linux uses Docker engine directly. Production uses **K8s** (**K**ubernete**s**) or AWS **ECS** (**E**lastic **C**ontainer **S**ervice)

### `run` vs `start` vs `exec`
| Command | Does | Notes |
|---|---|---|
| `docker run` | always creates a NEW container | all flags allowed |
| `docker start -i <name>` | restarts an **Exited** container | only `-i`; tty and entrypoint were fixed at creation |
| `docker exec -it <name> bash` | runs a command in a **running** container | container name comes BEFORE the command |

### Flags
| Short | Long | Meaning |
|---|---|---|
| `-i` | `--interactive` | keep STDIN (**st**andar**d in**put) open |
| `-t` | `--tty` | allocate a terminal (**t**ele**ty**pewriter) — but in `docker build`, `-t` = **t**ag |
| `-d` | `--detach` | run in the background |
| `-e` | `--env` | set an environment variable (runtime only) |
| `-v` | `--volume` | mount a volume |
| `-p` | `--publish` | port mapping `HOST:CONTAINER` |
| `-f` | `--force` | in `docker rm -f` — stop and remove |
| | `--rm` | delete the container on exit |
| | `--name` | custom name (instead of `nervous_haibt`) |
| | `--entrypoint` | override the startup command (at creation only) |

### Commands
| Command | Purpose |
|---|---|
| `docker ps` / `ps -a` / `ps -aq` | running / all / IDs only (**q**uiet) — `ps` = **p**rocess **s**tatus |
| `docker rm $(docker ps -aq)` | remove all containers |
| `docker rmi <image>` | remove an image (**rm** **i**mage) |
| `docker images` | list images + sizes |
| `docker logs <name>` | container output (useful with `-d`) |
| `docker container prune` | remove stopped containers |
| `docker system df` | disk usage (**d**isk **f**ree) |
| `docker system prune -a` | remove everything unused (rebuildable from the Dockerfile) |
| `docker network create/ls/inspect/connect` | manage networks |
| `docker volume ls/inspect/rm` | manage named volumes |

### Tags & registries
- `name:tag`; no tag → `latest`. **Always pin versions**. Use lowercase tags (`v001`) — tags are case-sensitive
- Variants: full (~1GB, has compilers) · **`slim`** (~150MB, the default for pipelines) · `alpine` (~50MB, compatibility issues)
- Registries: Docker Hub (`docker.io`, default), **GHCR** = **G**it**H**ub **C**ontainer **R**egistry (`ghcr.io`)

### Dockerfile — standard template
```dockerfile
FROM python:3.13.10-slim                          # 1. base image
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/ # 2. tools (copy a binary from another image)
WORKDIR /app                                       # 3. working dir (like cd)
ENV PATH="/app/.venv/bin:$PATH"                    # 4. env var baked into the image
COPY pyproject.toml uv.lock .python-version ./     # 5. dependency files only
RUN uv sync --locked --no-install-project          # 6. install deps
COPY pipeline.py .                                 # 7. code — changes most → last
ENTRYPOINT ["python", "pipeline.py"]               # 8. startup command (exec form)
```
- Instructions are case-insensitive (UPPERCASE by convention); filenames inside them ARE case-sensitive
- **Layer caching:** each step is a cached layer; a change invalidates that layer and everything below → stable things on top, code at the bottom (**split-COPY pattern**)
- **COPY:** the last argument is always the destination; with several sources the destination must be a directory (`./`). `.` = `./` = WORKDIR
- **Exec form** `["python","pipeline.py"]` → the process is PID (**p**rocess **id**) 1 and gets SIGTERM (**sig**nal **term**inate) for a clean shutdown. **Shell form** `python pipeline.py` → `sh` is PID 1, killed after 10s (half-written files)
- **Build context:** `docker build -t name:tag .` — the final `.` is required (build only, not `run`). Use `.dockerignore` to exclude large files

### Reading errors (bottom → top)
| Message | Cause |
|---|---|
| `not found` / `No such file or directory` | wrong name / case / version (ENTRYPOINT typo → builds OK, crashes at runtime) |
| `failed to calculate checksum` | COPY source missing (often a dotfile) |
| `exit code: 2` | command in `RUN` failed |
| `port is already allocated` | another container holds the port |
| `pull access denied` | image not built locally and not on Docker Hub |
| `Temporary failure in name resolution` | DNS — container has no internet |
| `Connection refused` | nothing is listening / wrong host |
- Exit codes: `0` success, `130` Ctrl+C (SIGINT = **sig**nal **int**errupt), other non-zero = failure

### Volumes, ports, networking
- **Bind mount** `-v /host/path:/container/path` — you choose the location (code, config)
- **Named volume** `-v name:/container/path` — Docker manages it; best for databases; survives `docker rm` (only `docker volume rm` deletes it)
- **Ports** `-p 5431:5432` = host 5431 → container 5432. Postgres 5432, MySQL 3306, Redis 6379, HTTP 80/443
- `localhost` inside a container = that container itself. Host → container works via `-p`. Container → container needs a **shared network + container name** (Docker's internal **DNS**). Never use IPs — they change

### Postgres in Docker
```bash
docker run -it --rm --name pgdatabase --network pg-network \
  -e POSTGRES_USER="root" -e POSTGRES_PASSWORD="root" -e POSTGRES_DB="ny_taxi" \
  -v ny_taxi_postgres_data:/var/lib/postgresql -p 5432:5432 postgres:18
```
- The image's startup script reads the `POSTGRES_*` env vars on first start
- **WAL** = **W**rite-**A**head **L**og — journal used for crash recovery
- Production: managed DBs — **RDS** (**R**elational **D**atabase **S**ervice, AWS), Cloud SQL (GCP)

### Docker Compose
- One `docker-compose.yml` (**YAML** = **Y**AML **A**in't **M**arkup **L**anguage) for many services; **auto-creates a shared network**; services reach each other by **service name**
- Container names: `<project>-<service>-<n>` (e.g. `pipeline-pgdatabase-1`)
- See [pipeline/docker-compose.yaml](../pipeline/docker-compose.yaml) for the Postgres + pgAdmin example

| `docker run` | Compose |
|---|---|
| `--name` | service key |
| `-e` | `environment:` |
| `-v` | `volumes:` (+ top-level `volumes:` declares named volumes) |
| `-p` | `ports:` |
| `--network` | automatic |
| `-d` / `--rm` | `up -d` / `down` |

- Commands: `up`, `up -d`, `down`, `ps`, `logs <svc>`, `exec <svc> <cmd>` (adds `-it` automatically)
- YAML: indent with **spaces** (never tabs), `key: value`, `-` = list item

---

## 2. Databases & ingestion

### Connecting from Python
- **SQLAlchemy** = connection layer (an **ORM** = **O**bject-**R**elational **M**apper); pandas uses it for `to_sql`. The steering wheel
- **psycopg** = the Postgres **driver**. The engine. `"psycopg[binary,pool]"` — extras: `binary` = pre-compiled, `pool` = reuse connections. Quote it: `[]` is a bash glob
- Connection string: `postgresql://user:password@host:port/database`

### psql / pgcli / pgAdmin
- **psql** = built-in CLI client. **pgcli** = with auto-complete (`-h` host, `-p` port, `-u` user, `-d` database)
- `\dt` **d**isplay **t**ables · `\d t` **d**escribe · `\dn` **n**amespaces (schemas) · `\l` **l**ist DBs · `\q` **q**uit · `\r` **r**eset — psql only, not bash
- End SQL with `;`. Prompt `=#` ready; `-#` still waiting for `;`
- **pgAdmin** = web GUI (port 8085 → 80). Host = the Postgres **service name**. **CSRF** (**C**ross-**S**ite **R**equest **F**orgery) token error → refresh

### Ingestion with pandas
- `pd.read_csv(url, dtype=..., parse_dates=[...])` — reads `.csv.gz` directly. Set `dtype` explicitly to avoid `DtypeWarning: mixed types`
- **Chunked load:**
```python
df_iter = pd.read_csv(url, dtype=dtype, parse_dates=parse_dates, iterator=True, chunksize=100000)
first = True
for chunk in tqdm(df_iter):
    if first:
        chunk.head(0).to_sql(name=table, con=engine, if_exists='replace')  # schema only
        first = False
    chunk.to_sql(name=table, con=engine, if_exists='append')  # NOT in an else — that skips chunk 1
```
- **tqdm** = Arabic *taqaddum* (تقدّم, progress) — progress bars. **click** = **C**ommand **L**ine **I**nterface **C**reation **K**it

### Notebook → script → Docker
1. Prototype in **Jupyter** (**Ju**lia + **Py**thon + **R**); `.ipynb` = **I**nteractive **Py**thon **N**ote**b**ook
2. `jupyter nbconvert --to=script notebook.ipynb`
3. Clean up: delete `get_ipython()...`, `df.head()`/`print`, `!pip install`, stray code; move imports to the top; replace hardcoded values with CLI args/env vars; use `if __name__ == '__main__':` (runs only when executed, not when imported)
4. Dockerize with an ENTRYPOINT

---

## 3. Linux, Bash & WSL

### WSL
- **WSL** = **W**indows **S**ubsystem for **L**inux — real Linux inside Windows
- `wsl -d Ubuntu` enter · `exit` leave · `wsl --shutdown` stop the VM (files kept) · `wsl --set-default Ubuntu`
- Work in `~`, never under `/mnt/c/...` (slow, permission issues). VS Code: `code .` from Ubuntu (WSL extension)
- Package managers: `apt` (**A**dvanced **P**ackage **T**ool, Ubuntu) · winget/Chocolatey (Windows) · Homebrew (macOS)
- `sudo` = **s**uper**u**ser **do** · `lsb_release -cs` = **L**inux **S**tandard **B**ase **c**odename, **s**hort

### Commands
- `ls` list · `cd` change directory · `pwd` print working directory · `mkdir` make directory · `mv` move · `cp` copy · `rm` remove · `cat` concatenate · `touch` create empty file · `chmod` change mode · `grep` global regular expression print · `which` path of a command · `wget` web get · `nc -zv host port` netcat (test a port)
- `ls -la` columns: type+permissions, links, owner, group, size, date, name
- `-` = single-letter flags (combinable: `-la`); `--` = word flags. `-v` usually verbose, `-V` version
- Use **Tab** to complete names — `cd` won't guess partial names

### Syntax gotchas
- `VAR=value` ✅ · `VAR = value` ❌ (bash runs `VAR` as a command)
- `\` at line end = continue; **nothing** after it, not even a space. PowerShell uses `` ` ``
- `>` **overwrites** ⚠️ · `>>` appends · `2>/dev/null` discards stderr (file descriptor 2)
- ⚠️ Pasting `docker build` output into the shell creates junk files — every `>` in it becomes a redirect (that's how `=`, `CACHED`, `ERROR`… appeared in this repo)
- `A && B` B if A succeeds · `A || B` B if A fails · `A ; B` always · `A &` background
- `$(cmd)` = command substitution (avoid the old backticks)
- Heredoc: `cat > file << 'EOF' ... EOF`
- Case-sensitive everything: `pipeline.py` ≠ `pipeline.PY`, `CLAUDE.md` ≠ `CLAUDE.MD`

### Dotfiles & config
- Files starting with `.` are hidden → `ls -a`. The dot is part of the name (`.python-version` ≠ `python-version`)
- `.bashrc` = bash **r**un **c**ommands, runs on every new shell. Add a line: `echo 'LINE' >> ~/.bashrc` then `source ~/.bashrc`
  - Past typos: `bashtc`, `bashrc` (no dot), missing `~/`, a quoted string with no `echo`
- `PS1` = **P**rompt **S**tring 1 · `/etc/skel/` = **skel**eton defaults for new users

### Environment variables
- `PATH` = `:`-separated dirs searched **left → right**, first match wins. Prepend: `PATH="/app/.venv/bin:$PATH"`
- WORKDIR = where you are; PATH = where programs are searched for
- OS-meaningful: `PATH`, `HOME`, `USER`, `LANG`, `EDITOR`. Your own (`DATABASE_URL`) are convention → `os.getenv()`
- **12-Factor App**: config lives in env vars, not in code
- **DNS** = **D**omain **N**ame **S**ystem — names → IPs (`8.8.8.8` = Google DNS)

| Bash | PowerShell |
|---|---|
| `$(pwd)` | `${PWD}` |
| `\` | `` ` `` |
| `export VAR=value` | `$env:VAR="value"` |

---

## 4. Python & uv

- `sys` (argv, exit) · `os` (files, env vars) · `pathlib.Path` (`Path.cwd()`, `Path(__file__).name`, `.is_file()`, `.read_text()`)
- `sys.argv[0]` = script name; arguments are **strings** → `int(sys.argv[1])`
- f-string `f'{prefix}file.csv'`; never mix with `+`. `{month:02d}` = zero-pad to width 2, **d**ecimal
- **REPL** = **R**ead-**E**val-**P**rint **L**oop — exit with `exit()` / Ctrl+D (Ctrl+C only interrupts)
- **pip** = "**p**ip **i**nstalls **p**ackages" (recursive acronym)

### uv (by Astral, 10–100× faster than pip)
| Command | Does |
|---|---|
| `uv init --no-package` | new simple project — use for pipelines (plain `uv init` creates a package → `uv sync` fails without `src/`) |
| `uv add pandas` | install **and** record in `pyproject.toml` |
| `uv pip install pandas` | install only, not tracked |
| `uv sync` | set up the env from the lockfile (runs nothing) |
| `uv sync --locked --no-install-project` | strict lockfile, deps only — for Docker |
| `uv run python x.py` | sync + run |
- `--no-package` → `uv init`; `--no-install-project` → `uv sync`. Don't mix them up
- uv reads `pyproject.toml` from the **current directory**
- `pyproject.toml` = the shopping list · `uv.lock` = the receipt (like `package-lock.json`, `poetry.lock`)
- Already have `[build-system]`/`[project.scripts]` and don't need them? Delete them and run `uv lock`
- `.venv/bin/python` — point VS Code at it ("Python: Select Interpreter")

---

## 5. Terraform & GCP

### Big picture
- **IaC** = **I**nfrastructure **a**s **C**ode. **Terraform** (HashiCorp) is **declarative**: describe the end state, it works out the steps
- Docker = reproducible *software*; Terraform = reproducible *cloud infrastructure*
- **HCL** = **H**ashiCorp **C**onfiguration **L**anguage (`.tf` files). Terraform is a standalone Go binary — not in a venv
- **Idempotent**: running `apply` twice creates no duplicates

### Workflow
| Command | Does |
|---|---|
| `terraform init` | **init**ialize — download providers, create `.terraform/` |
| `terraform plan` | dry run. `+` create · `-` destroy · `~` modify · `-/+` replace |
| `terraform apply` | make it real (asks `yes`) |
| `terraform destroy` | delete everything it manages — run it when done (cost) |
| `terraform fmt` / `validate` | **f**or**m**a**t** / check syntax |

### Building blocks
- `terraform {}` → `required_version` + required providers (pin versions)
- `provider` = plugin that talks to a platform **API** (**A**pplication **P**rogramming **I**nterface): `google`, `aws`, `azurerm`, `microsoft/fabric`, Snowflake, Databricks…
- `resource "type" "name" {}` · `variable` (in `variables.tf`, use as `var.name`) · `output` · `data` (read-only lookup)
- Reference: `resource_type.name.attribute`

### State — the most important concept
- `terraform.tfstate` maps code → real resources. Plan = compare **code vs state vs reality**
- ⚠️ Never commit state or keys (state can hold secrets). `.gitignore`: `.terraform/`, `*.tfstate*`, `*.tfvars`, `*.json`, `crash.log`
- Teams: a **remote backend** (GCS bucket) for sharing and locking
- **Drift** = someone changed the console → code ≠ reality
  - `terraform plan -refresh-only` shows it; `apply -refresh-only` writes reality into state (`terraform refresh` is deprecated). Code is unchanged — the next plan still enforces the code
  - `terraform import` / `import {}` block (≥ 1.5) adopts an existing resource (still needs a `resource` block)
  - `terraform state rm <addr>` = stop managing a resource without deleting it
  - Rule: every infra change goes through code

### GCP auth & course resources
- **Service Account** = non-human identity with **IAM** (**I**dentity and **A**ccess **M**anagement) roles — **least privilege**: Storage Admin + BigQuery Admin
- JSON key **outside the repo** (`~/.gcp/`) → `export GOOGLE_APPLICATION_CREDENTIALS=~/.gcp/key.json`
- **GCS** (**G**oogle **C**loud **S**torage) bucket = **data lake** (raw files); names are globally unique
- **BigQuery** dataset = **data warehouse**
- Ecosystem: tenv / tfswitch (version managers), TFLint, terraform-docs, Infracost, Terragrunt, OpenTofu (open-source fork)

---

## Key patterns for data engineering
- Secrets/config in **env vars**, never hardcoded or committed
- **Pin versions** everywhere: images, Python, packages (lockfiles), Terraform
- Reproducible environments: Dockerfile (software), Terraform (infra) — never manual installs or console clicks
- `slim` images · split-COPY · exec-form ENTRYPOINT · named volumes for DBs
- Compose for multi-service setups; one container = one purpose
- Notebook → script → Docker; CLI args for flexible pipelines
- `plan` before `apply`; `destroy` when done
