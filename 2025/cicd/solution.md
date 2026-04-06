# Week 6: Jenkins CI/CD Basics & Advanced Challenge — Solution

---

## Task 1: Create a Jenkins Pipeline Job for CI/CD

### Jenkinsfile

```groovy
pipeline {
    agent any

    environment {
        IMAGE_NAME = "your-username/sample-app"
        IMAGE_TAG  = "v1.0"
    }

    stages {

        // ── Stage 1: Build ─────────────────────────────────────────
        // Compiles/packages the application and builds a Docker image.
        // This is the first gate — if the build fails, nothing else runs.
        stage('Build') {
            steps {
                echo "Building Docker image: ${IMAGE_NAME}:${IMAGE_TAG}"
                sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .'
            }
        }

        // ── Stage 2: Test ──────────────────────────────────────────
        // Runs the test suite inside a temporary container.
        // The container is removed after tests complete (--rm flag).
        stage('Test') {
            steps {
                echo "Running unit tests..."
                sh 'docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} python -m pytest tests/ -v'
            }
        }

        // ── Stage 3: Deploy ────────────────────────────────────────
        // Deploys the image to the target environment.
        // Stops any existing container on port 8080 before starting a new one.
        stage('Deploy') {
            steps {
                echo "Deploying application..."
                sh '''
                    docker stop sample-app || true
                    docker rm sample-app   || true
                    docker run -d --name sample-app -p 8080:80 ${IMAGE_NAME}:${IMAGE_TAG}
                    docker ps | grep sample-app
                '''
            }
        }
    }

    // ── Post Actions ───────────────────────────────────────────────
    // Always runs regardless of build outcome.
    post {
        success {
            echo "Pipeline completed successfully."
        }
        failure {
            echo "Pipeline failed. Check console output for details."
        }
        always {
            echo "Cleaning up workspace..."
            cleanWs()
        }
    }
}
```

### Stage Explanations

| Stage | Purpose |
|---|---|
| **Build** | Compiles code and packages it as a Docker image. First gate — broken builds stop here. |
| **Test** | Runs automated tests in an isolated container. Catches regressions before deployment. |
| **Deploy** | Ships the verified image to the target environment. |
| **post** | Cleanup and notification regardless of outcome. |

### Interview Questions

**Q: How do declarative pipelines streamline CI/CD compared to scripted pipelines?**

Declarative pipelines enforce a predefined structure (`pipeline { stages { stage { steps {} } } }`) that is easier to read, validate, and onboard new team members to. Jenkins can also perform syntax validation before execution. Scripted pipelines offer more flexibility (full Groovy), but that power comes with complexity and more room for error. For most CI/CD use cases, declarative pipelines are the right choice.

**Q: What are the benefits of breaking the pipeline into distinct stages?**

Distinct stages provide visibility — you can see exactly which stage failed in the Jenkins UI without reading hundreds of lines of logs. They also enable selective re-runs (restart from a failed stage), parallel execution, and conditional logic (`when` blocks). Each stage acts as a quality gate, preventing bad code from advancing to the next phase.

---

## Task 2: Multi-Branch Pipeline for Microservices

### Pipeline Design

In a microservices architecture, each service lives in its own directory (or repo). A multi-branch pipeline scans for branches matching a pattern and runs a `Jenkinsfile` found in each branch automatically.

### Jenkinsfile (Microservices with Parallel Stages)

