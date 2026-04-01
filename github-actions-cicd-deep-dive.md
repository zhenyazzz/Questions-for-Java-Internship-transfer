# CI/CD & GitHub Actions — Deep Dive for Middle Java Developer

Practical guide to CI/CD with focus on GitHub Actions architecture, real-world workflows, and interview-ready explanations.

---

## 1) CI/CD Fundamentals

### CI vs CD

**CI (Continuous Integration)**
- Automated build and test on every code change
- Fast feedback loop (did my change break anything?)
- Merge only green builds

**CD (Continuous Delivery vs Deployment)**
- **Delivery**: Software is always in a releasable state; deployment to production is a manual decision
- **Deployment**: Every successful build automatically goes to production

**Real-world flow:**
```
Developer push → CI Build + Test → Artifact created → Deploy to Staging (auto)
                                                    → Deploy to Production (manual approval)
```

### Why CI Matters in Real Teams

1. **Catches integration issues early** — conflicts surface immediately, not during release week
2. **Enables trunk-based development** — merge small changes frequently instead of long-lived branches
3. **Living documentation** — pipeline shows exactly how to build and test
4. **Code review safety** — PR checks prevent broken code from entering main branch
5. **Reproducibility** — same build process on every developer's machine and CI server

### What Happens on Push / Pull Request

**On Push to main:**
1. GitHub detects the push event
2. Triggers workflow(s) configured for `push` event on `main` branch
3. Runner spins up (or uses existing)
4. Code is checked out
5. Dependencies installed (possibly from cache)
6. Build executed
7. Tests run
8. Artifact uploaded (optional)
9. Deployment triggered (optional)

**On Pull Request:**
- Same process but with `pull_request` event trigger
- Results posted as PR status check
- Branch protection can block merge until checks pass

---

## 2) GitHub Actions Architecture

### Core Concepts

**Workflow**
- Top-level unit defined in `.github/workflows/*.yml`
- Triggered by events
- Contains one or more jobs

**Job**
- Group of steps that run on the same runner
- Jobs run in parallel by default (or sequentially via `needs`)
- Each job gets a fresh VM/container

**Step**
- Individual task within a job
- Steps run sequentially
- Can be either:
  - `uses`: reusable action from marketplace or repo
  - `run`: shell command

**Runner**
- Machine that executes jobs
- **GitHub-hosted**: Managed by GitHub (Ubuntu, Windows, macOS), free minutes included
- **Self-hosted**: Your own machine (useful for private networks, special hardware, larger resources)

```yaml
# Example structure
jobs:
  build:                    # Job name
    runs-on: ubuntu-latest  # Runner
    steps:                  # Steps list
      - uses: actions/checkout@v4    # Action
      - run: mvn clean install       # Shell command
```

### Events

Common triggers:
```yaml
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:        # Manual trigger from UI
  schedule:
    - cron: '0 2 * * *'     # Nightly builds
```

**Real-world tip**: Use path filters to avoid unnecessary runs:
```yaml
on:
  push:
    paths:
      - 'src/**'
      - 'pom.xml'
      - '.github/workflows/**'
```

---

## 3) Workflow File Structure

### YAML Anatomy

```yaml
name: Java CI                  # Workflow display name

on:                            # Triggers
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:                       # Job ID
    name: Build and Test       # Display name
    runs-on: ubuntu-latest     # Runner
    
    steps:
      # Step 1: Checkout code
      - name: Checkout
        uses: actions/checkout@v4
      
      # Step 2: Setup JDK
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      
      # Step 3: Build (run command)
      - name: Build with Maven
        run: mvn -B clean package -DskipTests
      
      # Step 4: Run tests
      - name: Test
        run: mvn test
        
      # Step 5: Upload artifact (action)
      - name: Upload JAR
        uses: actions/upload-artifact@v4
        with:
          name: application-jar
          path: target/*.jar
```

### `uses` vs `run`

**`uses`** — Reusable action (pre-built logic):
```yaml
- uses: actions/setup-java@v4    # Official action
- uses: owner/repo@v1            # Community or private action
```

**`run`** — Direct shell command:
```yaml
- run: mvn clean install
- run: |
    echo "Multi"
    echo "line"
    echo "command"
```

**Under the hood**: GitHub downloads the action's code and executes it. For `actions/checkout@v4`, it clones your repo into the runner's workspace.

