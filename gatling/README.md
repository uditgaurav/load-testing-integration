# Gatling Load Run Step Template

<img width="993" alt="Image" src="https://github.com/user-attachments/assets/dd58df58-ec21-4e92-81e8-ed79697b5631" />

This Harness step template is designed to **trigger a Gatling Enterprise Cloud simulation**, monitor its lifecycle phase, and report its status in real time. It uses the Gatling Public API to start a simulation using a `simulation_id`, authenticates with an API `token`, and continuously polls for its current run phase.

---

## Pre-requisites

- An active [Gatling Enterprise Cloud](https://gatling.io/enterprise/) account with a valid **simulation ID** already configured in Gatling
- An **API token** with access to start simulations and view run status.

---

## Input Variables

| Variable Name   | Mandatory | Description                                                                 |
|-----------------|----------|-----------------------------------------------------------------------------|
| `simulation_id` | Yes   | The unique ID of the simulation in Gatling Enterprise Cloud                 |
| `token`         | Yes   | Token used for authenticating with the Gatling API                   |
| `delegateSelectors` | Yes | Harness delegate(s) to execute this step on                                 |

---


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


### Validate mandatory variables

Returns an error if either simulation_id or token is empty.

```bash
if [ -z $simulation_id ]; then
  echo "[error]: simulation_id is empty"
  exit 1
fi

if [ -z $token ]; then
  echo "[error]: token is empty"
  exit 1
fi
```

### Start Gatling simulation

Triggers the simulation via the Gatling API and captures the run_id.

```bash
start_simulation_response=$(curl -s -w "\n%{http_code}" -X POST \
  https://api.gatling.io/api/public/simulations/start?simulation=$simulation_id \
  -H "accept: application/json" \
  -H "authorization: $token" \
  -H "Content-Type: application/json" \
  -d '{}')

run_id=$(echo $start_simulation_body | jq .runId | tr -d '"')

```

### Poll simulation run status

The script polls the run’s current phase and maps it to a readable status using phase_string.

```bash
while [ $simulation_run_phase -le 3 ]; do
  simulation_run_response=$(curl -s -w "\n%{http_code}" -X GET \
    https://api.gatling.io/api/public/run?run=$run_id \
    -H "accept: application/json" \
    -H "authorization: $token" \
    -H "Content-Type: application/json" \
    -d '{}')

  simulation_run_phase=$(echo $simulation_run_body | jq .status)
  phase_string=$(phase_string $simulation_run_phase)
  echo "[info]: simulation run $run_id phase: [$simulation_run_phase] $phase_string"
done

```

#### Phase Mapping

| Code | Phase      | Description                                   |
|------|------------|-----------------------------------------------|
| 0    | Building   | Preparing the environment for the simulation  |
| 1    | Deploying  | Deploying necessary simulation components     |
| 2    | Deployed   | Deployment completed, pre-run setup finished  |
| 3    | Injecting  | Simulation is actively running                |
| 4    | Completed  | Simulation has finished execution             |