```groovy
pipeline {
    agent any

    stages {

        // ── Checkout ───────────────────────────────────────────────
        stage('Checkout') {
            steps {
                checkout scm
                echo "Building branch: ${env.BRANCH_NAME}"
            }
        }

        // ── Build & Test (Parallel) ────────────────────────────────
        // Run builds for multiple services simultaneously to save time.
        stage('Build & Test') {
            parallel {

                stage('Service A') {
                    steps {
                        dir('service-a') {
                            sh 'docker build -t your-username/service-a:${env.BUILD_NUMBER} .'
                            sh 'docker run --rm your-username/service-a:${env.BUILD_NUMBER} npm test'
                        }
                    }
                }

                stage('Service B') {
                    steps {
                        dir('service-b') {
                            sh 'docker build -t your-username/service-b:${env.BUILD_NUMBER} .'
                            sh 'docker run --rm your-username/service-b:${env.BUILD_NUMBER} npm test'
                        }
                    }
                }
            }
        }

        // ── Deploy ─────────────────────────────────────────────────
        // Only deploy when on main branch — feature branches are built and tested only.
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                echo "Deploying all services to production..."
                sh 'docker-compose -f docker-compose.prod.yml up -d'
            }
        }
    }
}
```

### How Multi-Branch Pipelines Help Microservices

- Each feature branch gets its own isolated pipeline run — developers get fast feedback without affecting `main`.
- The `when { branch 'main' }` guard ensures only stable, reviewed code reaches production.
- PR builds run automatically when the Multibranch plugin detects a pull request, enabling mandatory CI checks before merge.
- Different services build in parallel, cutting total pipeline time significantly.

### Interview Questions

**Q: How does a multi-branch pipeline improve CI for microservices?**

Every branch gets its own build history and status. Teams working on different services can push to feature branches and get instant feedback without coordinating with other teams. Jenkins automatically creates and deletes pipeline runs as branches are created and removed, keeping the system self-managing.

**Q: What challenges arise when merging feature branches in a multi-branch pipeline?**

The most common challenge is **merge conflicts** when two branches have modified the same `Jenkinsfile` or shared configuration. Another challenge is **environment drift** — a feature branch may have been open long enough that `main` has advanced significantly, causing integration failures on merge. Regular rebasing and short-lived feature branches mitigate this.

---

## Task 3: Configure and Scale Jenkins Agents/Nodes

### Agent Configuration

**Step 1: Add a new agent in Jenkins UI**
- Go to **Manage Jenkins → Manage Nodes and Clouds → New Node**.
- Enter a node name (e.g., `linux-agent-01`), select **Permanent Agent**.
- Set **Remote root directory** (e.g., `/home/jenkins`), **Labels** (e.g., `linux`), and **Launch method** (SSH or JNLP).

**Step 2: Docker-based agent (recommended for reproducibility)**

```groovy
// In your Jenkinsfile, target agents by label
pipeline {
    agent none   // No default agent — each stage picks its own

    stages {

        stage('Build on Linux') {
            agent { label 'linux' }
            steps {
                sh 'docker build -t sample-app:latest .'
            }
        }

        stage('Test on Windows') {
            agent { label 'windows' }
            steps {
                bat 'dotnet test'
            }
        }
    }
}
```

**Step 3: Run parallel jobs across agents**

```groovy
stage('Cross-Platform Tests') {
    parallel {
        stage('Linux Tests') {
            agent { label 'linux' }
            steps { sh './run-tests.sh' }
        }
        stage('Windows Tests') {
            agent { label 'windows' }
            steps { bat 'run-tests.bat' }
        }
    }
}
```

### Verifying Agent Status

```bash
# Check agent connectivity from Jenkins controller
# Jenkins UI: Manage Jenkins → Manage Nodes → <node> → Log

# If using Docker agents, verify the container is running
docker ps | grep jenkins-agent
```

### Interview Questions

**Q: What are the benefits and challenges of distributed agents?**

Benefits: parallel execution cuts build times dramatically; workloads are isolated so one failing job can't starve others; different agents can run platform-specific builds (Linux containers vs. Windows binaries). Challenges: agent connectivity issues (network, firewall, SSH key rotation); keeping agents in sync (same tool versions, same Docker daemon version); and orchestrating secrets/credentials across agents securely.

**Q: How do you ensure jobs run on the correct agent?**

Use the `label` directive in the `agent` block. Labels are assigned when configuring the node and can be compound (`linux && docker && gpu`). Jenkins matches the label expression against available online agents and queues the job until a matching agent is free.

---

## Task 4: Implement RBAC in a Multi-Team Environment

