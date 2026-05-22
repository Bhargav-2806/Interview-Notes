# Docker Question 5 — What is the Difference Between CMD and ENTRYPOINT?

> **Section:** Docker &nbsp;|&nbsp; **Topic:** Dockerfile Instructions &nbsp;|&nbsp; **Level:** Beginner–Mid &nbsp;|&nbsp; **Frequency:** Very High
>
> _Part of the DevOps & DevSecOps Interview Preparation series._

---

## ❓ The Question

> **"What is the difference between CMD and ENTRYPOINT in Docker?"**

You may also hear this phrased as:

- "Can you override CMD at runtime? What about ENTRYPOINT?"
- "When would you use ENTRYPOINT instead of CMD?"
- "What happens when you combine CMD and ENTRYPOINT in the same Dockerfile?"
- "What is the difference between shell form and exec form for CMD?"
- "Why does my container ignore the command I pass on `docker run`?"

---

## 🎯 Why Interviewers Ask This

CMD vs ENTRYPOINT is one of the most commonly asked Docker questions at every level. Interviewers ask it to validate:

- You understand **how containers start** — what process runs as PID 1 and how Docker determines that.
- You know the **practical difference**: CMD is a default that can be replaced; ENTRYPOINT is a fixed executable.
- You understand **exec form vs shell form** — `["echo", "hello"]` vs `echo hello` — and why exec form is preferred in production.
- You can design a **flexible container** — ENTRYPOINT as the executable, CMD as the overridable default argument.
- For DevSecOps: PID 1 matters for **signal handling** — how a container responds to `docker stop` depends on how CMD/ENTRYPOINT are written.

> **The instant win:** Lead with the analogy — "ENTRYPOINT is the verb, CMD is the default noun." Then walk through the three forms: CMD alone, ENTRYPOINT alone, and combined. That structured answer shows you've actually debugged container startup behaviour.

---

## 📖 Key Terminologies

| Term | Plain-English Meaning |
|------|----------------------|
| **CMD** | Provides the default command and/or arguments when a container starts. Can be completely replaced by passing a command to `docker run`. |
| **ENTRYPOINT** | Defines the executable that always runs when a container starts. Cannot be overridden by `docker run` arguments — only by `--entrypoint`. |
| **Exec form** | JSON array syntax: `["executable", "arg1", "arg2"]`. Runs the executable directly — no shell involved. The process becomes PID 1. Preferred in production. |
| **Shell form** | Plain string syntax: `CMD echo hello`. Docker wraps it as `/bin/sh -c "echo hello"`. The shell becomes PID 1, not your executable — this breaks signal handling. |
| **PID 1** | The first process in a container. Docker sends SIGTERM to PID 1 on `docker stop`. If a shell is PID 1 (shell form), it may not forward SIGTERM to your application — causing force-kills after the timeout. |
| **Signal handling** | How a process responds to OS signals like SIGTERM (graceful shutdown), SIGKILL (force kill), SIGHUP (reload). Proper signal handling is critical for zero-downtime deployments. |
| **`docker run <image> <command>`** | Passing a command after the image name overrides `CMD` entirely. Example: `docker run my-app sh` replaces CMD with `sh`. |
| **`docker run --entrypoint`** | The only way to override ENTRYPOINT at runtime. Example: `docker run --entrypoint sh my-app`. |
| **`ENTRYPOINT` + `CMD` pattern** | Best practice: ENTRYPOINT sets the executable, CMD sets the default arguments. Users can override the arguments (CMD) without overriding the executable (ENTRYPOINT). |

---

## 🗣️ How to Answer (Structured)

**1) Lead with the one-line distinction:**
> "CMD provides the default command that runs when a container starts — but it can be completely replaced by passing a command to `docker run`. ENTRYPOINT defines the executable that always runs — you can't replace it without using the `--entrypoint` flag."

