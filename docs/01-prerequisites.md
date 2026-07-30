# Prerequisites

In this lab you will review the machine requirements necessary to follow this tutorial.

## Virtual or Physical Machines

This tutorial requires four (8) virtual or physical ARM64 or AMD64 machines running Ubuntu 24.04.4-live-server-amd64. The following table lists the four machines and their CPU, memory, and storage requirements.

| Name         | Description            | CPU | RAM   | Storage |
|--------------|------------------------|-----|-------|---------|
| jumpbox      | Administration host    | 1   | 512MB | 10GB    |
| haproxy      | Balancer               | 2   | 2GB   | 30GB    |
| k8s-master-1 | Kubernetes server 1    | 2   | 2GB   | 30GB    |
| k8s-master-2 | Kubernetes server 2    | 2   | 2GB   | 30GB    |
| k8s-master-3 | Kubernetes server 3    | 2   | 2GB   | 30GB    |
| k8s-worker-1 | Kubernetes worker node | 2   | 2GB   | 30GB    |
| k8s-worker-2 | Kubernetes worker node | 2   | 2GB   | 30GB    |
| k8s-worker-3 | Kubernetes worker node | 2   | 2GB   | 30GB    |

How you provision the machines is up to you, the only requirement is that each machine meet the above system requirements including the machine specs and OS version. Once you have all four machines provisioned, verify the OS requirements by viewing the `/etc/os-release` file:

```bash
cat /etc/os-release
```

You should see something similar to the following output:

```text
PRETTY_NAME="Ubuntu 24.04.4 LTS"
NAME="Ubuntu"
VERSION_ID="24.04"
VERSION="24.04.4 LTS (Noble Numbat)"
VERSION_CODENAME=noble
ID=ubuntu
ID_LIKE=debian
```

Next: [setting-up-the-jumpbox](02-jumpbox.md)