### Configuration Steps

**Step 1: Install the Role Strategy Plugin**
- Manage Jenkins → Manage Plugins → Available → search "Role-based Authorization Strategy" → Install.

**Step 2: Enable Role-Based Strategy**
- Manage Jenkins → Configure Global Security → Authorization → **Role-Based Strategy**.

**Step 3: Define Roles**
- Manage Jenkins → Manage and Assign Roles → **Manage Roles**.

| Role | Permissions |
|---|---|
| **Admin** | All permissions |
| **Developer** | Job: Build, Cancel, Read; View: Read |
| **Tester** | Job: Read, Workspace; View: Read |
| **Ops** | Job: Build, Deploy, Read; Agent: Connect; View: Read |

**Step 4: Create Users and Assign Roles**
- Manage Jenkins → Manage Users → Create User (e.g., `dev-alice`, `tester-bob`).
- Manage Jenkins → Manage and Assign Roles → **Assign Roles** → map users to roles.

**Step 5: Verify**

Log in as each user and confirm they can only perform actions within their role. A Developer should see and trigger jobs but not access system configuration. A Tester should be able to read job output but not trigger builds.

### Importance of RBAC

Without RBAC, any authenticated user could delete jobs, modify system settings, or push to production — accidentally or maliciously. RBAC enforces the **principle of least privilege**: each person has exactly the access they need and nothing more.

**Risk scenario:** Without RBAC, a junior developer with access to the Jenkins controller could accidentally delete all production pipelines or trigger a deployment to production mid-sprint. With RBAC, that developer's account can only build and read — destructive operations require an Ops or Admin account.

### Interview Questions

**Q: Why is RBAC essential in CI/CD?**

CI/CD pipelines have direct pathways to production systems. A misconfigured or compromised account with broad Jenkins access could deploy malicious code, exfiltrate secrets, or destroy build history. RBAC limits the blast radius of any single compromised account or human error.

**Q: Scenario where inadequate RBAC causes security issues:**

A contractor is given full Jenkins Admin access "temporarily" to set up a job. They leave the company but the account is never disabled. An attacker who compromises their email gains full Jenkins access, injects malicious steps into a production pipeline, and exfiltrates database credentials stored as Jenkins secrets.

---

## Task 5: Jenkins Shared Library

### Repository Structure

```
jenkins-shared-library/
├── vars/
│   ├── buildDockerImage.groovy    # Reusable build step
│   ├── runTests.groovy            # Reusable test step
│   └── sendNotification.groovy   # Reusable notification step
└── src/
    └── org/
        └── devops/
            └── Utils.groovy       # Utility class
```

### `vars/buildDockerImage.groovy`

```groovy
// Callable as: buildDockerImage(imageName: 'user/app', tag: 'v1.0')
def call(Map config = [:]) {
    String imageName = config.imageName ?: error("imageName is required")
    String tag       = config.tag       ?: 'latest'

    echo "Building Docker image: ${imageName}:${tag}"
    sh "docker build -t ${imageName}:${tag} ."
}
```

### `vars/runTests.groovy`

```groovy
// Callable as: runTests(imageName: 'user/app', tag: 'v1.0', command: 'pytest')
def call(Map config = [:]) {
    String imageName = config.imageName ?: error("imageName is required")
    String tag       = config.tag       ?: 'latest'
    String command   = config.command   ?: 'npm test'

    echo "Running tests in container: ${imageName}:${tag}"
    sh "docker run --rm ${imageName}:${tag} ${command}"
}
```

### `vars/sendNotification.groovy`

```groovy
// Callable as: sendNotification(status: 'SUCCESS', recipients: 'team@company.com')
def call(Map config = [:]) {
    String status     = config.status     ?: 'UNKNOWN'
    String recipients = config.recipients ?: 'devops@company.com'

    emailext(
        subject: "[${status}] ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
        body: """
            Build status: ${status}
            Job: ${env.JOB_NAME}
            Build: #${env.BUILD_NUMBER}
            URL: ${env.BUILD_URL}
        """,
        to: recipients
    )
}
```

