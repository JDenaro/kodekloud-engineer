# Day 15: Setup SSL for Nginx

The system admins team of xFusionCorp Industries needs to deploy a new application on App Server 3 in Stratos Datacenter. They have some pre-requites to get ready that server for application deployment. Prepare the server as per requirements shared below:

1. Install and configure nginx on App Server 3.

2. On App Server 3 there is a self signed SSL certificate and key present at location /tmp/nautilus.crt and /tmp/nautilus.key. Move them to some appropriate location and deploy the same in Nginx.

3. Create an index.html file with content Welcome! under Nginx document root.

4. For final testing try to access the App Server 3 link (via hostname) from jump host using curl command. For example: curl -Ik https://<app-server-name>/.

## Specific Requirements:

1. Install and configure `nginx` on App Server 3.
2. Move `/tmp/nautilus.crt` and `/tmp/nautilus.key` to an appropriate location and configure Nginx to use them.
3. Create `/usr/share/nginx/html/index.html` with the content `Welcome!`.
4. Verify HTTPS access from the Jump Host using `curl -Ik https://stapp03/`.

## Solution

Nginx was installed on App Server 3 (`stapp03`) and configured to serve HTTPS on port `443` using the provided self-signed certificate and private key. The certificate and key were moved to `/etc/nginx/ssl/`, the private key received restrictive permissions, and the requested page was created under Nginx's document root.

### 🔌 Step 1: Connect to App Server 3

From the Jump Host, connect as `banner`:

```bash
ssh banner@stapp03
```

> **Why:** `ssh` opens a secure remote shell. `banner` is the login account for App Server 3, and `stapp03` is the server where Nginx and HTTPS must be configured.

### 📦 Step 2: Install and start Nginx

```bash
sudo yum install -y nginx
```

The installation completed successfully:

```text
Installed:
  nginx-2:1.20.1-31.el9.x86_64
  nginx-core-2:1.20.1-31.el9.x86_64
  nginx-filesystem-2:1.20.1-31.el9.noarch

Complete!
```

Enable and start Nginx:

```bash
sudo systemctl enable --now nginx
```

> **Why:** `sudo` provides the administrative privileges required to install packages and manage services. `yum install` installs Nginx and its dependencies, while `-y` automatically confirms the package operation. `systemctl enable --now` both starts Nginx immediately and configures it to start automatically after a reboot.

### 🔐 Step 3: Move the SSL certificate and private key

Create a dedicated directory and move the provided files:

```bash
sudo mkdir -p /etc/nginx/ssl
sudo mv /tmp/nautilus.crt /etc/nginx/ssl/nautilus.crt
sudo mv /tmp/nautilus.key /etc/nginx/ssl/nautilus.key
```

Set appropriate permissions:

```bash
sudo chmod 600 /etc/nginx/ssl/nautilus.key
sudo chmod 644 /etc/nginx/ssl/nautilus.crt
```

> **Why:** `mkdir -p` creates the SSL directory and any missing parent directories. `mv` moves the certificate and key out of `/tmp` into a dedicated system configuration location. `chmod 600` allows only the file owner to read and write the private key, while `chmod 644` allows the public certificate to be read by Nginx and other processes. Restricting the private key protects the server identity used for TLS.

### ⚙️ Step 4: Configure an HTTPS server block

Create a separate Nginx configuration file:

```bash
sudo vi /etc/nginx/conf.d/nautilus.conf
```

Add this configuration:

```nginx
server {
    listen 443 ssl;
    server_name stapp03;

    ssl_certificate /etc/nginx/ssl/nautilus.crt;
    ssl_certificate_key /etc/nginx/ssl/nautilus.key;

    root /usr/share/nginx/html;
    index index.html;
}
```

Save and exit `vi` with `Esc`, `:wq`, and `Enter`.

> **Why:** `vi` edits the Nginx configuration file. The `server` block defines a virtual server, `listen 443 ssl` enables HTTPS on the standard TLS port, and `server_name stapp03` selects this block when the request uses the App Server 3 hostname. `ssl_certificate` points to the public certificate, and `ssl_certificate_key` points to the private key. `root` sets Nginx's document root, while `index index.html` tells Nginx which file to serve for the root URL.

### 📄 Step 5: Create the web page

Create the requested file under the Nginx document root:

```bash
sudo vi /usr/share/nginx/html/index.html
```

Enter exactly:

```text
Welcome!
```

Save and exit `vi` with `Esc`, `:wq`, and `Enter`.

> **Why:** `/usr/share/nginx/html` is the document root configured for this server block. Creating `index.html` there gives Nginx content to return when a client requests `/`. The requested content is exactly `Welcome!`.

### 🧪 Step 6: Validate and reload Nginx

Test the configuration before applying it:

```bash
sudo nginx -t
```

The syntax check succeeded:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Reload Nginx:

```bash
sudo systemctl reload nginx
```

> **Why:** `nginx -t` checks the configuration syntax and verifies that referenced files can be opened. `systemctl reload` applies the new configuration without fully stopping the running service, which avoids an unnecessary service interruption.

### ✅ Step 7: Verify HTTP and HTTPS from the Jump Host

Return to the Jump Host:

```bash
exit
```

The page was reachable over HTTP:

```bash
curl http://stapp03
```

Output:

```text
Welcome!
```

Verify the required HTTPS endpoint:

```bash
curl -Ik https://stapp03/
```

The HTTPS response was successful:

```text
HTTP/1.1 200 OK
Server: nginx/1.20.1
Content-Type: text/html
Content-Length: 9
```

> **Why:** `curl` sends an HTTP request from the Jump Host to the specified URL. `-I` requests only the response headers, which is enough to verify the HTTP status without printing the page body. `-k` allows the connection to proceed with the self-signed certificate, which is not trusted by a public certificate authority. The `200 OK` response confirms that Nginx served the requested page over HTTPS using the hostname `stapp03`.

## Best Practices

- **Keep private keys separate from temporary storage.** Moving `/tmp/nautilus.key` to `/etc/nginx/ssl/` gives it a stable, purpose-specific location.
- **Restrict private-key permissions.** The `600` mode prevents other users from reading the TLS private key.
- **Validate before reloading.** `nginx -t` catches syntax errors and missing certificate files before the running service is reconfigured.
- **Use a separate server configuration file.** `/etc/nginx/conf.d/nautilus.conf` keeps the application-specific HTTPS block separate from the main Nginx configuration.
- **Use `-k` only for this self-signed test.** In production, clients should validate certificates issued by a trusted certificate authority instead of bypassing verification.
- **Test from the required source host.** Running `curl` from the Jump Host verifies the complete path to App Server 3 rather than only testing Nginx locally.

### 📚 Official Documentation

- [Nginx documentation](https://nginx.org/en/docs/)
- [Nginx: Configuring HTTPS servers](https://nginx.org/en/docs/http/configuring_https_servers.html)
- [Nginx HTTP core module](https://nginx.org/en/docs/http/ngx_http_core_module.html)
- [Nginx index module](https://nginx.org/en/docs/http/ngx_http_index_module.html)
- [Nginx command-line parameters](https://nginx.org/en/docs/switches.html)
- [curl man page](https://curl.se/docs/manpage.html)