**2) Use the verb/noun analogy:**
> "A useful mental model: ENTRYPOINT is the verb — the thing that always happens. CMD is the default noun — the default thing it acts on. Together: `ENTRYPOINT ["python"]` + `CMD ["app.py"]` means 'always run Python, defaulting to app.py' — but a user can do `docker run my-app other_script.py` to override just the argument."

**3) Walk through the three forms:**
> "There are three patterns you see in Dockerfiles:
> - CMD alone: `CMD ["echo", "hello"]` — runs echo by default, user can replace entirely.
> - ENTRYPOINT alone: `ENTRYPOINT ["echo", "This is ENTRYPOINT"]` — always runs echo with this argument, no override without `--entrypoint`.
> - Combined: `ENTRYPOINT ["echo"]` + `CMD ["default arg"]` — always runs echo, but the argument defaults to 'default arg' and can be replaced."

**4) Explain exec form vs shell form:**
> "Both CMD and ENTRYPOINT can be written in two forms. Exec form — `["python", "app.py"]` — runs the process directly as PID 1. Shell form — `python app.py` — wraps it as `/bin/sh -c 'python app.py'`, making the shell PID 1. Always use exec form in production — shell form breaks signal handling because the shell doesn't forward SIGTERM to your application, so `docker stop` force-kills after the timeout."

**5) Close with the best practice:**
> "Best practice: use ENTRYPOINT for the fixed executable (your application binary or an entrypoint script), and CMD for the default arguments. This gives users the flexibility to override arguments at `docker run` time while ensuring the container always does what it's designed to do."

---

## 🔐 Security Perspective (DevSecOps)

CMD and ENTRYPOINT have direct security implications around process isolation and signal handling:

- **Exec form for proper signal handling and graceful shutdown** — Shell form (`CMD python app.py`) makes `/bin/sh` PID 1. When Kubernetes sends SIGTERM to drain a pod gracefully, the shell may not forward it to your Python process. The result: your app gets SIGKILL after the termination grace period — in-flight requests are dropped. Use exec form (`CMD ["python", "app.py"]`) so your app is PID 1 and handles SIGTERM directly.

- **Use an entrypoint script for security bootstrapping** — A common production pattern: `ENTRYPOINT ["./entrypoint.sh"]` where the script validates environment variables, checks required secrets are present, applies runtime configuration, then `exec "$@"` to hand off to CMD. The `exec "$@"` replaces the shell with the CMD process — keeping your app as PID 1 after bootstrapping.

- **Avoid running as root via ENTRYPOINT** — If `ENTRYPOINT ["python", "app.py"]` runs and the container's user is root (Docker default), the application runs as root. Combine ENTRYPOINT with a `USER appuser` instruction earlier in the Dockerfile. If using an entrypoint script that needs to start as root (e.g., for port binding), use `gosu` or `su-exec` to drop privileges before `exec`-ing the application.

- **Validate inputs in entrypoint scripts** — If CMD arguments can be supplied by users at `docker run` time, treat them as untrusted input in your entrypoint script. Validate arguments before passing them to the application, especially if they're used in file paths or commands.

- **`--entrypoint` in CI/CD should be intentional** — In CI pipelines, scanner containers often use `--entrypoint sh` to explore image contents. If your production image is being run with an overridden entrypoint in unexpected contexts, that's worth auditing. Immutable infrastructure means the ENTRYPOINT should match exactly what's in the Dockerfile.

> **One-liner for the room:** *"Use exec form, always. Shell form is the silent killer of graceful shutdowns — and a graceful shutdown is the difference between zero dropped requests and a bunch of angry users."*

---

## 🖼️ Visuals

### Mermaid — CMD vs ENTRYPOINT at Runtime

