# Jenkins SRE Mastery – Step‑by‑Step Guide, Hands‑On Labs & Interview Q&A

> A practical, SRE‑focused Jenkins guide: install → harden → scale → automate → observe. Includes ready‑to‑run pipelines, JCasC samples, and troubleshooting playbooks.

---

## 1) What is Jenkins (in SRE terms)?
- **Purpose:** Orchestrate CI/CD reliably and repeatably. As SRE, you care about **availability, repeatability, security, metrics, and change safety**.
- **Core pieces:**
  - **Controller (Master):** UI, queue, scheduling, configuration.
  - **Agents (Workers):** Execute builds (static VMs, Docker, Kubernetes pods, cloud instances).
  - **Jobs/Pipelines:** Declarative or scripted Groovy pipelines.
  - **State:** `$JENKINS_HOME` (jobs, credentials, configs, plugins) – protect and back it up.
  - **Plugins:** Extend functionality (source of power *and* fragility). Manage carefully.

---

## 2) Install Jenkins (macOS, Linux, Docker, Kubernetes)

### A. macOS (Homebrew – LTS)
```bash
brew update
brew install jenkins-lts
# Apple Silicon: data lives in /opt/homebrew/var/jenkins_lts
# Intel:         /usr/local/var/jenkins_lts
brew services start jenkins-lts
open http://localhost:8080
# Unlock:
cat /opt/homebrew/var/jenkins_lts/secrets/initialAdminPassword  # Apple Silicon
# or
cat /usr/local/var/jenkins_lts/secrets/initialAdminPassword      # Intel
```

### B. Ubuntu/Debian (service)
```bash
sudo apt-get update && sudo apt-get install -y ca-certificates curl gnupg openjdk-17-jdk
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update && sudo apt-get install -y jenkins
sudo systemctl enable --now jenkins
```

### C. Docker (quick lab)
```bash
docker network create jenkins || true
mkdir -p $HOME/jenkins_home
chmod 777 $HOME/jenkins_home
# Controller
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v $HOME/jenkins_home:/var/jenkins_home \
  --restart unless-stopped \
  jenkins/jenkins:lts-jdk17
# Unlock
docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

### D. Kubernetes (production-ish)
- Use the official Helm chart `jenkinsci/jenkins`.
```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update
kubectl create ns jenkins || true
helm upgrade --install jenkins jenkins/jenkins -n jenkins \
  --set controller.jenkinsUrl=http://jenkins.example.com \
  --set controller.adminUser=admin \
  --set controller.adminPassword=admin123 \
  --set controller.installPlugins={kubernetes:4245.vb_134a_6a_5f12d,workflow-aggregator:596.v8c21c963d92d,git:5.2.2,configuration-as-code:1806.v0f6c3fe1d5ee,credentials:1311.vcf0a_900b_478b_} \
  --set controller.resources.requests.cpu=500m \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.memory=2Gi
```
- Pods as **ephemeral agents** via Kubernetes plugin → elastic, isolated, reproducible.

---

## 3) First Steps: Secure-by-default Jenkins
1. **Create admin, then a non-admin user** for daily work.
2. **Global Security**
   - Enable **Matrix-based security** (lock down to groups/roles).
   - Disable anonymous read.
3. **Credentials** (folder/job scoped Service Accounts).
4. **Disable the legacy CLI over remoting**; use HTTP/SSH CLI only.
5. **Restrict plugins** to what you actually need.
6. **Backups** (see §10) and **Configuration as Code** (see §6).

---

## 4) Jenkinsfile – SRE‑grade patterns (Declarative)

### 4.1 Minimal “Hello, CI”
```groovy
pipeline {
  agent any
  stages {
    stage('Build') { steps { echo 'Building…' } }
    stage('Test')  { steps { echo 'Testing…' } }
    stage('Deploy'){ steps { echo 'Deploying…' } }
  }
}
```

### 4.2 With timeouts, retries, post conditions
```groovy
pipeline {
  agent any
  options { timeout(time: 20, unit: 'MINUTES') }
  stages {
    stage('Unit tests') {
      steps {
        retry(2) { sh 'pytest -q' }
        junit 'reports/**/*.xml'
      }
    }
    stage('Build image') {
      steps { sh 'docker build -t app:$BUILD_NUMBER .' }
    }
  }
  post {
    always { archiveArtifacts artifacts: 'logs/**', allowEmptyArchive: true }
    failure { slackSend channel: '#alerts', message: "${env.JOB_NAME} #${env.BUILD_NUMBER} failed" }
  }
}
```

### 4.3 Parallel + Matrix (multi‑env tests)
```groovy
pipeline {
  agent any
  stages {
    stage('Matrix') {
      matrix {
        axes {
          axis { name 'PY'; values '3.9', '3.10' }
          axis { name 'OS'; values 'ubuntu', 'alpine' }
        }
        agent { label 'docker' }
        stages { stage('Test') { steps { sh 'make test' } } }
      }
    }
  }
}
```

### 4.4 Docker agent (clean builds)
```groovy
pipeline {
  agent { docker { image 'python:3.11-alpine' reuseNode false } }
  stages { stage('Lint') { steps { sh 'pip install ruff && ruff check .' } } }
}
```

### 4.5 Kubernetes ephemeral agent (podTemplate)
```groovy
pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: ci
    image: ghcr.io/curl/curl:8
    command: ['cat']
    tty: true
  restartPolicy: Never
