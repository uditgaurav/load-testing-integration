# Load Testing Integration

This repository contains reusable pipeline templates and step templates to facilitate automated load testing workflows using tools like Jenkins and Gatling.

---

## Structure

- [`/jenkins`](./jenkins/README.md)  
  Contains a Harness pipeline that triggers a Jenkins job, monitors its execution, and prints the console output. Useful for integrating Jenkins-based test runners or custom CI flows into Harness.

- [`/gatling`](./gatling/README.md)  
  Contains a step template to run a simulation on Gatling Enterprise Cloud. It uses the Gatling public API to trigger and track the status of load test runs.

---

## Usage

Each subdirectory includes:
- A `pipeline-template.yaml` or step YAML
- A `README.md` with detailed documentation
- Environment variables and pre-requisites needed for execution

Refer to the linked READMEs for full setup, usage, and example configurations.

---

## Prerequisites

- Active Harness project with permissions to create templates
- Delegate installed and tagged accordingly (used in templates)
- Access to Jenkins or Gatling Enterprise APIs depending on the template in use

---

## Quick Links

- [Jenkins Integration README](./jenkins/README.md)
- [Gatling Integration README](./gatling/README.md)