### Jenkinsfile Using the Shared Library

```groovy
@Library('jenkins-shared-library') _

pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                buildDockerImage(imageName: 'your-username/sample-app', tag: 'v1.0')
            }
        }
        stage('Test') {
            steps {
                runTests(imageName: 'your-username/sample-app', tag: 'v1.0', command: 'pytest tests/')
            }
        }
    }

    post {
        always {
            sendNotification(status: currentBuild.currentResult, recipients: 'team@company.com')
        }
    }
}
```

### Registering the Library in Jenkins

- Manage Jenkins → Configure System → **Global Pipeline Libraries** → Add.
- Name: `jenkins-shared-library`, Default version: `main`, Retrieval method: **Modern SCM → Git**, enter the repository URL.

### Interview Questions

**Q: How do shared libraries improve maintainability?**

Instead of copy-pasting the same Docker build logic across 20 Jenkinsfiles, it lives in one place. When the build process changes (e.g., adding a new flag), you update one file and every pipeline picks it up on the next run. This also ensures consistency — every team uses the same tested, reviewed deployment function rather than their own potentially divergent versions.

**Q: Ideal function for a shared library:**

A `deployToKubernetes(namespace, imageTag)` function that handles `kubectl set image`, waits for rollout to complete, and rolls back automatically if the new deployment fails health checks. Every team needs deployment logic; centralizing it in a shared library means the rollback safety net is never accidentally omitted.

---

## Task 6: Integrate Vulnerability Scanning with Trivy

### Updated Jenkinsfile Stage

```groovy
stage('Vulnerability Scan') {
    steps {
        echo "Scanning Docker image for vulnerabilities..."
        sh '''
            # Install Trivy if not present on agent
            which trivy || (
                curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
            )

            # Run the scan — exit non-zero if CRITICAL vulnerabilities found
            trivy image \
                --exit-code 1 \
                --severity CRITICAL,HIGH \
                --no-progress \
                your-username/sample-app:v1.0
        '''
    }
}
```

### Understanding Trivy Output

```
your-username/sample-app:v1.0 (debian 12.5)
Total: 12 (CRITICAL: 1, HIGH: 4, MEDIUM: 5, LOW: 2)

┌──────────────────────┬───────────────┬──────────┬───────────────────┬──────────────────┬──────────────────────────────────┐
│      Library         │ Vulnerability │ Severity │ Installed Version │   Fixed Version  │             Title                │
├──────────────────────┼───────────────┼──────────┼───────────────────┼──────────────────┼──────────────────────────────────┤
│ libssl3              │ CVE-2024-XXXX │ CRITICAL │ 3.0.11-1          │ 3.0.13-1         │ OpenSSL: buffer overflow in ...  │
│ libc6                │ CVE-2024-YYYY │ HIGH     │ 2.36-9            │ 2.36-9+deb12u4   │ glibc: heap-based buffer ...     │
└──────────────────────┴───────────────┴──────────┴───────────────────┴──────────────────┴──────────────────────────────────┘
```

### Remediation Steps

| Severity | Action |
|---|---|
| **Critical** | Immediately update the base image or the specific package. Rebuild and re-scan before merging. |
| **High** | Schedule fix within current sprint. Pin the affected package to the fixed version in your Dockerfile. |
| **Medium/Low** | Track in issue backlog. Accept risk if no exploit path exists; document the decision. |

**Example fix in Dockerfile:**
```dockerfile
FROM python:3.12-slim

# Explicitly upgrade vulnerable packages identified by Trivy
RUN apt-get update && apt-get upgrade -y libssl3 libc6 && rm -rf /var/lib/apt/lists/*
```

### Interview Questions

**Q: Why integrate vulnerability scanning into CI/CD?**

Catching vulnerabilities at build time is dramatically cheaper than discovering them in production. Automated scanning ensures security is not a manual afterthought — every image is verified before it can reach any environment. It also provides an audit trail showing that security checks were performed for every release.

