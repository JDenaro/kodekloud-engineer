# Day 12: Linux Network Services

Our monitoring tool has reported an issue in Stratos Datacenter. One of our app servers has an issue, as its Apache service is not reachable on port 8088 (which is the Apache port). The service itself could be down, the firewall could be at fault, or something else could be causing the issue.

Use tools like telnet, netstat, etc. to find and fix the issue. Also make sure Apache is reachable from the jump host without compromising any security settings.

Once fixed, you can test the same using command curl http://stapp02:8088 command from jump host.

Note: Please do not try to alter the existing index.html code, as it will lead to task failure.

## Specific Requirements:

1. Troubleshoot the Apache service and network connectivity for port `8088`.
2. Ensure Apache is reachable from the Jump Host without compromising existing security settings.
3. Verify the result with `curl` from the Jump Host.
4. Do not modify the existing `index.html` file.

## Solution

The task description referenced `stapp02` in the final test, so all three application servers were tested first. `stapp02` and `stapp03` accepted connections, while `stapp01` returned `No route to host`. The actual problem was on `stapp01`: `sendmail` was using port `8088`, which prevented Apache from starting, and an `iptables` rule rejected new connections before they could reach Apache.

The fix was limited to stopping the conflicting `sendmail` service, starting Apache, and adding a specific `iptables` exception for `8088/tcp` before the existing reject rule. The website content was not changed.

### 🔎 Step 1: Test port 8088 on every application server

From `thor@jump-host`, test the same port on all three application servers:

```bash
telnet stapp01 8088
telnet stapp02 8088
telnet stapp03 8088
```

The connection results were:

```text
stapp01: No route to host
stapp02: Connected
stapp03: Connected
```

The two successful connections returned `HTTP/1.1 400 Bad Request` because `telnet` does not send a complete HTTP request. The TCP connection itself was successful. `stapp01` was the server that required troubleshooting.

> **Why:** `telnet` opens a raw TCP connection to a host and port, which is useful for checking reachability before investigating the application protocol. The first argument identifies the server, and the second argument identifies the TCP port. A successful `Connected` message proves that the network path and a listening process are available; an HTTP `400` response from a raw telnet session is expected because the input is not a valid browser-style HTTP request. Press `Ctrl+]`, type `quit`, and press `Enter` to leave an interactive telnet session.

### 🔌 Step 2: Connect to the affected server

```bash
ssh tony@stapp01
```

> **Why:** `ssh` opens a secure remote shell. `tony` is the sudo-enabled user for App Server 1, and `stapp01` is the server identified by the connectivity tests.

### 🔎 Step 3: Check Apache and the listening port

```bash
sudo systemctl status httpd --no-pager -l
sudo ss -tulnp | grep 8088
```

Apache was failed because the port was already occupied:

```text
(98)Address already in use: AH00072: make_sock: could not bind to address [::]:8088
(98)Address already in use: AH00072: make_sock: could not bind to address 0.0.0.0:8088
```

The process using the port was `sendmail`:

```text
tcp LISTEN 0 10 127.0.0.1:8088 0.0.0.0:* users:(("sendmail",pid=29920,fd=4))
```

> **Why:** `sudo` runs the diagnostic commands with the privileges needed to inspect system services and sockets. `systemctl status` displays the state and recent messages for the `httpd` service; `--no-pager` keeps the output in the terminal, and `-l` shows complete lines. `ss` displays listening network sockets; `-t` selects TCP, `-u` selects UDP, `-l` shows listening sockets, and `-n` keeps addresses and ports numeric. `-p` displays the process using each socket. `grep 8088` keeps only the lines related to the required port. The `Address already in use` message means another process had bound the port before Apache could use it.

### 🛑 Step 4: Stop the process that owns port 8088

```bash
sudo systemctl stop sendmail
```

> **Why:** `systemctl stop` asks systemd to stop the named service and release its listening socket. Only the service occupying the required port was stopped; the firewall was not disabled and no website files were changed.

### ▶️ Step 5: Start Apache

