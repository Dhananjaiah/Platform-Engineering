# Lab 06-06 - Migrating Jenkins to GitHub Actions (playbook with a worked example)

## Goal
Run a safe, measured migration of 300 Jenkins jobs: inventory, classify, parity-test in parallel, cut over, decommission. Practise with one real pipeline.

## Prereqs
Docker (a local Jenkins), `gh`.

## Step 1 - A Jenkins to migrate (the "before")
```bash
docker run -d --name jenkins -p 8081:8080 -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts-jdk17
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```
Create a pipeline job using this Jenkinsfile:
```groovy
pipeline {
  agent any
  environment { REGISTRY = 'ghcr.io/acme-lab-you' }
  stages {
    stage('Build')  { steps { sh 'npm ci && npm run build --if-present' } }
    stage('Test')   { steps { sh 'npm test' } }
    stage('Docker') { steps { sh 'docker build -t $REGISTRY/payments-api:$BUILD_NUMBER .' } }
    stage('Deploy') { when { branch 'main' } steps { sh './deploy.sh dev' } }
  }
}
```

## Step 2 - Inventory with data, not guesses
```bash
curl -s -u admin:<token> "http://localhost:8081/api/json?tree=jobs[name,url,color,lastBuild[timestamp,result]]" | jq -r '.jobs[] | [.name, .color, (.lastBuild.timestamp//0)] | @tsv'
```
Classify each job: dead (no build in 90 days: delete), duplicates, standard (fits a golden pipeline), special (plugins, shared libraries, manual gates). Expect: 30 % dead, 50 % standard, 20 % special. Standard jobs migrate by template; special jobs get a design review. Use GitHub Actions Importer for a dry run:
```bash
gh extension install github/gh-actions-importer
gh actions-importer audit jenkins --output-dir audit --jenkins-instance-url http://localhost:8081 --jenkins-username admin --jenkins-access-token <token>
```
Read `audit/audit_summary.md`: % convertible, unsupported plugins. Treat the generated YAML as a draft to reshape onto the reusable workflows, not as the final product.

## Step 3 - Map concepts
| Jenkins | Actions |
|---------|---------|
| Shared library | reusable workflow / composite action |
| Credentials store | OIDC first, environment secrets second |
| Agent labels | `runs-on` runner groups / ARC scale sets |
| `input` approval | environment required reviewers |
| Multibranch | `on: push/pull_request` triggers |
| Post conditions | `if: always()/failure()` steps, `needs` with `if` |
| Parameterised build | `workflow_dispatch` inputs |

## Step 4 - The "after" (the golden pipeline, thin caller)
```yaml
name: ci
on: { pull_request: {}, push: { branches: [main] } }
permissions: { contents: read }
jobs:
  ci:    { uses: acme-lab-you/acme-platform-workflows/.github/workflows/ci-node.yml@v1 }
  image:
    needs: ci
    if: github.ref == 'refs/heads/main'
    permissions: { contents: read, packages: write, id-token: write, attestations: write, security-events: write }
    uses: acme-lab-you/acme-platform-workflows/.github/workflows/build-image.yml@v1
    with: { image-name: acme-lab-you/payments-api }
```
`deploy.sh` disappears: deploy becomes the GitOps handoff (Lab 02 Step 5). This is the real win: migrating *to the target architecture*, not lift-and-shift.

## Step 5 - Run in parallel, prove parity
For two weeks both run. Compare per commit: same pass/fail, similar duration, same artifact digest content (image layers/SBOM diff). Track in a table: job, owner, Jenkins result, Actions result, diff reason. Cutover criterion: 10 consecutive matching runs and owner sign-off.

## Step 6 - Cutover and decommission
1. Make Actions the required check in the ruleset; set the Jenkins job read-only.
2. Announce, with a date, owner-by-owner, in the channel and via Backstage (a scorecard "uses standard pipeline").
3. After 30 days: archive job configs to git, shut agents, remove credentials (rotate everything Jenkins held), delete the controller.

## Migration metrics
% repos on Actions, % on the golden workflow, number of exceptions, median CI duration before/after, Jenkins cost removed, incidents attributable to migration.

## Break it on purpose
Make Jenkins pass and Actions fail on a test that depends on `BUILD_NUMBER`/workspace state; document the parity diff and fix. Common causes: implicit tools on Jenkins agents, env vars, network access to internal hosts from hosted runners (use ARC runners in the right network).

## Cleanup
```bash
docker rm -f jenkins && docker volume rm jenkins_home
```

## Talking points
Migrate to the target model; inventory first; templates cover the long tail; parallel-run parity; owner-by-owner cutover with a date; decommission and rotate credentials; measure and report.
