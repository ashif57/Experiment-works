# Configure a temporary NAT instance using your public EC2

We will use your existing public EC2 as a temporary NAT instance to provide internet access to your private EC2, install Docker, and then completely reverse the temporary networking changes.



No NAT Gateway required

No new EC2 required



## Architecture

Internet

Internet Gateway

Public subnet

Public EC2 – Bastion / Temporary NAT

Public IP + IP forwarding + NAT masquerading

Private subnet route: 0.0.0.0/0 → Public EC2 ENI

Private subnet

Private EC2

Downloads Docker packages and images

## Phase 1 – Pre-checks

Before making changes, verify these conditions:

1. Both EC2 instances are in the same VPC.
2. Public EC2 is in a public subnet.
3. Public EC2 has a public IPv4 address or Elastic IP.
4. Public subnet route table has `0.0.0.0/0 → Internet Gateway`.
5. Private subnet route table is identified and its existing routes are recorded.

On both EC2 instances, execute:

```
ip route
ip -br addr
```

Check OS:

```
cat /etc/os-release
```

On the public EC2, verify internet:

```
curl -I https://google.com
```

Save the existing network configuration before proceeding.

## Phase 2 – Configure the public EC2

### Step 1: Disable source/destination checking

AWS Console:

EC2 → Instances → Select public EC2 → Actions → Networking → Change source/destination check → Disable.

This is required because the public EC2 will forward packets that are not originally addressed to itself.

### Step 2: Enable IP forwarding

On the public EC2:

```
sudo sysctl -w net.ipv4.ip_forward=1
```

Verify:

```
sysctl net.ipv4.ip_forward
```

Expected:

```
net.ipv4.ip_forward = 1
```

### Step 3: Identify the network interface

```
ip route
```

Example:

```
default via 10.0.1.1 dev ens5
```

Here, the network interface is `ens5`.

Do not assume yours is also `ens5`; verify first.

### Step 4: Configure NAT masquerading

Install iptables if required:

```
sudo dnf install iptables -y
```

For an interface named `ens5` and a private subnet CIDR of `10.0.6.0/24`, execute:

```
sudo iptables -t nat -A POSTROUTING \
-s 10.0.6.0/24 -o ens5 -j MASQUERADE
```

Allow forwarding from the private subnet:

```
sudo iptables -A FORWARD \
-s 10.0.6.0/24 -o ens5 -j ACCEPT
```

Allow return traffic:

```
sudo iptables -A FORWARD \
-d 10.0.6.0/24 -i ens5 \
-m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

These commands assume the private subnet is `10.0.6.0/24`; replace it with your actual subnet CIDR.

Security: Do not add a blanket `ACCEPT` forwarding rule. Keep forwarding restricted to the intended private subnet.

### Step 5: Configure the public EC2 security group

The NAT instance needs to accept forwarded traffic from your private subnet.

Add inbound rules to the public EC2 security group:

| Type            | Protocol | Port | Source              |
| --------------- | -------- | ---- | ------------------- |
| HTTPS           | TCP      | 443  | Private subnet CIDR |
| HTTP            | TCP      | 80   | Private subnet CIDR |
| DNS (if needed) | UDP      | 53   | Private subnet CIDR |
| DNS (if needed) | TCP      | 53   | Private subnet CIDR |

Keep your existing SSH rule unchanged.

Also verify:

- Public EC2 outbound rules allow required internet traffic.
- Private EC2 outbound rules permit the required traffic.
- Network ACLs permit the traffic and return ephemeral ports.

## Phase 3 – Update the private subnet route table

Go to:

AWS Console → VPC → Route Tables.

1. Select the route table associated with your private EC2 subnet.
2. Record its existing routes.
3. Select Edit routes.
4. Add the following route:

| Destination | Target                 |
| ----------- | ---------------------- |
| `0.0.0.0/0` | Public EC2 instance ID |

Example:

```
Destination: 0.0.0.0/0
Target:      i-xxxxxxxxxxxxxxxxx
```

Choose the public EC2 instance as the target.

Important

This route affects every instance using that route table, not just your target EC2. If other production workloads use it, create a separate route table for a dedicated temporary subnet or otherwise isolate the target before proceeding.

## Phase 4 – Test internet access from private EC2

SSH into your private EC2 through the bastion.

Check:

```
ip route
```

Test DNS:

```
getent hosts google.com
```

Test HTTPS:

```
curl -I https://google.com
```

Test package repositories:

```
sudo dnf makecache
```

If these work, your temporary NAT is functioning.

## Phase 5 – Install Docker

Since your instance appears to be Amazon Linux 2023, execute:

```
sudo dnf install docker -y
```

Start and enable Docker:

```
sudo systemctl enable --now docker
```

Add EC2 user to Docker group:

```
sudo usermod -aG docker ec2-user
```

Verify:

```
docker --version
sudo docker run hello-world
```

Reconnect to SSH and check:

```
docker ps
```

At this point, Docker is installed.

## Phase 6 – Undo everything (cleanup)

Once installation and testing are complete, reverse the changes in this order.

### Step 1: Restore the private subnet route table

AWS Console → VPC → Route Tables.

- Remove the temporary `0.0.0.0/0 → Public EC2` route.
- Restore the original route if one existed.

### Step 2: Remove NAT iptables rules

On the public EC2, execute the same rules with `-D` instead of `-A`:

```
sudo iptables -D FORWARD \
-d 10.0.6.0/24 -i ens5 \
-m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

```
sudo iptables -D FORWARD \
-s 10.0.6.0/24 -o ens5 -j ACCEPT
```

```
sudo iptables -t nat -D POSTROUTING \
-s 10.0.6.0/24 -o ens5 -j MASQUERADE
```

Replace the interface and subnet with the actual values you used.

### Step 3: Disable IP forwarding

On the public EC2:

```
sudo sysctl -w net.ipv4.ip_forward=0
```

If you added the persistent setting to `/etc/sysctl.conf`, remove that entry too.

### Step 4: Re-enable source/destination checking

AWS Console:

EC2 → Instances → Select public EC2 → Actions → Networking → Change source/destination check → Enable.

### Step 5: Remove temporary security group rules

Remove the temporary HTTP, HTTPS and DNS inbound rules that you added for NAT forwarding.

### Step 6: Verify cleanup

On public EC2:

```
sysctl net.ipv4.ip_forward
sudo iptables -t nat -L -n -v
sudo iptables -L FORWARD -n -v
```

On private EC2:

```
docker --version
sudo systemctl status docker
```

Docker remains installed even after NAT is removed.

## Complete execution checklist

Implementation progress

0/14 completed

Confirm both EC2 instances are in the same VPC

Record subnet CIDRs and existing route tables

Verify public EC2 internet connectivity

Disable source/destination check

Enable IP forwarding

Configure restricted iptables NAT rules

Update private subnet route table

Test private EC2 internet connectivity

Install Docker and run hello-world

Restore original route table

Remove temporary iptables rules

Disable IP forwarding

Re-enable source/destination check

Remove temporary security group rules

## Before executing: send me these details

I don't want you to accidentally change the wrong route table or network interface.

Run these commands:

Public EC2:

```
ip route
ip -br addr
cat /etc/os-release
```

Private EC2:

```
ip route
ip -br addr
cat /etc/os-release
```

Also provide:

- Public subnet CIDR.
- Private subnet CIDR.
- Whether both instances are in the same VPC.

I can then give you the exact commands with your actual interface name and subnet CIDR, rather than relying on example values.