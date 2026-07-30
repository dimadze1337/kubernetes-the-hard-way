# Install Haproxy

## Configuring SSH Access

SSH will be used to configure the haproxy balancer for the cluster. Verify that you have `root` SSH access to the machine. You may need to enable root SSH access on each node by updating the sshd_config file and restarting the SSH server.

### Enable root SSH Access

If `root` SSH access is enabled for each of your machines you can skip this section.

By default, a new `debian` install disables SSH access for the `root` user. This is done for security reasons as the `root` user has total administrative control of unix-like systems. If a weak password is used on a machine connected to the internet, well, let's just say it's only a matter of time before your machine belongs to someone else. As mentioned earlier, we are going to enable `root` access over SSH in order to streamline the steps in this tutorial. Security is a tradeoff, and in this case, we are optimizing for convenience. Log on to each machine via SSH using your user account, then switch to the `root` user using the `su` command:

```bash
su - root
```

Edit the `/etc/ssh/sshd_config` SSH daemon configuration file and set the `PermitRootLogin` option to `yes`:

```bash
sed -i \
  's/^#*PermitRootLogin.*/PermitRootLogin yes/' \
  /etc/ssh/sshd_config
```

Restart the `sshd` SSH server to pick up the updated configuration file:

```bash
systemctl restart sshd
```

### Generate and Distribute SSH Keys

In this section you will generate and distribute an SSH keypair to the `server`, `node-0`, and `node-1`, machines, which will be used to run commands on those machines throughout this tutorial. Run the following commands from the `jumpbox` machine.

Generate a new SSH key:

```bash
ssh-keygen
```

```text
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
```

Copy the SSH public key to the machine:

```bash
  ssh-copy-id root@$192.168.31.210
```

Once each key is added, verify SSH public key access is working:

```bash
  ssh -n root@${192.168.31.210} hostname
```

```text
k8s-haproxy
```

## Setup haproxy

```bash
  sudo apt update
  sudo apt install haproxy -y
  sudo systemctl enable haproxy
```

Minimal configuration for k8s cluster

```bash
  sudo nano /etc/haproxy/haproxy.cfg
```

```text

global
    log /dev/log local0
    log /dev/log local1 notice
    daemon
    maxconn 4000

defaults
    log global
    mode tcp
    option tcplog
    timeout connect 5s
    timeout client 50s
    timeout server 50s
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http


frontend kubernetes
    bind *:6443
    default_backend kube_api

backend kube_api
    balance roundrobin
    option tcp-check

    server master1 192.168.31.201:6443 check
    server master2 192.168.31.202:6443 check
    server master3 192.168.31.203:6443 check
    
frontend http
    bind *:80
    default_backend ingress_http

backend ingress_http
    balance roundrobin
    option httpchk
    server master1 192.168.31.201:30855 check
    server master2 192.168.31.202:30855 check
    server master3 192.168.31.203:30855 check
    
frontend https
    bind *:443
    default_backend ingress_https

backend ingress_https
    balance roundrobin
    option tcp-check
    mode tcp
    server master1 192.168.31.201:32286 check
    server master2 192.168.31.202:32286 check
    server master3 192.168.31.203:32286 check

```

Check and run configuration

```bash
  sudo haproxy -c -f /etc/haproxy/haproxy.cfg
  sudo systemctl restart haproxy
  sudo systemctl status haproxy
```

Check port

```bash
  ss -lntp | grep 6443
```

Should be like

```text
LISTEN 0 4096 *:6443
```


Next: [Provisioning Compute Resources](04-compute-resources.md)