---

## 4) Working with Dependencies

### Caching with `actions/cache`

**Why cache**: Maven/Gradle dependencies don't change often. Downloading them every build wastes time and bandwidth.

```yaml
- name: Cache Maven dependencies
  uses: actions/cache@v4
  with:
    path: ~/.m2/repository          # What to cache
    key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}  # Cache key
    restore-keys: |
      ${{ runner.os }}-m2-          # Fallback keys
```

**How it works**:
1. Before job starts, GitHub tries to restore cache by key
2. If hit — dependencies already present, instant restore
3. If miss — download dependencies, cache uploaded after successful job
4. Cache key includes hash of `pom.xml` — if dependencies change, new cache created

**Real-world Maven workflow with cache:**
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'              # Built-in caching since setup-java@v3!
      
      - name: Build
        run: mvn -B clean verify
```

**setup-java with cache** is the modern, simpler approach vs manual `actions/cache`.

### Gradle Example
```yaml
- name: Set up Gradle
  uses: gradle/actions/setup-gradle@v3
  with:
    gradle-home-cache-cleanup: true

- name: Build
  run: ./gradlew build
```

**Pitfall**: Don't cache build outputs (target/, build/) — they should be fresh each run. Cache only dependency repositories.

---

## 5) Environment Variables and Secrets

### Storing Secrets

Go to: Repository → Settings → Secrets and variables → Actions

Types:
- **Repository secrets**: Available to all workflows in repo
- **Environment secrets**: Scoped to specific deployment environments (staging, production)
- **Organization secrets**: Shared across repos in org

### Using in Workflow

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to staging
        run: |
          ./deploy.sh \
            --api-key "${{ secrets.API_KEY }}" \
            --db-url "${{ secrets.DB_URL }}"
        env:                          # Pass as env vars to script
          DEPLOY_USER: ${{ secrets.DEPLOY_USER }}
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```

**Security best practices:**
1. Never log secrets (GitHub masks them in output automatically)
2. Use environment secrets for production (requires approval)
3. Rotate secrets regularly
4. Never commit secrets to code (use `.env.example` instead)
5. Review workflow files in PRs carefully — malicious code can exfiltrate secrets

### Regular Variables

```yaml
env:                              # Workflow-level
  JAVA_VERSION: '21'
  MAVEN_OPTS: '-Xmx2g'

jobs:
  build:
    env:                          # Job-level
      GRADLE_OPTS: '-Dorg.gradle.daemon=false'
    steps:
      - run: echo "Building with Java $JAVA_VERSION"
```

---

## 6) Build Pipeline

### Complete Java Pipeline

```yaml
name: Build Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  lint-and-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'
      
      # Linting (Checkstyle, SpotBugs, etc.)
      - name: Run Checkstyle
        run: mvn checkstyle:check
        continue-on-error: true    # Don't fail build, just report
      
      # Compile
      - name: Compile
        run: mvn -B compile
      
      # Tests with coverage
      - name: Test
        run: mvn -B verify
      
      # Upload coverage to external service
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: target/site/jacoco/jacoco.xml

  integration-tests:
    runs-on: ubuntu-latest
    needs: lint-and-build          # Run only if build succeeds
    services:                      # Service containers
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'
      
      - name: Integration tests
        run: mvn -B verify -Pintegration-tests
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/test
```

**Real-world additions:**
- SonarQube scan for code quality
- OWASP dependency check for vulnerabilities
- Javadoc generation
- Docker image build and push

---

## 7) Parallelism

### Jobs vs Steps

**Steps**: Sequential execution on same runner (share filesystem, fast)
```yaml
steps:
  - run: mvn compile      # Step 1
  - run: mvn test         # Step 2 (waits for 1)
  - run: mvn package      # Step 3 (waits for 2)
```

**Jobs**: Parallel execution on different runners (fresh environment each)
```yaml
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps: [...]
  
  integration-tests:
    runs-on: ubuntu-latest
    steps: [...]          # Runs at same time as unit-tests!
  
  code-quality:
    runs-on: ubuntu-latest
    steps: [...]          # Also parallel
```

**Dependencies with `needs`:**
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps: [...]
  
  test:
    runs-on: ubuntu-latest
    needs: build           # Waits for 'build' job
    steps: [...]
  
  deploy:
    runs-on: ubuntu-latest
    needs: [build, test]   # Waits for both
    steps: [...]
