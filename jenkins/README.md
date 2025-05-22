# Jenkins Job Trigger Pipeline

<img width="1144" alt="Image" src="https://github.com/user-attachments/assets/7a45a1bd-b0b7-4c4f-81ff-db21800be530" />

This Harness pipeline is designed to **trigger a Jenkins job**, **monitor its progress**, and **fetch logs**, offering a seamless integration between Harness and Jenkins CI/CD workflows.
It is a reusable Harness CI/CD template. It automates the process of triggering Jenkins jobs & monitors it in real-time, and fetches detailed console logs—all within a single step. 

## Pre-requisites

- Jenkins server must be accessible from the Harness delegate environment.
- Jenkins user must have:
  - API access enabled (this will be used to trigger jobs)
  - Permissions to trigger and monitor builds
- Jenkins job (`JOB_NAME`) should exist and be reachable.


## Input Variables

| Variable Name   |  Mandatory | Description                                                                |
|----------------|----------|----------------------------------------------------------------------------|
| `JENKINS_URL`  | Yes   | Base URL of the Jenkins server (e.g., `http://jenkins.local:8080`)        |
| `JENKINS_USER` | Yes   | Jenkins username                                                           |
| `JENKINS_TOKEN`| Yes   | Jenkins API token (stored securely in Harness as secret)                            |
| `JOB_NAME`     | Yes   | Jenkins job name to trigger                                               |
| `JOB_PARAMS`   | No    | Optional job parameters (e.g., `env=prod&runId=123`)                      |



## Trigger And Monitor Jenkins Job

### Ensure `jq` is installed

The script installs `jq` using the available package manager or binary fallback if not already present.

```bash
if ! command -v jq >/dev/null 2>&1; then
  echo "[Info]: jq not found, attempting to install…"

  install_jq_pm() {
    if command -v apt-get >/dev/null 2>&1; then
      apt-get update && apt-get install -y jq
    elif command -v yum >/dev/null 2>&1; then
      yum install -y jq
    elif command -v apk >/dev/null 2>&1; then
      apk add --no-cache jq
    else
      return 1
    fi
  }

  install_jq_binary() {
    local url="https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux64"
    local dest="$HOME/.local/bin/jq"
    mkdir -p "$(dirname "$dest")"
    if curl -sSL "$url" -o "$dest"; then
      chmod +x "$dest"
      export PATH="$HOME/.local/bin:$PATH"
      return 0
    else
      return 1
    fi
  }

  if install_jq_pm; then
    echo "[Info]: jq installed via package manager ($(jq --version))"
  elif install_jq_binary; then
    echo "[Info]: jq downloaded to ~/.local/bin and available ($(jq --version))"
  else
    echo "[Error]: Failed to install jq. Please install it manually and re-run."
    exit 1
  fi
else
  echo "[Info]: jq is already installed ($(jq --version))"
fi
```

### Trigger Jenkins job

The job is triggered using Jenkins' REST API, with or without parameters

```bash
crumb=$(curl -s -u "$JENKINS_USER:$JENKINS_TOKEN" \
  "$JENKINS_URL/crumbIssuer/api/json" | jq -r .crumb)

if [[ -n "$PARAMS" ]]; then
  endpoint="buildWithParameters?$PARAMS"
else
  endpoint="build"
fi

curl -s -u "$JENKINS_USER:$JENKINS_TOKEN" \
     -H "Jenkins-Crumb:$crumb" \
     -X POST \
     "$JENKINS_URL/job/$JOB_NAME/$endpoint"
```

### Wait for build to start

The script polls Jenkins API until a new build number is greater than the last recorded build number.

```bash
old_build=$(curl -s -u "$JENKINS_USER:$JENKINS_TOKEN" \
  "$JENKINS_URL/job/$JOB_NAME/api/json" | jq -r '.lastBuild.number // 0')

new_build=$old_build
while (( new_build <= old_build )); do
  sleep 2
  new_build=$(curl -s -u "$JENKINS_USER:$JENKINS_TOKEN" \
    "$JENKINS_URL/job/$JOB_NAME/api/json" | jq -r '.lastBuild.number // 0')
done
```

### Monitor job status

Monitor until build finishes by checking the building flag and capturing the result

```bash
while :; do
  info=$(curl -s -u "$JENKINS_USER:$JENKINS_TOKEN" \
    "$JENKINS_URL/job/$JOB_NAME/$new_build/api/json")
  building=$(jq -r '.building' <<<"$info")
  result=$(jq -r '.result // empty' <<<"$info")
  ts=$(date '+%H:%M:%S')

  if [[ "$building" == "true" ]]; then
    echo "[$ts] RUNNING"
  elif [[ -n "$result" ]]; then
    echo "[$ts] DONE → $result"
    break
  else
    echo "[$ts] PENDING"
  fi
  sleep 5
done
```

### Print Jenkins console output

Displays full logs from the Jenkins build for transparency and debugging.

```bash
curl -s -u "$JENKINS_USER:$JENKINS_TOKEN" \
     "$JENKINS_URL/job/$JOB_NAME/$new_build/consoleText"
```
