# DataCamp-Infrastructure-Compromise
Critical Kubernetes Service Account token leakage and infrastructure compromise vulnerability report.
# Critical Bug Report: Kubernetes Token and Source Code Leak in DataCamp DataLab

Hey everyone, this is a security report about a critical bug I found in DataCamp's DataLab workspace environment. The bug was reported to the team and they marked it as a Duplicate (which means the bug is real, but someone found it before me). 
By exploiting this misconfiguration, I was able to get a valid Kubernetes token and access their internal backend code.

# Target Details:
- Target Platform: DataCamp DataLab Workspace
- Environment: AWS EKS (us-east-1 region)
- Impact: Critical (Infrastructure takeover / Source code leak)
  
# How I Found It (Steps to Reproduce):

1. First, I opened a standard DataLab workspace session on DataCamp.
2. I wanted to check the filesystem security, so I ran a simple command to read the service account folder:
   cat /var/run/secrets/kubernetes.io/serviceaccount/token
3. To my surprise, the system let me read it and gave me a valid K8s JWT token! I checked the token and saw it came from their AWS EKS cluster (`https://amazonaws.com`).
4. Then I checked the system environment variables by running `env`. I found a high-privilege proxy token (`PROXYTOKEN=eebd4935-1567-4262-85e3-650852cb46ca`) and the active socket path (`/session-rpc/rpc.sock`).
5. After that, I explored the directories and went to `/home/repl/backend/dist/`. The permission was wrong, so I could read all their production backend files like `config.js` and `collabApiClient.js`. There was more than 18GB of internal files and course templates accessible to any basic user.

# Total Impact:
- **Full Infrastructure Risk:** With the leaked Kubernetes token, an attacker can directly talk to the main K8s API server. This means they can control containers, move around the AWS network, and hack the whole namespace.
- **Bypassing Auth:** The leaked `PROXYTOKEN` allows anyone to make fake internal service calls and hijack active user sessions.
- **IP Theft:** Anyone could download 18GB+ of DataCamp's private source code and templates, destroying their software secrecy.

---

# How to Fix This:
1. Don't auto-mount the K8s token. Use `automountServiceAccountToken: false` in the pod configurations.
2. Fix the folder permissions so a normal workspace user (`repl` user) cannot read the `/var/run/secrets/` directory or backend system files.
3. Keep strict RBAC rules so even if a token leaks, it has zero power to control the cluster.