```

**Pipeline visualization:**
```
build ──→ test ──→ deploy
    └────→ code-quality
    └────→ security-scan
```

**Optimization tip**: Run independent checks (linting, security scan, unit tests) in parallel to get faster feedback.

---

## 8) Matrix Builds

### Testing Multiple Java Versions

```yaml
strategy:
  matrix:
    java-version: [17, 21, 22]
    os: [ubuntu-latest, windows-latest]
    include:
      - java-version: 21
        os: macos-latest
        experimental: true
    exclude:
      - java-version: 22
        os: windows-latest

runs-on: ${{ matrix.os }}
steps:
  - uses: actions/setup-java@v4
    with:
      java-version: ${{ matrix.java-version }}
      distribution: 'temurin'
  - run: mvn test
```

**Why matrix builds matter:**
- Verify compatibility with multiple Java versions before users upgrade
- Catch OS-specific issues (Windows path separators, line endings)
- Test with different dependency versions (Spring Boot 2.x vs 3.x)

**Interview insight**: Matrix builds multiply your job count (3 Java × 2 OS = 6 jobs). Be mindful of parallel job limits on free tiers.

---

## 9) Artifacts

### Saving Build Outputs

```yaml
- name: Upload build artifacts
  uses: actions/upload-artifact@v4
  with:
    name: java-app-${{ github.sha }}
    path: |
      target/*.jar
      target/*.war
    retention-days: 7        # Auto-delete after 7 days
```

**Use cases:**
- Pass JARs between jobs (build in one job, test in another, deploy in third)
- Store test reports for debugging
- Archive release binaries
- Share coverage reports

### Downloading in Another Job

```yaml
jobs:
  deploy:
    needs: build
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: java-app-${{ github.sha }}
          path: ./artifacts
      
      - run: ls ./artifacts
```

**Under the hood**: GitHub stores artifacts in blob storage. They're not Git LFS or releases — temporary storage for CI/CD workflow.

---

## 10) Deployment Basics

### Deployment Approaches

**Option 1: Direct server deployment (simple)**
```yaml
- name: Deploy to Staging
  if: github.ref == 'refs/heads/develop'
  run: |
    scp target/app.jar user@staging-server:/opt/app/
    ssh user@staging-server "sudo systemctl restart app"
```

**Option 2: Docker-based**
```yaml
- name: Build and push Docker image
  run: |
    docker build -t ghcr.io/${{ github.repository }}:${{ github.sha }} .
    echo ${{ secrets.GITHUB_TOKEN }} | docker login ghcr.io -u ${{ github.actor }} --password-stdin
    docker push ghcr.io/${{ github.repository }}:${{ github.sha }}
```

**Option 3: Platform deploy (Heroku, Railway, etc.)**
```yaml
- uses: akhileshns/heroku-deploy@v3.12.14
  with:
    heroku_api_key: ${{ secrets.HEROKU_API_KEY }}
    heroku_app_name: myapp-staging
    heroku_email: user@example.com
```

### Staging vs Production

```yaml
jobs:
  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    environment: staging          # Can have protection rules
    steps: [...]
  
  deploy-production:
    if: github.ref == 'refs/heads/main'
    environment: production       # Requires approval!
    steps: [...]
```

**Environment protection rules:**
- Required reviewers before deployment
- Deployment branches limitation
- Wait timer

---

## 11) Best Practices

### Fast Pipelines

1. **Use caching aggressively** — dependencies, Gradle wrapper, Node modules
2. **Fail fast** — run quick checks (linting, compilation) before slow tests
3. **Conditional execution** — skip irrelevant workflows with path filters
4. **Parallel jobs** — run independent checks simultaneously
5. **Small artifacts** — don't upload unnecessary files

```yaml
# Skip if only docs changed
on:
  push:
    paths-ignore:
      - '**.md'
      - 'docs/**'
```

### Avoiding Unnecessary Runs

```yaml
# Don't run on WIP commits
if: "!contains(github.event.head_commit.message, '[skip ci]')"

# Don't run on draft PRs
if: github.event.pull_request.draft == false
```

### Readable Pipelines

1. **Name your steps** — `run: mvn test` vs `name: Run unit tests with Maven`
2. **Group related steps** — use YAML anchors or composite actions for reuse
3. **Comments** — explain non-obvious workarounds
4. **Consistent formatting** — 2-space indentation, blank lines between jobs

### Security Checklist

- Pin action versions (`@v4` not `@main`) — prevents supply chain attacks
- Review third-party actions before use
- Use least-privilege tokens
- No secrets in logs (GitHub masks automatically, but be careful with custom scripts)
- Enable branch protection with required status checks

---

## 12) Pitfalls and How to Avoid Them

### Long Builds

**Symptoms**: 15+ minutes per PR, developers bypassing CI

**Solutions**:
- Aggressive caching (check if cache actually hits with `actions/cache@v4` post step)
- Parallel jobs for independent tasks
- Test splitting (JUnit parallel execution, surefire/failsafe)
- Conditional testing (only run affected module tests)

**Debug cache misses:**
```yaml
- name: Debug cache
  run: |
    ls -la ~/.m2/repository 2>/dev/null || echo "No Maven cache"
```

### Wrong Cache Configuration

**Mistake**: Caching `target/` or `build/` directories
```yaml
# BAD - corrupts builds
path: |
  ~/.m2/repository
  target/
```

**Fix**: Cache only dependencies, never build outputs.

### Secret Leaks

**Mistake**: Printing environment in logs
```yaml
- run: env  # Shows all secrets in plaintext!
```

**Mistake**: Passing secrets to untrusted actions
```yaml
- uses: random-user/action@v1  # Who reviewed this?
  with:
    token: ${{ secrets.API_KEY }}  # Leaked to unknown code
```

### Flaky Tests

**Symptom**: Same code passes/fails randomly

**Causes**:
- Race conditions in tests
- External service dependencies
- Shared state between tests
- Time-based assertions

**Mitigation:**
```yaml
- name: Test with retries
  uses: nick-fields/retry@v2
  with:
    timeout_minutes: 10
    max_attempts: 3
    command: mvn test
```

**Better fix**: Rewrite flaky tests to be deterministic. Retries hide real problems.

### Self-Hosted Runner Issues

**Pitfalls**:
- Runner offline during deployment
- Disk full from old Docker images
- Stale dependency caches
- Security (runner has access to your network)

**Maintenance:**
```bash
# Automated cleanup job
- run: |
    docker system prune -f
    rm -rf ~/.m2/repository/com/yourcompany/*
```

---

## Interview Discussion

### Common Questions

**Q: How do you speed up a slow Maven build in GitHub Actions?**
**A**: Use `setup-java` with built-in caching, ensure cache key includes `pom.xml` hash, parallelize independent jobs (unit vs integration tests), run only affected module tests in PRs, and use Maven's `-T` parallel build option.

**Q: How do you handle secrets safely?**
**A**: Store in GitHub Secrets (environment-scoped for production), never log them, use least-privilege tokens, pin action versions to prevent supply chain attacks, and enable required reviewers for production deployments.

**Q: Difference between jobs and steps?**
**A**: Steps run sequentially on same runner (share files, fast). Jobs run on separate runners (parallel by default, isolated). Use steps for dependent operations, jobs for parallelizable work.

**Q: When would you use matrix builds?**
**A**: Testing multiple Java versions before rolling out upgrades, verifying OS-specific behavior (Windows vs Linux paths), or testing against multiple dependency versions (Spring Boot 2.x and 3.x).

**Q: How do you prevent deploying broken code?**
**A**: Branch protection with required status checks, mandatory PR reviews, staging environment that auto-deploys but production requires approval, and post-deploy health checks with automatic rollback.

---

## Summary

GitHub Actions organizes CI/CD into workflows triggered by events, containing jobs that run on runners, which execute steps sequentially. Jobs run in parallel by default, coordinated via `needs` for pipeline dependencies. Key optimizations include aggressive dependency caching (using `setup-java` built-in cache), parallel job execution for independent checks, and path filters to skip irrelevant runs. Secrets management requires environment-scoped secrets, least-privilege tokens, and careful action version pinning. Matrix builds verify compatibility across Java versions and operating systems before production rollout. Artifacts enable passing build outputs between jobs. Deployment strategies range from direct server deployment to Docker-based or platform-specific approaches, with staging auto-deployment and production requiring manual approval through environment protection rules. Common pitfalls include misconfigured caching, secret exposure, flaky tests masking real issues, and long build times that erode developer productivity.