**Q: How does Trivy improve Docker image security?**

Trivy scans OS packages, language-specific dependencies (pip, npm, gem), and checks against multiple CVE databases (NVD, GitHub Advisory, OSV). Unlike manual audits, it runs in seconds and integrates directly into the pipeline, making it easy to enforce a policy of "no Critical CVEs reach production."

---

## Task 7: Dynamic Pipeline Parameterization

### Parameterized Jenkinsfile

```groovy
pipeline {
    agent any

    // ── Runtime Parameters ─────────────────────────────────────────
    parameters {
        string(
            name: 'TARGET_ENV',
            defaultValue: 'staging',
            description: 'Deployment target environment (staging / production)'
        )
        string(
            name: 'APP_VERSION',
            defaultValue: '1.0.0',
            description: 'Application version to build and deploy'
        )
        booleanParam(
            name: 'RUN_TESTS',
            defaultValue: true,
            description: 'Whether to run the test suite'
        )
        choice(
            name: 'LOG_LEVEL',
            choices: ['INFO', 'DEBUG', 'WARN'],
            description: 'Log verbosity for this run'
        )
    }

    environment {
        IMAGE_TAG = "${params.APP_VERSION}-${params.TARGET_ENV}"
    }

    stages {

        stage('Build') {
            steps {
                echo "Building v${params.APP_VERSION} for ${params.TARGET_ENV}..."
                sh "docker build -t your-username/sample-app:${IMAGE_TAG} ."
            }
        }

        stage('Test') {
            // Skip tests if the RUN_TESTS parameter is false
            when {
                expression { return params.RUN_TESTS }
            }
            steps {
                echo "Running tests with LOG_LEVEL=${params.LOG_LEVEL}..."
                sh "docker run --rm -e LOG_LEVEL=${params.LOG_LEVEL} your-username/sample-app:${IMAGE_TAG} pytest"
            }
        }

        stage('Deploy') {
            steps {
                echo "Deploying ${IMAGE_TAG} to ${params.TARGET_ENV}..."
                sh """
                    docker stop sample-app-${params.TARGET_ENV} || true
                    docker rm   sample-app-${params.TARGET_ENV} || true
                    docker run -d \
                        --name sample-app-${params.TARGET_ENV} \
                        -p 8080:80 \
                        -e ENV=${params.TARGET_ENV} \
                        your-username/sample-app:${IMAGE_TAG}
                """
            }
        }
    }
}
```

### Triggering with Parameters

```bash
# Via Jenkins CLI
java -jar jenkins-cli.jar -s http://localhost:8080 build sample-pipeline \
  -p TARGET_ENV=production \
  -p APP_VERSION=2.1.0 \
  -p RUN_TESTS=true
```

Or via the Jenkins UI: **Build with Parameters → fill in the form → Build**.

### Interview Questions

**Q: How does parameterization improve CI/CD flexibility?**

The same Jenkinsfile can deploy to staging and production without any code changes — just a different parameter value. This eliminates environment-specific Jenkinsfiles that drift over time. Parameters also enable on-demand deployments where an operator can choose the exact version to roll out, useful during hotfixes.

**Q: Scenario where dynamic parameters are critical:**

A hotfix needs to be deployed to production immediately. The pipeline parameter `APP_VERSION=1.9.1-hotfix` combined with `TARGET_ENV=production` and `RUN_TESTS=false` (tests already passed on the hotfix branch) allows ops to deploy the specific fix in minutes without triggering a full test suite run.

---

## Task 8: Email Notifications for Build Events

### SMTP Configuration

1. **Manage Jenkins → Configure System → Extended E-mail Notification**
2. Set **SMTP server** (e.g., `smtp.gmail.com`), **SMTP Port** (`587`), enable **Use TLS**.
3. Add credentials: **Jenkins credentials** of type "Username and Password" with your email and app password.
4. Set **Default user email suffix** (e.g., `@yourcompany.com`).
5. Send a test email to verify connectivity.

### Updated Jenkinsfile with Notifications

