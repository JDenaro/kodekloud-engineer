# Day 14: Linux Process Troubleshooting

The production support team of xFusionCorp Industries has deployed some of the latest monitoring tools to keep an eye on every service, application, etc. running on the systems. One of the monitoring systems reported about Apache service unavailability on one of the app servers in Stratos DC.

Identify the faulty app host and fix the issue. Make sure Apache service is up and running on all app hosts. They might not have hosted any code yet on these servers, so you don't need to worry if Apache isn't serving any pages. Just make sure the service is up and running. Also, make sure Apache is running on port 8088 on all app servers.

## Specific Requirements:

1. Identify the App Server where Apache is unavailable.
2. Fix the issue causing Apache to be unavailable.
3. Ensure the `httpd` service is running on all App Servers.
4. Ensure Apache is listening on port `8088` on all App Servers.

## Solution

The three App Servers were checked individually. `stapp01` was faulty: Apache had failed because `sendmail` was already using port `8088`. `stapp02` and `stapp03` were already running Apache successfully on the required port.

The fix was to stop the process that occupied the required port and start Apache again. No website files or Apache content were changed.

### 🔎 Step 1: Diagnose App Server 1

Connect to App Server 1:

```bash
ssh tony@stapp01
```

Check the Apache service and the process listening on port `8088`:

```bash
sudo systemctl status httpd --no-pager -l
sudo ss -tulnp | grep 8088
```

Apache was not running:

```text
Active: failed (Result: exit-code)
```

The service log reported a port conflict:

```text
(98)Address already in use: AH00072: make_sock: could not bind to address [::]:8088
(98)Address already in use: AH00072: make_sock: could not bind to address 0.0.0.0:8088
```

The `ss` output identified `sendmail` as the process using the port:

```text
tcp LISTEN 0 10 127.0.0.1:8088 0.0.0.0:* users:(("sendmail",pid=57922,fd=4))
```

> **Why:** `ssh` opens a secure remote shell, and `tony` is the login account for App Server 1. `sudo systemctl status` displays the current state and recent messages for the `httpd` service; `--no-pager` keeps the output in the terminal, and `-l` displays complete lines. `ss` investigates network sockets; `-t` selects TCP, `-u` selects UDP, `-l` shows listening sockets, `-n` keeps addresses and ports numeric, and `-p` shows the process using a socket. `grep 8088` filters the result to the required port. The `Address already in use` message means another process had claimed the port before Apache could bind to it.

### 🛑 Step 2: Release port 8088 and start Apache on App Server 1

```bash
sudo systemctl stop sendmail
sudo systemctl start httpd
```

Check the result:

```bash
sudo systemctl status httpd --no-pager -l
sudo ss -tulnp | grep 8088
```

Apache started successfully:

```text
Active: active (running)
Server configured, listening on: port 8088
```

The port was now owned by Apache:

```text
tcp LISTEN 0 511 *:8088 *:* users:(('httpd',pid=84446,fd=4), ...)
```

> **Why:** `systemctl stop sendmail` stops the service that occupied port `8088` and releases its socket. `systemctl start httpd` starts Apache with its existing configuration. The second status and socket checks confirm both parts of the fix: Apache is running, and the required port is owned by `httpd`. The `ServerName` warning in the service output was not fatal because Apache continued to start successfully.

### 🔎 Step 3: Check Apache on App Server 2

Return to the Jump Host and connect to App Server 2:

```bash
exit
exit
ssh steve@stapp02
```

Check the service and port:

```bash
sudo systemctl status httpd --no-pager -l
sudo ss -tulnp | grep 8088
```

`stapp02` was already healthy:

```text
Active: active (running)
Server configured, listening on: port 8088
tcp LISTEN 0 511 *:8088 *:* users:(('httpd',pid=55557,fd=4), ...)
```

> **Why:** The same two checks are used on every App Server so that service state and socket ownership are evaluated consistently. No change was needed on `stapp02` because Apache was already running and listening on the required port.

### 🔎 Step 4: Check Apache on App Server 3

Return to the Jump Host and connect to App Server 3:

```bash
exit
exit
ssh banner@stapp03
```

Check the service and port:

```bash
sudo systemctl status httpd --no-pager -l
sudo ss -tulnp | grep 8088
```

`stapp03` was also healthy:

```text
Active: active (running)
Server configured, listening on: port 8088
tcp LISTEN 0 511 *:8088 *:* users:(('httpd',pid=55622,fd=4), ...)
```

> **Why:** These checks confirm that the third App Server also meets the requirement. No change was needed on `stapp03` because its Apache service and port were already correct.

### ✅ Step 5: Final result

The final state was:

| Server | Apache service | Port `8088` | Action |
| --- | --- | --- | --- |
| `stapp01` | `active (running)` | `httpd` listening | Stopped `sendmail` and started `httpd` |
| `stapp02` | `active (running)` | `httpd` listening | No change required |
| `stapp03` | `active (running)` | `httpd` listening | No change required |

> **Why:** The challenge checks the service state and listening port rather than website content. All three App Servers now have Apache running on `8088`, and the existing web content was left untouched.

## Best Practices

- **Check every candidate host.** The faulty server was not assumed from the task wording; each App Server was inspected.
- **Read the service error before changing configuration.** Apache explicitly reported that port `8088` was already in use.
- **Identify the process owning a port.** `ss -tulnp` showed that `sendmail`, not Apache, owned the required socket.
- **Change only the conflicting service.** Stopping `sendmail` released the port without modifying Apache content or unrelated services.
- **Verify both service state and socket ownership.** `active (running)` alone does not prove that Apache is listening on the required port.
- **Treat non-fatal warnings separately from failures.** The `ServerName` warning did not prevent Apache from starting and listening on `8088`.

### 📚 Official Documentation

- [Apache HTTP Server 2.4 documentation](https://httpd.apache.org/docs/2.4/)
- [Apache HTTP Server program reference](https://httpd.apache.org/docs/current/en/programs/httpd.html)
- [Apache HTTP Server log files](https://httpd.apache.org/docs/2.4/logs.html)
- [ss(8) Linux manual](https://www.man7.org/linux/man-pages/man8/ss.8.html)
- [systemd debugging guidance](https://wiki.freedesktop.org/www/Software/systemd/Debugging/)