```mermaid
flowchart TD
    subgraph CMD_ONLY["CMD only\n(Dockerfile: CMD [\"echo\", \"hello\"])"]
        C1["docker run my-app"] --> C2["Runs: echo hello\n(CMD used)"]
        C3["docker run my-app echo world"] --> C4["Runs: echo world\n(CMD replaced entirely)"]
    end

    subgraph EP_ONLY["ENTRYPOINT only\n(Dockerfile: ENTRYPOINT [\"echo\", \"hello\"])"]
        E1["docker run my-app"] --> E2["Runs: echo hello\n(ENTRYPOINT used)"]
        E3["docker run my-app world"] --> E4["Runs: echo hello world\n(args appended to ENTRYPOINT)"]
        E5["docker run --entrypoint sh my-app"] --> E6["Runs: sh\n(ENTRYPOINT replaced)"]
    end

    subgraph COMBINED["ENTRYPOINT + CMD\n(ENTRYPOINT [\"echo\"] + CMD [\"default\"])"]
        CC1["docker run my-app"] --> CC2["Runs: echo default\n(CMD used as arg)"]
        CC3["docker run my-app custom"] --> CC4["Runs: echo custom\n(CMD overridden, ENTRYPOINT fixed)"]
    end
```

### Mermaid — Exec Form vs Shell Form (PID 1 Signal Handling)

```mermaid
flowchart TD
    subgraph SHELL_FORM["Shell Form ❌\nCMD python app.py"]
        S1["Docker runs:\n/bin/sh -c 'python app.py'"]
        S2["PID 1 = /bin/sh"]
        S3["PID 2 = python app.py"]
        S1 --> S2 --> S3
        S4["docker stop sends SIGTERM → /bin/sh (PID 1)"]
        S5["Shell may NOT forward SIGTERM to python"]
        S6["After grace period: SIGKILL sent\n→ In-flight requests DROPPED ❌"]
        S4 --> S5 --> S6
    end

    subgraph EXEC_FORM["Exec Form ✅\nCMD [\"python\", \"app.py\"]"]
        E1["Docker runs:\npython app.py directly"]
        E2["PID 1 = python app.py"]
        E1 --> E2
        E3["docker stop sends SIGTERM → python (PID 1)"]
        E4["Python handles SIGTERM gracefully\n→ Finishes in-flight requests ✅"]
        E3 --> E4
    end
```

---

## 📊 Quick Comparison — CMD vs ENTRYPOINT

| Feature | **CMD** | **ENTRYPOINT** |
|---------|---------|----------------|
| **Purpose** | Default command/arguments | Fixed executable that always runs |
| **Override with `docker run <cmd>`** | ✅ Yes — completely replaced | ❌ No — arguments are appended, not replaced |
| **Override with `--entrypoint`** | N/A | ✅ Yes — but must be explicit |
| **Combined behaviour** | Provides default args to ENTRYPOINT | Receives CMD as its arguments |
| **Exec form** | `CMD ["python", "app.py"]` | `ENTRYPOINT ["python"]` |
| **Shell form** | `CMD python app.py` | `ENTRYPOINT python` |
| **PID 1 (exec form)** | ✅ Your process | ✅ Your process |
| **PID 1 (shell form)** | ❌ `/bin/sh` | ❌ `/bin/sh` |
| **Best used for** | Flexible defaults, arguments | Application binary, entrypoint scripts |

---

## 🛠️ Hands-On: Commands & Configuration

### 1) Demonstrate CMD override

```bash
# Dockerfile with CMD only
cat > Dockerfile.cmd << 'EOF'
FROM ubuntu:22.04
CMD ["echo", "This is CMD"]
EOF

docker build -f Dockerfile.cmd -t cmd-demo .

# Default run — CMD executes
docker run cmd-demo
# Output: This is CMD

# Override CMD at runtime
docker run cmd-demo echo "CMD was overridden"
# Output: CMD was overridden

# Override CMD with a shell
docker run -it cmd-demo /bin/bash
# (drops into bash — CMD completely replaced)
```

### 2) Demonstrate ENTRYPOINT behaviour