```groovy
pipeline {
    agent any

    stages {
        stage('Build')  { steps { sh 'docker build -t sample-app:latest .' } }
        stage('Test')   { steps { sh 'docker run --rm sample-app:latest pytest' } }
        stage('Deploy') { steps { sh 'docker run -d -p 8080:80 sample-app:latest' } }
    }

    post {
        // Notify on every build outcome
        always {
            emailext (
                subject: "[${currentBuild.currentResult}] ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                body: """
                    <h2>Build ${currentBuild.currentResult}</h2>
                    <p><strong>Job:</strong> ${env.JOB_NAME}</p>
                    <p><strong>Build:</strong> #${env.BUILD_NUMBER}</p>
                    <p><strong>Duration:</strong> ${currentBuild.durationString}</p>
                    <p><a href="${env.BUILD_URL}">View full build log</a></p>
                """,
                mimeType: 'text/html',
                recipientProviders: [
                    [$class: 'DevelopersRecipientProvider'],   // Committer who triggered the build
                    [$class: 'RequesterRecipientProvider']     // Person who manually triggered the build
                ]
            )
        }

        // Additional alert only on failure
        failure {
            emailext (
                subject: "URGENT: Build Failed — ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Build failed. Immediate attention required.\n\n${env.BUILD_URL}console",
                to: 'oncall@yourcompany.com'
            )
        }
    }
}
```

### Troubleshooting Email Issues

| Problem | Likely Cause | Fix |
|---|---|---|
| No email sent | SMTP credentials wrong | Re-enter credentials; check app password (not account password) for Gmail |
| Connection refused | Wrong port or TLS setting | Try port 587 with TLS, or 465 with SSL |
| Email sent but not received | Spam filter | Whitelist Jenkins server IP; use a company SMTP relay |
| `javax.mail` exception in logs | Missing email plugin | Install "Email Extension Plugin" from Plugin Manager |

### Interview Questions

**Q: Advantages of automated email notifications in CI/CD:**

Teams are informed of failures without having to monitor a dashboard. Developers get feedback tied to their commits via `DevelopersRecipientProvider`. On-call rotation receives critical failure alerts immediately. Notification history in email threads provides a searchable audit trail of deployment events.

**Q: How to troubleshoot failed notifications:**

Check Jenkins system logs (`Manage Jenkins → System Log`) for SMTP errors. Test SMTP connectivity directly from the agent: `telnet smtp.gmail.com 587`. Verify credentials haven't expired. Use the **Test Configuration** button in Jenkins SMTP settings to send a test email before trusting the pipeline notification.

---

## Task 9: Troubleshooting, Monitoring & Advanced Debugging

### Simulating and Debugging a Failure

**Step 1: Introduce an error**

```groovy
stage('Build') {
    steps {
        sh 'docker build -t sample-app:latest .'   // Intentional typo below
        sh 'dokcer ps'                             // 'dokcer' → command not found
    }
}
```

**Step 2: Read the console output**

Jenkins marks the failed step in red in the Stage View. Click the stage → **Logs** to see the exact error:
```
+ dokcer ps
/var/jenkins_home/workspace/...: dokcer: command not found
```

**Step 3: Docker-level debugging**

```bash
# Check container logs
docker logs <container_id>

# Inspect a stopped container
docker inspect <container_id>

# Check Jenkins agent logs
docker logs jenkins-agent-01

# Check disk space (a full disk is a common silent killer)
df -h
```

**Step 4: Add debugging statements to Jenkinsfile**

```groovy
stage('Debug Info') {
    steps {
        // Print all environment variables available in this run
        sh 'env | sort'

        // Print tool versions to verify agent has expected dependencies
        sh 'docker --version && python3 --version && git --version'

        // Echo a key variable to confirm parameter is passed correctly
        echo "TARGET_ENV=${params.TARGET_ENV}"
        echo "Workspace: ${env.WORKSPACE}"
        echo "Build URL: ${env.BUILD_URL}"
    }
}
```

**Step 5: Use the Replay Feature**

