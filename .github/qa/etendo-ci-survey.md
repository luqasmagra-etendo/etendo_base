# Survey: Etendo's current CI (Jenkins on Kubernetes)

Date: 2026-11-24. Source: the public `pipelines/Jenkinsfile` of each `etendosoftware` repository,
read from GitHub. Jira ticket: INT-7470.

**Out of scope (not visible from outside Jenkins):** job configuration and triggers, the pod
templates the pipelines inherit from (`jenkins-node-rx`, `jenkins-node-app`, `jenkins-ui-library`,
`default`), and any job for `etendo_core` / `etendo_base`, which have no public Jenkinsfile.

**Scope warning:** Etendo also runs Jenkins pipelines defined outside these public repositories,
including jobs that run core tests. They are not covered here. The findings below describe the 9
public files only, not Etendo's whole CI.

## Where it runs

9 of the 95 public repositories have a `pipelines/Jenkinsfile`. Each build is an ephemeral pod in
a Kubernetes cluster (namespace `jenkins2025`, from `com.etendoerp.docker/pipelines/utils/Agent.yaml`):

| Container | Image | Resources | Notes |
|---|---|---|---|
| `jnlp` | inherited template | — | Jenkins agent |
| `compiler` | `etendo/compiler_jenkins:1.0.7-jdk.17.0.13` | 1–2 CPU, 4–6.5 GB | `privileged: true`, `runAsUser: 0` |
| `dind` | `docker:25-dind` | 0.5–1 CPU, 1–2 GB | `privileged: true`; runs Postgres via Docker Compose |

Storage is `emptyDir`, so nothing survives between builds. Etendo itself is deployed with Docker
Compose (`com.etendoerp.docker`, `docker_environment`); Kubernetes is used only as Jenkins
infrastructure.

## What each pipeline validates

| Repository | Lines | Stages | Tests run |
|---|---:|---|---|
| `com.etendoerp.docker` | 412 | Wait for DinD → Stack Configuration → Build Environment | None: install + smartbuild + login page returns 200 |
| `com.etendoerp.tomcat` | 449 | Wait for DinD → Build Environment → Check Tomcat status | None: boot check |
| `etendo_schema_forge` | 644 | Preflight → Checkout & Align Branches → Offline Regeneration Check → Install Etendo (DB) → Schema Forge Regen → Export Database → Verify No Local Changes | None, but verifies that regeneration is reproducible. Uses Jira to resolve a branch's parent epic |
| `com.etendoerp.db.extended` | 637 | Build Environment Etendo Core → Run module tests → SonarQube → Partitioning test | `gradlew test --tests "com.etendoerp.db.extended.*"` + Jacoco |
| `com.etendoerp.etendorx` | 764 | RX Setup → Build Core → Code and Schema Verification → Build RX → Run module tests → SonarQube → Compare Coverage → Run Das Tests | `com.etendoerp.etendorx.*`; DAS tests one Gradle call per test |
| `com.etendorx.gradlepluginrx` | 238 | Setup → Run Tests | `gradlew test`; a failure marks the build **UNSTABLE, not FAILED** |
| `com.etendorx.workspace-ui` | 599 | Build → Tests → data-testid/format check → Storybook → SonarQube → Compare Coverage | pnpm unit tests with coverage + Storybook |
| `etendo_mobile` | 322 | Build → Tests → SonarQube → Compare Coverage | `yarn test` |
| `etendo_ui_library` | 304 | Build → Tests → Publish Next Version | `yarn test` |

Walkthrough of `com.etendoerp.docker`, the reference pattern for the backend repositories:
1. Wait up to 60s for DinD.
2. Pick the stack: a branch containing `-Y` or starting with `release` is a backport. Java, Tomcat and the core range come from Jenkins job variables. Tomcat is downloaded on every run.
3. Set the GitHub commit status to pending. Clone `etendo_base` and write `gradle.properties` with Nexus and `etendobot` credentials. Append the core dependency range to `build.gradle`.
4. Run `docker run hello-world`, pulled from Docker Hub on every run. Clone the module at the commit under test. `prepareConfig`, `setup`, `resources.up` (Postgres in DinD).
5. Read the container IP, rewrite `bbdd.url`, run `setup` again, then a fixed `sleep 15`. `install` + `smartbuild`.
6. Start Tomcat and poll the login page for up to 60s. Stop Tomcat, then `resources.stop`, a fixed `sleep 30`, and check the DB container stopped.
7. Post actions: `resources.down`; set the commit status to success or failure; send an HTML email on failure and on fixed.

## Cross-cutting findings

1. **None of the 9 files publishes test results.** There is no `junit` step, so failures exist only in the console log. There is no history, no trends and no flaky detection.
2. **None of the 9 files runs Etendo's core tests.** Backend repositories test only their own package. Core test jobs live outside the public repositories (see the scope warning above).
3. **Everything is sequential and built from scratch.** There is no `parallel` in any of the 9 files. Five of them install Etendo end to end on every build, with no cache.
4. **Copy-paste instead of a shared library.** The same blocks are repeated across repositories: `gradle.properties` generation, the Etendo build, the commit-status script (`build-update.sh` and a variant `update-status.sh`) and the HTML email.
5. **Feedback is a commit status plus an email.** No PR comment and no failure diagnosis.
6. **Security and robustness:**
   - privileged containers in `docker`, `tomcat` and `db.extended`;
   - tokens passed as script arguments, and `build-update.sh` echoes a URL containing the token (Jenkins usually masks bound credentials, but the pattern is risky);
   - fixed `sleep`s in three repositories;
   - most pipelines have no global timeout;
   - `disableConcurrentBuilds` makes pushes queue.

What works well: SonarQube and coverage compared against `main` in several repositories, and the
reproducibility check in `etendo_schema_forge` (an Etendo Go repository, cited only as a reference).

## Where the prototype fits

| Finding | Already proven in the GitHub prototype (`.github/qa/`) |
|---|---|
| Built from scratch every time | Build once + DB snapshot. In Jenkins: a shared image or snapshot of an installed Etendo reused by every module pipeline, which saves the most time |
| No parallelism | Ordered, time-balanced shards (`plan-shards.py`); one pod per shard |
| Invisible test results | `report.py` (summary, quarantine, flaky, test floor) + Jenkins `junit` |
| Core tests absent from the public pipelines | Full regression, 4788 tests, that blocks the change |
| Copy-paste | A Jenkins shared library for build, status and notification |
| Poor feedback | PR comment with deterministic risk and AI triage |
| Silent pipeline rot | Weekly canary (injected bug must be caught) |

Next step (INT-7470): a proposal for the Etendo team with prioritized improvements and the GitHub
runs as evidence. The proposal is kept outside this repository, because it also covers the
non-public pipelines.