```bash
# Dockerfile with ENTRYPOINT only
cat > Dockerfile.ep << 'EOF'
FROM ubuntu:22.04
ENTRYPOINT ["echo", "This is ENTRYPOINT"]
EOF

docker build -f Dockerfile.ep -t ep-demo .

# Default run — ENTRYPOINT executes
docker run ep-demo
# Output: This is ENTRYPOINT

# Try to pass a command — it APPENDS, not replaces
docker run ep-demo extra-arg
# Output: This is ENTRYPOINT extra-arg

# Override ENTRYPOINT — requires --entrypoint flag
docker run --entrypoint /bin/bash -it ep-demo
# (drops into bash)
```

### 3) Combined ENTRYPOINT + CMD (best practice)

```bash
cat > Dockerfile.combined << 'EOF'
FROM ubuntu:22.04
ENTRYPOINT ["echo"]
CMD ["This is the default message"]
EOF

docker build -f Dockerfile.combined -t combined-demo .

# Default run — ENTRYPOINT + CMD
docker run combined-demo
# Output: This is the default message

# Override CMD only — ENTRYPOINT stays
docker run combined-demo "Custom message"
# Output: Custom message

# Override ENTRYPOINT
docker run --entrypoint /bin/bash combined-demo
# (drops into bash — both replaced)
```

### 4) Real-world production pattern — entrypoint script

```bash
#!/bin/sh
# entrypoint.sh — security bootstrapping before app starts

set -e

# Validate required environment variables
if [ -z "$DB_HOST" ]; then
  echo "ERROR: DB_HOST environment variable is required" >&2
  exit 1
fi

if [ -z "$DB_PASSWORD" ]; then
  echo "ERROR: DB_PASSWORD environment variable is required" >&2
  exit 1
fi

echo "Starting application with DB_HOST=$DB_HOST"

# Drop privileges if running as root
if [ "$(id -u)" = "0" ]; then
  echo "WARNING: Running as root. Consider setting USER in Dockerfile."
fi

# Hand off to CMD — exec replaces shell with the app (PID 1 becomes the app)
exec "$@"
```

```dockerfile
# Dockerfile — production pattern
FROM python:3.11-alpine

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy entrypoint script and app
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
COPY src/ ./src/

# Non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# ENTRYPOINT = always run the bootstrap script
ENTRYPOINT ["/entrypoint.sh"]

# CMD = default application command (can be overridden)
CMD ["python", "src/app.py"]
```

```bash
# Build and run
docker build -t my-app:latest .

# Default run
docker run -e DB_HOST=localhost -e DB_PASSWORD=secret my-app:latest
# → runs entrypoint.sh then python src/app.py

# Override CMD to run a different script
docker run -e DB_HOST=localhost -e DB_PASSWORD=secret my-app:latest python src/worker.py
# → runs entrypoint.sh then python src/worker.py

# Debug: override ENTRYPOINT to skip bootstrapping
docker run -it --entrypoint sh my-app:latest
```

### 5) Demonstrate signal handling difference (exec vs shell form)

```bash
# Shell form — PID 1 is /bin/sh, SIGTERM not forwarded
cat > Dockerfile.shell << 'EOF'
FROM python:3.11-alpine
COPY app.py .
CMD python app.py   # shell form
EOF

# Exec form — PID 1 is python, SIGTERM handled directly
cat > Dockerfile.exec << 'EOF'
FROM python:3.11-alpine
COPY app.py .
CMD ["python", "app.py"]   # exec form
EOF

# Simple app that handles SIGTERM gracefully
cat > app.py << 'EOF'
import signal
import time
import sys

def handler(signum, frame):
    print("Received SIGTERM — shutting down gracefully")
    sys.exit(0)

signal.signal(signal.SIGTERM, handler)
print("App running...")
while True:
    time.sleep(1)
EOF

# Build and test both
docker build -f Dockerfile.shell -t shell-form .
docker build -f Dockerfile.exec -t exec-form .

# Run shell form container
docker run -d --name shell-test shell-form
sleep 2
time docker stop shell-test
# Waits full 10s timeout then force-kills (SIGTERM not forwarded)

# Run exec form container
docker run -d --name exec-test exec-form
sleep 2
time docker stop exec-test
# Stops in <1s (SIGTERM received and handled gracefully)
```