In the Jenkins build view, click **Replay** to edit the Jenkinsfile inline and re-run the build without committing changes to source control. This is invaluable for rapid iteration when debugging a broken pipeline.

### Monitoring Jenkins

| Method | Description |
|---|---|
| **Jenkins system logs** | Manage Jenkins → System Log → All Jenkins Logs |
| **Build history** | Each job shows a timeline of pass/fail with duration trends |
| **Monitoring Plugin** | Exposes JVM metrics, thread count, build queue depth via a dashboard |
| **Prometheus + Grafana** | Install the Prometheus Metrics plugin; scrape `/prometheus` endpoint; build dashboards for build duration, queue time, failure rate |
| **CloudWatch / Datadog** | For cloud-hosted Jenkins, forward logs to a central observability platform |

### Key Metrics to Watch

- **Build queue time** — rising queue time means you need more agents.
- **Build failure rate** — a spike indicates either a code quality issue or a broken shared dependency.
- **Disk usage** — Jenkins workspaces and Docker images accumulate fast; set up `cleanWs()` in post blocks and a Docker image pruning cron job.
- **Executor utilization** — if all executors are always busy, scale out agents.

### Interview Questions

**Q: How would you approach troubleshooting a failing pipeline?**

Start with the **console output** for the exact error message. Narrow it down to the failing stage and step. Reproduce the failing command manually on the agent (`docker exec -it jenkins-agent bash`). Check whether the issue is environmental (missing tool, wrong credentials, network) or code-level (bad Jenkinsfile syntax, failing tests). Use **Replay** to test fixes without polluting the commit history.

**Q: Effective strategies for monitoring Jenkins in production:**

Export metrics to Prometheus and build a Grafana dashboard tracking queue depth, build duration p95, and daily failure rate. Set alerts on queue depth > 10 (signals agent shortage) and failure rate > 20% (signals systemic issue). Regularly archive and prune old builds to prevent disk exhaustion, which causes mysterious silent failures.

---

## Full Command Reference

```bash
# ── Docker (used in pipeline stages) ─────────────────────────────
docker build -t <user>/app:v1.0 .
docker run -d -p 8080:80 --name app <user>/app:v1.0
docker ps
docker logs <container_id>
docker stop app && docker rm app
docker images

# ── Trivy ──────────────────────────────────────────────────────────
trivy image --severity CRITICAL,HIGH --exit-code 1 <user>/app:v1.0
trivy image <user>/app:v1.0 > trivy_report.txt

# ── Jenkins CLI ────────────────────────────────────────────────────
java -jar jenkins-cli.jar -s http://localhost:8080 build <job> -p PARAM=value
java -jar jenkins-cli.jar -s http://localhost:8080 list-jobs

# ── Jenkins Agent (JNLP) ──────────────────────────────────────────
java -jar agent.jar -jnlpUrl http://jenkins:8080/computer/agent-1/slave-agent.jnlp \
     -secret <secret> -workDir /home/jenkins

# ── Debugging on Agent ────────────────────────────────────────────
docker exec -it jenkins-agent-01 bash
env | sort
df -h
docker --version && git --version
```

---

## Interview Question Summary

| Task | Key Takeaway |
|---|---|
| Pipeline | Declarative > Scripted for readability; stages = visibility + quality gates |
| Multi-branch | Auto-discovers branches; parallel stages cut build time; `when { branch }` guards production |
| Agents | Labels route jobs; distributed builds = speed + isolation; keep tool versions in sync |
| RBAC | Least privilege; Role Strategy Plugin; protects secrets and production pipelines |
| Shared Library | DRY principle for pipelines; centralized, versioned, tested common logic |
| Trivy | Shift-left security; block Critical CVEs before they reach production |
| Parameters | Same Jenkinsfile, multiple environments; enables targeted hotfix deployments |
| Notifications | Automatic feedback loop; `DevelopersRecipientProvider` ties alerts to commits |
| Debugging | Console output → Replay → Manual reproduction → Monitoring dashboards |

---