```bash
sudo systemctl start httpd
```

Apache started successfully and reported that it was listening on port `8088`:

```text
Active: active (running)
Server configured, listening on: port 8088
```

> **Why:** `systemctl start` launches the Apache service using its existing configuration. Starting the service after releasing the port tests whether the port conflict was the cause of the failure. The `ServerName` warning shown by Apache was unrelated to the failure because Apache continued to start successfully.

### 🧱 Step 6: Inspect the host firewall rules

```bash
sudo iptables -L INPUT -n -v --line-numbers
```

The relevant rules were:

```text
4  ACCEPT  tcp  --  0.0.0.0/0  0.0.0.0/0  state NEW tcp dpt:22
5  REJECT  all  --  0.0.0.0/0  0.0.0.0/0  reject-with icmp-host-prohibited
```

The final `REJECT` rule blocked new inbound connections that were not explicitly allowed earlier in the chain.

> **Why:** `iptables` manages packet-filtering rules. `-L INPUT` lists rules in the `INPUT` chain, which handles packets arriving at the server. `-n` keeps addresses and ports numeric, `-v` shows packet and byte counters, and `--line-numbers` displays the rule positions. Rules are evaluated from top to bottom, so the existing reject rule had to remain in place while a narrow allow rule was inserted before it.

### 🔓 Step 7: Allow only the required Apache port

```bash
sudo iptables -I INPUT 5 -p tcp --dport 8088 -j ACCEPT
```

The resulting rule order included:

```text
5  ACCEPT  tcp  --  0.0.0.0/0  0.0.0.0/0  tcp dpt:8088
6  REJECT  all  --  0.0.0.0/0  0.0.0.0/0  reject-with icmp-host-prohibited
```

> **Why:** `-I INPUT 5` inserts a rule at position `5`, immediately before the existing reject rule. `-p tcp` limits the exception to TCP traffic, `--dport 8088` limits it to the Apache port, and `-j ACCEPT` permits matching packets. This preserves the reject rule and opens only the required service port instead of disabling the firewall.

### ✅ Step 8: Verify Apache from the Jump Host

Leave App Server 1 and run the required connectivity test from the Jump Host:

```bash
exit
curl http://stapp01:8088
```

The request returned the existing Apache test page. The connection was successful, and the `index.html` content was not modified.

> **Why:** `exit` closes the remote SSH session and returns to the Jump Host. `curl` transfers data from the specified URL and therefore tests the complete path: DNS or host resolution, network connectivity, firewall access, Apache listening state, and HTTP response handling. `stapp01` identifies the affected server, and `8088` identifies its Apache port. The returned HTML confirms that the service is reachable from the required source host.

## Best Practices

- **Test all candidate servers when the mapping is uncertain.** The task text referenced `stapp02`, but port testing identified `stapp01` as the actual affected server.
- **Resolve port conflicts before changing application configuration.** The Apache logs identified `sendmail` as the process already using `8088`.
- **Inspect firewall rules instead of disabling the firewall.** A single port-specific allow rule preserved the existing `REJECT` protection for other inbound traffic.
- **Use socket inspection to identify the owning process.** `ss -tulnp` connected the port conflict directly to the `sendmail` process.
- **Do not change application content during network troubleshooting.** The existing website files were left untouched, as required by the challenge.
- **Interpret diagnostic tools correctly.** A `400 Bad Request` from telnet means Apache received an incomplete HTTP request; it does not mean the TCP connection failed.

### 📚 Official Documentation

- [Apache HTTP Server 2.4 documentation](https://httpd.apache.org/docs/2.4/)
- [Apache HTTP Server: Binding to Addresses and Ports](https://httpd.apache.org/docs/2.4/bind.html)
- [Netfilter: Using iptables](https://www.netfilter.org/documentation/HOWTO/packet-filtering-HOWTO-7.html)
- [systemd service debugging](https://wiki.freedesktop.org/www/Software/systemd/Debugging/)
- [curl man page](https://curl.se/docs/manpage.html)