### 6) Check PID 1 inside a running container

```bash
# Check what process is PID 1 (reveals shell form vs exec form)
docker exec my-app ps aux | grep PID
# Or:
docker exec my-app cat /proc/1/cmdline | tr '\0' ' '
# Shell form output:  /bin/sh -c python app.py   ← shell is PID 1
# Exec form output:   python app.py               ← app is PID 1
```

---

## 🤖 AI & The New Trend (2024–2025)

> CMD and ENTRYPOINT are stable Docker primitives — but how they interact with modern orchestration and security tools is evolving.

### How the landscape is evolving

- **Kubernetes `command` and `args` override CMD/ENTRYPOINT** — In Kubernetes pod specs, `command` overrides ENTRYPOINT and `args` overrides CMD. This is a frequent source of confusion when migrating Docker Compose workloads to Kubernetes. AI-assisted migration tools (Kompose, Helm AI generators) now flag this mapping explicitly, helping teams avoid the mismatch.

- **Distroless and signal handling** — Google's distroless images have no shell at all, so shell form is impossible. This forces exec form adoption — a security win. Tools like `tini` (a minimal init system) are increasingly used as ENTRYPOINT to properly reap zombie processes and forward signals, especially in Java and Node.js containers that don't handle PID 1 responsibilities natively.

- **AI-generated Dockerfiles and CMD/ENTRYPOINT mistakes** — AI tools frequently generate Dockerfiles with shell form CMD (`CMD python app.py`) because it looks natural in training data. DevOps engineers reviewing AI-generated Dockerfiles should specifically check for shell form and convert to exec form — it's one of the most common AI Dockerfile mistakes.

- **OpenTelemetry SDK auto-instrumentation via ENTRYPOINT** — A 2024 pattern: ENTRYPOINT is used to inject OpenTelemetry auto-instrumentation wrappers before the application starts. For example: `ENTRYPOINT ["opentelemetry-instrument"]` + `CMD ["python", "app.py"]` — the OTel SDK instruments the app at startup without any code changes. This is signal-safe because both use exec form.

### Mention this in interviews:

> "One thing I always check in code reviews is whether Dockerfiles use exec form or shell form for CMD and ENTRYPOINT. Shell form is surprisingly common in AI-generated Dockerfiles, and it silently breaks graceful shutdown — which only shows up as dropped requests during rolling deployments in Kubernetes."

---

## ✅ Prerequisites (be solid on these first)

- **Dockerfile basics** — Know what `FROM`, `RUN`, `COPY`, `ENV` do before tackling CMD/ENTRYPOINT.
- **`docker run` syntax** — `docker run [options] <image> [command] [args]`. The `[command]` part is what overrides CMD.
- **Process fundamentals** — What PID 1 is, what SIGTERM and SIGKILL mean, what "graceful shutdown" means for a web server.
- **JSON array syntax** — `["python", "app.py"]` vs `["python app.py"]` — these are different! The array form passes each element as a separate argument; don't concatenate them into one string.
- **Basic shell scripting** — `exec "$@"` in an entrypoint script replaces the current shell with the arguments passed — this is the critical line that makes the entrypoint pattern work correctly.

---

## 📚 Further Reading (current docs)

- **Dockerfile CMD reference** — <https://docs.docker.com/engine/reference/builder/#cmd>
- **Dockerfile ENTRYPOINT reference** — <https://docs.docker.com/engine/reference/builder/#entrypoint>
- **Best practices — CMD and ENTRYPOINT** — <https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#cmd>
- **tini — minimal init for containers** — <https://github.com/krallin/tini>
- **Kubernetes command and args** — <https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/>
- **Docker signal handling guide** — <https://docs.docker.com/engine/reference/run/#signals>
- **KodeKloud lesson (video):** <https://learn.kodekloud.com/user/courses/devops-interview-preparation-course/module/955d2fcf-4c92-4480-b86e-081d67d83e88/lesson/9d51a058-3b10-4691-8bd8-548c60c946d2>

