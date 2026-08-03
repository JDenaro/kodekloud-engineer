# Day 13: IPtables Installation And Configuration

We have one of our websites up and running on our Nautilus infrastructure in Stratos DC. Our security team has raised a concern that right now Apache’s port i.e 6000 is open for all since there is no firewall installed on these hosts. So we have decided to add some security layer for these hosts and after discussions and recommendations we have come up with the following requirements:

1. Install iptables and all its dependencies on each app host.

2. Block incoming port 6000 on all apps for everyone except for LBR host.

3. Make sure the rules remain, even after system reboot.

## Specific Requirements:

1. Install `iptables` and all its dependencies on each App Server.
2. Block incoming port `6000` on all App Servers for everyone except the LBR host.
3. Preserve the rules after a system reboot.

## Solution

The LBR hostname was resolved before changing any firewall rules. It resolved to `10.244.195.16`, so that IP was used as the only allowed source for `TCP/6000`.

The `iptables` and `iptables-services` packages were installed on all three App Servers. On each host, one rule allowed the LBR to reach port `6000`, and a second rule dropped traffic to that port from every other source. The active rules were saved to `/etc/sysconfig/iptables`, and the `iptables` service was enabled so the saved rules are restored after a reboot.

### 🔎 Step 1: Find the LBR IP address

From `thor@jump-host`, resolve the LBR hostname:

```bash
getent hosts stlb01
```

The LBR resolved to:

```text
10.244.195.16 stlb01.xm5goxzt6bylamss.svc.cluster.local
```

> **Why:** `getent hosts` queries the system's configured name-service sources and returns the address associated with a hostname. `stlb01` is the LBR hostname, and its resolved IP is needed for a source-specific firewall exception. Using the LBR address instead of allowing the entire network keeps the rule limited to the required client.

### 🔌 Step 2: Connect to App Server 1

```bash
ssh tony@stapp01
```

> **Why:** `ssh` opens a secure remote shell. `tony` is the sudo-enabled user for App Server 1, and `stapp01` is the first server where the firewall configuration will be applied and explained in full.

### 📦 Step 3: Install iptables on App Server 1

```bash
sudo yum install -y iptables iptables-services
```

> **Why:** `sudo` provides the administrative privileges needed to install system packages, and `yum install` installs the requested packages. The `-y` option confirms the package operation automatically. `iptables` provides the packet-filtering command, while `iptables-services` provides the system service that can load saved rules during boot.

### 🔓 Step 4: Allow the LBR to reach Apache on App Server 1

```bash
sudo iptables -I INPUT 1 -p tcp -s 10.244.195.16 --dport 6000 -j ACCEPT
```

> **Why:** `iptables` manages packet-filtering rules. `-I INPUT 1` inserts the rule at position `1` in the `INPUT` chain, which processes incoming packets. `-p tcp` limits the rule to TCP traffic, `-s 10.244.195.16` limits the source to the LBR, `--dport 6000` selects Apache's destination port, and `-j ACCEPT` permits matching packets. Inserting this rule first ensures the LBR is accepted before the general block is evaluated.

### 🚫 Step 5: Block all other access to port 6000 on App Server 1

```bash
sudo iptables -I INPUT 2 -p tcp --dport 6000 -j DROP
```

> **Why:** `-I INPUT 2` inserts this rule at position `2`, after the LBR allow rule. The rule matches TCP traffic destined for port `6000` regardless of its source, and `-j DROP` silently discards it. Because the LBR exception is evaluated first, only the LBR can reach the Apache port; other sources are blocked. The existing firewall chains and unrelated rules are preserved.

### 💾 Step 6: Save the rules on App Server 1

```bash
sudo iptables-save -f /etc/sysconfig/iptables
```

The saved file contained the required rules:

```text
-A INPUT -s 10.244.195.16/32 -p tcp -m tcp --dport 6000 -j ACCEPT
-A INPUT -p tcp -m tcp --dport 6000 -j DROP
```

> **Why:** `iptables-save` exports the active rules in a format that can be restored later. The `-f` option writes the rules to the specified file instead of printing them to the terminal. `/etc/sysconfig/iptables` is the file used by the `iptables` service for persistent IPv4 rules on this host.

### ⚙️ Step 7: Enable iptables at boot on App Server 1

```bash
sudo systemctl enable iptables
```

The command created the systemd startup link:

```text
Created symlink /etc/systemd/system/multi-user.target.wants/iptables.service → /usr/lib/systemd/system/iptables.service.
```