"""
    }
  }
  stages { stage('Curl') { steps { container('ci'){ sh 'curl -s https://example.com' } } } }
}
```

### 4.6 Shared Library (enterprise reuse)
**vars/notifySlack.groovy**
```groovy
def call(String msg) { slackSend channel: '#ci', message: msg }
```
**Jenkinsfile**
```groovy
@Library('company-libs@main') _
pipeline {
  agent any
  stages { stage('Build'){ steps{ notifySlack('CI started'); sh 'make' } } }
}
```

---

## 5) Credentials, Secrets & SCM Webhooks
- **Credentials Types:** username+password, SSH key, secret text, AWS creds.
- **Best practice:** scope to **folder**/**multibranch**; reference only via `withCredentials`.
```groovy
withCredentials([string(credentialsId: 'slack-token', variable: 'SLACK')]) {
  sh 'curl -H "Authorization: Bearer $SLACK" -d msg=ok https://slack/api'
}
```
- **GitHub Webhooks:** expose Jenkins reachable URL (public or via **ngrok**, or GitHub App), create **Multibranch Pipeline** job.

---

## 6) Configuration as Code (JCasC)
Keep Jenkins config in Git for audit & reproducibility.

**`jenkins-casc.yml` (minimal)**
```yaml
jenkins:
  systemMessage: "Managed by JCasC"
  numExecutors: 0
security:
  globalJobDslSecurityConfiguration:
    useScriptSecurity: true
unclassified:
  location:
    url: "https://jenkins.example.com/"
credentials:
  system:
    domainCredentials:
    - credentials:
      - usernamePassword:
          id: "git-creds"
          username: "git"
          password: "***"
```
**Enable** via environment var:
```bash
JAVA_OPTS="-Djenkins.install.runSetupWizard=false -Dcascade.config=/var/jenkins_home/casc/jenkins-casc.yml"
```
(For Helm: `controller.JCasC.defaultConfig=true` and mount your YAML.)

---

## 7) Observability & Reliability Gates
- **Metrics:** enable `/metrics` (Prometheus plugin) → alert on queue length, executor usage, job failures, agent disconnects.
- **Logs:** ship `jenkins.log` and `build logs` to ELK/Datadog.
- **Change safety gates:**
  - SLO pre‑checks (query error budget via Prometheus API before deploy).
  - Canary/Blue‑Green, automatic rollback on health checks.
  - Manual `input` for prod with audit trail.

**SLO Gate example**
```groovy
stage('SLO gate') {
  steps {
    script {
      def burn = sh(script: 'curl -s http://prometheus/api | jq .burn', returnStdout: true).trim()
      if (burn.toFloat() > 1.0) error('Error budget burn too high; aborting deploy')
    }
  }
}
```

---

## 8) Security Hardening (SRE focus)
- **Upgrade to LTS** regularly; pin plugin versions; test on staging.
- **Matrix auth** + folders; disable anonymous; **read‑only** views for auditors.
- **Credentials hygiene:** rotate, least privilege, integrate with Vault/Cloud KMS.
- **Agent isolation:** containers/pods; read‑only workspace where possible.
- **Script Approval:** restrict; use Shared Libraries for vetted code.
- **CSRF & CLI:** keep CSRF on; prefer HTTP/SSH CLI; disable remoting CLI.

---

## 9) Scaling, HA & Disaster Recovery
- **Scale out with agents:** Kubernetes plugin (pod per build), EC2/Fargate, GCE.
- **Controller sizing:** CPU for pipeline scheduling; RAM for plugins/UI. Start 2–4 vCPU, 8–16 GB RAM for medium orgs.
- **HA approach:**
  - *Active/standby controllers* (shared external DB unsupported – Jenkins is file‑based). Replicate `$JENKINS_HOME` + DNS failover.
  - Use **Pipeline Shared Libraries** to keep logic stateless.
- **DR:** frequent snapshots of `$JENKINS_HOME`, S3/versioned bucket, test restore monthly.

**Backup snippet (Docker)**
```bash
docker run --rm -v $HOME/jenkins_home:/src -v $HOME/backups:/dst \
  alpine tar czf /dst/jenkins-$(date +%F).tgz -C /src .
```
**Restore:** stop Jenkins → restore archive → start Jenkins.

---

## 10) Performance Tuning Checklist
- **Executors:** `#executors == cores` on agents; controller usually 0–2.
- **GC/Heap:** `-Xms2g -Xmx4g` (adjust to usage). Monitor GC pauses.
- **Disable heavy plugins** you don’t need (Job Config History/Heavyweight forks).
- **Workspace cleanup** (`cleanWs()`) to cut I/O.
- **Parallel stages** instead of serial where safe.
- **HTTP keep‑alive** to artifact stores.

---

## 11) Hands‑On Labs (copy/paste ready)

### Lab 1 – Hello Jenkinsfile
Create a Multibranch Pipeline → point at repo with this `Jenkinsfile`:
```groovy
pipeline { agent any; stages { stage('Hello'){ steps { echo "Hello from Jenkins" } } } }
```

### Lab 2 – Lint/Test/Package (Node.js example)
```groovy
pipeline {
  agent { docker { image 'node:20-alpine' } }
  stages {
    stage('Install'){ steps { sh 'npm ci' } }
    stage('Lint'){ steps { sh 'npx eslint . || true' } }
    stage('Test'){ steps { sh 'npm test -- --ci --reporters=jest-junit', junit 'junit.xml' } }
    stage('Package'){ steps { sh 'npm run build' } }
  }
}
```

### Lab 3 – Build & Push Docker Image
```groovy
pipeline {
  agent any
  environment { IMAGE = "ghcr.io/acme/app:${env.BUILD_NUMBER}" }
  stages {
    stage('Docker build'){ steps { sh 'docker build -t $IMAGE .' } }
    stage('Login & push'){
      steps {
        withCredentials([string(credentialsId: 'ghcr-token', variable: 'TOKEN')]) {
          sh 'echo $TOKEN | docker login ghcr.io -u acme --password-stdin'
          sh 'docker push $IMAGE'
        }
      }
    }
  }
}
```

### Lab 4 – K8s Deploy (Kubectl)
```groovy
pipeline {
  agent { label 'k8s-tools' }
  environment { KUBECONFIG = credentials('kubeconfig-prod') }
  stages {
    stage('Deploy'){ steps { sh 'kubectl apply -f k8s/' } }
    stage('Smoke'){ steps { sh 'kubectl rollout status deploy/app -n prod' } }
  }
}
```

### Lab 5 – Terraform Plan & Approval
```groovy
pipeline {
  agent { docker { image 'hashicorp/terraform:1.7' } }
  stages {
    stage('Init'){ steps { sh 'terraform init -input=false' } }
    stage('Plan'){ steps { sh 'terraform plan -out=tf.plan -input=false' ; archiveArtifacts 'tf.plan' } }
    stage('Approve?'){ steps { input message: 'Apply to prod?', ok: 'Apply' } }
    stage('Apply'){ steps { sh 'terraform apply -input=false tf.plan' } }
  }
}
```

### Lab 6 – Parallel Tests
```groovy
pipeline {
  agent any
  stages {
    stage('Tests'){
      parallel {
        stage('Unit'){ steps { sh 'pytest -q' } }
        stage('Integration'){ steps { sh 'make itest' } }
      }
    }
  }
}
```

### Lab 7 – Blue/Green via NGINX Ingress
```groovy
stage('Blue/Green') {
  steps {
    sh 'kubectl set image deploy/app-blue app=$IMAGE -n prod'
    sh 'kubectl rollout status deploy/app-blue -n prod'
    sh 'kubectl patch svc app -n prod -p "{\"spec\":{\"selector\":{\"app\":\"app-blue\"}}}"'
  }
}
```

### Lab 8 – Slack Notifications (shared lib)
```groovy
@Library('company-libs') _
pipeline { agent any; stages { stage('Build'){ steps { notifySlack('Build started'); sh 'make'; notifySlack('Build done') } } } }
```

### Lab 9 – SBOM + Scan
```groovy
pipeline {
  agent { docker { image 'anchore/syft:latest' } }
  stages {
    stage('SBOM'){ steps { sh 'syft dir:. -o cyclonedx-json > sbom.json'; archiveArtifacts 'sbom.json' } }
  }
}
```

### Lab 10 – Retry + Lock (serialize env)
```groovy
pipeline {
  agent any
  stages {
    stage('Serialize'){ steps { lock(resource: 'prod-deploy'){ retry(2){ sh './deploy.sh' } } } }
  }
}
```

### Lab 11 – GitOps Trigger (ArgoCD)
```groovy
stage('Update manifests') { steps { sh 'yq -i ".image.tag=\"${BUILD_NUMBER}\"" charts/app/values.yaml'; sh 'git commit -am "bump" && git push' } }
```

### Lab 12 – Chaos Smoke (example)
```groovy
stage('Chaos') { steps { sh 'litmusctl run chaos --name pod-delete --namespace prod' } }
```

> Tip: Run labs on a **kind** cluster locally with a k8s‑tools agent image that includes `kubectl`, `helm`, `yq`, `jq`.

---

## 12) Troubleshooting Playbook (SRE‑style)
- **Can’t log in / setup wizard again** → wrong `$JENKINS_HOME` or permissions.
- **UI slow** → check heap, GC logs, plugin bloat, large build logs; enable gzip.
- **Jobs stuck in queue** → no agents with matching labels; offline agent; executor = 0.
- **Webhook not triggering** → check GitHub delivery logs, Jenkins URL, webhook endpoint `/github-webhook/` reachable, CSRF crumbs.
- **K8s agents pending** → RBAC for SA, podTemplate image exists, node selectors/tolerations.
- **Docker inside Jenkins** → bind docker socket carefully or use DinD agents.
- **Plugin breakage** → use **safe restart**, roll back with tested plugin list, keep a staging controller.

Useful files/paths:
- macOS (brew): `/opt/homebrew/var/jenkins_lts/` (AS) or `/usr/local/var/jenkins_lts/` (Intel)
- Linux: `/var/lib/jenkins/` (home), `/var/log/jenkins/jenkins.log` (logs)

---

## 13) Jenkins CLI & Admin Snippets
```bash
# List plugins
java -jar jenkins-cli.jar -s http://jenkins:8080/ -auth user:token list-plugins
# Safe restart
java -jar jenkins-cli.jar -s http://jenkins:8080/ -auth user:token safe-restart
```
**Script Console (Manage Jenkins → Script Console)**
```groovy
println Jenkins.instance.getRootUrl()
Jenkins.instance.pluginManager.plugins.each { println "${it.shortName}:${it.version}" }
```

---

## 14) Jenkins Cheat Sheet (quick)
- `agent any|label('x')|docker{}` – where to run.
- `environment {}` – vars; `withCredentials {}` – secrets.
- `options { timeout(); buildDiscarder() }` – safety & retention.
- `when { branch 'main' }` – conditional.
- `post { success|failure|always }` – notifications & cleanup.
- `stash/unstash` – pass artifacts across stages/agents.
- `input` – manual approval with audit.
- `lock` – serialize environments.
- `retry`, `timeout` – resilience.

---

## 15) Interview Questions (with crisp answers)

### Architecture & Core
1. **Jenkins controller vs agent?** Controller schedules jobs & hosts UI; agents execute builds. SRE uses **ephemeral agents** for isolation & scalability.
2. **Where is state stored?** `$JENKINS_HOME` filesystem (jobs, configs, creds, plugins). Must be backed up.
3. **Declarative vs Scripted pipeline?** Declarative = opinionated, simpler; Scripted = full Groovy freedom. Prefer Declarative for consistency; use Shared Libraries for complexity.
4. **How to scale Jenkins?** Increase agents (K8s/Cloud), separate heavy jobs, reduce controller workload, tune heap/GC, cache dependencies.
5. **Why plugins are risky?** Version drift & compatibility can break controller. Maintain a **pinned, tested plugin bill of materials**.

### Security & Governance
6. **How do you manage credentials?** Folder‑scoped creds, least privilege, rotate, integrate Vault/KMS, avoid printing to logs, use `withCredentials`.
7. **Locking down Jenkins?** Matrix auth, disable anonymous, CSRF enabled, read‑only for viewers, disable remoting CLI, audit logs, JCasC.
8. **Supply‑chain security steps?** SBOM (Syft), vulnerability scan (Grype/Trivy), signed images (Cosign), pinned base images, reproducible builds.

### Reliability & Change Safety
9. **Implement change freeze/SLO gate?** Use pipeline `input` + automated SLO checks from Prometheus; block deploys if burn rate high.
10. **Blue‑Green vs Canary in Jenkins?** Jenkins orchestrates rollout steps (kubectl/Argo Rollouts). Canary = partial traffic shift + metrics gates; Blue‑Green = full switch.

### Kubernetes & Cloud
11. **K8s agents not launching – debug path?** Check SA/RBAC, plugin version, podTemplate image, node selectors, resource quotas; inspect pod events.
12. **Artifact management?** Use Nexus/JFrog/S3; cache dependencies; fingerprint artifacts; immutable tags.

### Operations & DR
13. **Backup & restore plan?** Nightly `$JENKINS_HOME` snapshot to versioned storage; test restore monthly; keep plugin list & JCasC in Git.
14. **Rolling upgrades?** Stage → backup → upgrade LTS → verify → prod; use active/standby with DNS swap for minimal downtime.

### Pipelines & Groovy
15. **Common pipeline reliability patterns?** `retry`, `timeout`, `lock`, `parallel`, `stash/unstash`, workspace cleanup, idempotent deploy scripts, post‑failure triage.

> For live interviews, walk through a recent pipeline you built, the **safety gates** you added, how you **measured reliability**, and how you **handled a failed rollout**.

---

## 16) Capstone: Production‑style CI/CD
**Goal:** Containerize a service → test → build image → push → deploy to K8s with canary + SLO check + rollback.

**Outline Jenkinsfile:**
```groovy
pipeline {
  agent any
  environment { IMAGE="registry.example.com/app:${env.BUILD_NUMBER}" }
  options { timeout(time: 45, unit: 'MINUTES'); buildDiscarder(logRotator(numToKeepStr: '20')) }
  stages {
    stage('Checkout'){ steps { checkout scm } }
    stage('Build & Test'){ steps { sh 'make ci' ; junit 'reports/*.xml' } }
    stage('Image'){ steps { sh 'docker build -t $IMAGE .' ; sh 'docker push $IMAGE' } }
    stage('Deploy canary'){
      steps { sh 'helm upgrade --install app chart/ --set image.tag=${BUILD_NUMBER} --set canary.enabled=true' }
    }
    stage('SLO gate'){
      steps { script { def burn = sh(script: './scripts/burn_rate.sh', returnStdout: true).trim(); if(burn.toFloat()>1.0) error('Abort: SLO burn') } }
    }
    stage('Promote to 100%'){ steps { sh 'helm upgrade app chart/ --set canary.enabled=false' } }
  }
  post {
    failure { sh 'helm rollback app || true' }
    always  { slackSend channel: '#deploys', message: "${currentBuild.currentResult} – ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
  }
}
```

---

## 17) What to memorize vs automate
- **Memorize:** core DSL, safety primitives (timeout/retry/lock), plugin hygiene, backup/restore, RBAC, JCasC basics.
- **Automate:** full controller bootstrap (Docker/Helm + JCasC), shared libraries, plugin BOM, agent images, observability wiring.

---

## 18) Next Steps
1. Spin up Jenkins (Docker or Helm) and import **JCasC**.
2. Run Labs 1→6, then wire **K8s agents** and complete Labs 7→12.
3. Track metrics & alerts; add SLO gates to at least one production pipeline.
4. Practice restore from backup. Document your **runbooks**.

---

*End of guide. Use this as a living document in your team wiki/Notion and keep configs in Git.*