---

## 🔁 Related / Follow-up Questions (they often go here next)

1. **"What is exec form and shell form in Docker?"** → Exec form is JSON array syntax: `["python", "app.py"]`. Each array element is a separate argument; no shell involved. Shell form is plain text: `python app.py`. Docker wraps it as `/bin/sh -c "python app.py"`. The critical difference: exec form runs your process as PID 1; shell form makes `/bin/sh` PID 1, breaking signal forwarding.

2. **"What happens when you pass a command to `docker run` after the image name?"** → It replaces CMD entirely. `docker run my-app python other.py` — `other.py` replaces whatever CMD was in the Dockerfile. If ENTRYPOINT is set, the passed command becomes the argument to ENTRYPOINT instead. Understanding this distinction is critical for debugging unexpected container behaviour.

3. **"Why do my containers take 10 seconds to stop with `docker stop`?"** → This is the classic shell form signal handling problem. `docker stop` sends SIGTERM and waits (default 10s) for the container to exit. If PID 1 is `/bin/sh` (shell form), it ignores or doesn't forward SIGTERM to your app. After 10 seconds, Docker sends SIGKILL. Fix: switch CMD/ENTRYPOINT to exec form so your application is PID 1 and handles SIGTERM directly.

4. **"What does `exec "$@"` do in an entrypoint script?"** → It replaces the current shell process with the command passed as arguments (`"$@"`). Before this line, the shell is PID 1. After `exec "$@"`, the CMD process becomes PID 1 — the shell is gone. This is critical: without `exec`, the shell stays as PID 1 and your app is a child process, breaking signal handling.

5. **"How do CMD and ENTRYPOINT work in Kubernetes?"** → Kubernetes pod spec has `command` (maps to Docker's ENTRYPOINT) and `args` (maps to Docker's CMD). If you set `command` in Kubernetes, it overrides the Dockerfile's ENTRYPOINT. If you set `args`, it overrides CMD. This mapping is a common source of confusion — many teams accidentally override ENTRYPOINT when they only meant to override CMD.

6. **"Should I use `ENTRYPOINT` or `CMD` for my application?"** → Use both together. `ENTRYPOINT` for the fixed executable (your app binary or entrypoint script), `CMD` for the default arguments. This gives users the flexibility to run `docker run my-app --debug` to override the arguments without changing the executable. If you only use CMD, your container is too easy to repurpose for arbitrary commands.

7. **"What is `tini` and why do some Dockerfiles use it as an ENTRYPOINT?"** → `tini` is a minimal init process designed to be PID 1 in containers. It properly handles zombie process reaping (important for apps that spawn child processes) and signal forwarding. Used as: `ENTRYPOINT ["tini", "--"]` + `CMD ["python", "app.py"]`. It's built into Docker (`--init` flag on `docker run`) but many teams embed it directly in the image for Kubernetes compatibility.

8. **"What does `CMD []` (empty array) do?"** → An empty CMD clears any CMD inherited from a parent image (via `FROM`). If ENTRYPOINT is set, CMD is the argument list — an empty CMD means ENTRYPOINT runs with no arguments. This is used in base images that define ENTRYPOINT and want child images to provide their own CMD.

---

> 📌 **30-second interview summary:** CMD and ENTRYPOINT both define what runs when a container starts, but with different override behaviours. CMD is the default — `docker run my-app some-command` replaces it entirely. ENTRYPOINT is fixed — `docker run` arguments are appended to it, not replacing it (only `--entrypoint` replaces it). Best practice: combine them — `ENTRYPOINT` for the executable, `CMD` for overridable default arguments. Always use exec form (`["python", "app.py"]` not `python app.py`) so your application is PID 1 and handles SIGTERM correctly. Shell form silently breaks graceful shutdown, which causes dropped requests during Kubernetes rolling deployments.