> **Why:** `systemctl enable` configures a service to start automatically during future boots. Enabling `iptables` ensures the rules saved in `/etc/sysconfig/iptables` are restored after a reboot. This command does not require a reboot and does not remove the rules currently active in memory.

### ✅ Step 8: Verify App Server 1

```bash
sudo iptables -L INPUT -n --line-numbers
sudo systemctl is-enabled iptables
```

The rules should appear in this order:

```text
1  ACCEPT  tcp  --  10.244.195.16  0.0.0.0/0  tcp dpt:6000
2  DROP    tcp  --  0.0.0.0/0      0.0.0.0/0  tcp dpt:6000
```

The persistence check should return:

```text
enabled
```

> **Why:** `iptables -L INPUT` lists the incoming rules, `-n` keeps addresses and ports numeric, and `--line-numbers` makes the evaluation order visible. `systemctl is-enabled` reports whether systemd is configured to start the `iptables` service automatically. Together, these checks confirm both the access restriction and reboot persistence on App Server 1.

### 🔁 Step 9: Repeat the complete configuration on App Servers 2 and 3

The full procedure has now been completed and explained on App Server 1. Apply the same installation, firewall rules, persistence, and verification steps to the remaining App Servers. Only the login account and hostname change.

On App Server 2:

```bash
exit
ssh steve@stapp02
sudo yum install -y iptables iptables-services
sudo iptables -I INPUT 1 -p tcp -s 10.244.195.16 --dport 6000 -j ACCEPT
sudo iptables -I INPUT 2 -p tcp --dport 6000 -j DROP
sudo iptables-save -f /etc/sysconfig/iptables
sudo systemctl enable iptables
sudo iptables -L INPUT -n --line-numbers
sudo systemctl is-enabled iptables
```

On App Server 3:

```bash
exit
ssh banner@stapp03
sudo yum install -y iptables iptables-services
sudo iptables -I INPUT 1 -p tcp -s 10.244.195.16 --dport 6000 -j ACCEPT
sudo iptables -I INPUT 2 -p tcp --dport 6000 -j DROP
sudo iptables-save -f /etc/sysconfig/iptables
sudo systemctl enable iptables
sudo iptables -L INPUT -n --line-numbers
sudo systemctl is-enabled iptables
```

> **Why:** The same commands are repeated because all three App Servers require the same security policy. `stapp02` uses `steve`, and `stapp03` uses `banner`; the LBR source IP, port, rule order, persistence file, and boot configuration remain unchanged. The verification commands confirm that each server has the allow rule before the block rule and that `iptables` is enabled.

### ✅ Step 10: Verify all App Servers

On each server, the saved rules should contain:

```text
-A INPUT -s 10.244.195.16/32 -p tcp -m tcp --dport 6000 -j ACCEPT
-A INPUT -p tcp -m tcp --dport 6000 -j DROP
```

Each server should also return:

```text
enabled
```

> **Why:** Checking all three hosts confirms that the requirement was applied consistently across the application tier. The LBR remains the only allowed source for `TCP/6000`, all other sources are blocked, and the configuration will be restored after a reboot.

## Best Practices

- **Resolve the LBR before writing rules.** A source-specific rule requires the actual IP address used by the LBR in the lab.
- **Allow the trusted source before blocking the port.** Firewall rules are evaluated in order, so the LBR exception must precede the general `DROP` rule.
- **Limit rules by protocol, source, and destination port.** The configuration affects only TCP traffic to port `6000` and does not change unrelated traffic.
- **Persist the active ruleset explicitly.** Runtime `iptables` changes are held in memory; saving `/etc/sysconfig/iptables` and enabling the service preserves them across reboots.
- **Do not flush existing firewall rules.** Avoid `iptables -F` because it removes unrelated protections and can expose other services.
- **Verify order and boot enablement separately.** A correct rule set and an enabled service solve different parts of the requirement.

### 📚 Official Documentation

- [Red Hat: Setting and Controlling IP sets using iptables](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/security_guide/sec-Setting_and_Controlling_IP_sets_using_iptables)
- [Red Hat: Saving iptables rules](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/6/html/security_guide/sect-security_guide-iptables-saving_iptables_rules)
- [Netfilter: Using iptables](https://www.netfilter.org/documentation/HOWTO/packet-filtering-HOWTO-7.html)
- [iptables-save(8) manual](https://man7.org/linux/man-pages/man8/iptables-save.8.html)
- [systemd service debugging](https://wiki.freedesktop.org/www/Software/systemd/Debugging/)
