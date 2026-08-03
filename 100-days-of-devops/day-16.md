# Day 16: Install and Configure Nginx as an LBR

Day by day traffic is increasing on one of the websites managed by the Nautilus production support team. Therefore, the team has observed a degradation in website performance. Following discussions about this issue, the team has decided to deploy this application on a high availability stack i.e on Nautilus infra in Stratos DC. They started the migration last month and it is almost done, as only the LBR server configuration is pending. Configure LBR server as per the information given below:

a. Install nginx on the LBR (load balancer) server if it is not already installed.

b. Configure load-balancing with the http context making use of all App Servers. Ensure that you update only the main Nginx configuration file located at /etc/nginx/nginx.conf.

c. Make sure you do not update the apache port that is already defined in the apache configuration on all app servers, also make sure apache service is up and running on all the app servers.

d. Once done, you can access the website by running curl http://stlb01:80 in the terminal.

## Specific Requirements:

1. Install `nginx` on the LBR server if it is not already installed.
2. Configure HTTP load balancing using all App Servers and update only `/etc/nginx/nginx.conf`.
3. Preserve the existing Apache port and ensure Apache is running on all App Servers.
4. Make the website available through `curl http://stlb01:80`.

## Solution

The LBR was `stlb01`, accessed with the `loki` user. Nginx was already installed and was upgraded by the package operation. The three App Servers were checked before configuring the load balancer; Apache was active on all of them and each server listened on port `6200`.

The LBR configuration was added only to `/etc/nginx/nginx.conf`. An `upstream` group listed all three App Servers, and the existing HTTP server block proxied incoming requests to that group. Nginx's default upstream method distributes requests using weighted round-robin, so no Apache configuration changes were necessary.

### 🔌 Step 1: Connect to the LBR

From the Jump Host, connect to `stlb01`:

~~~
ssh loki@stlb01
~~~

> **Why:** `ssh` opens a secure remote shell. `loki` is the login account for the LBR, and `stlb01` is the server that will receive client requests and forward them to the App Servers.

### 📦 Step 2: Install and start Nginx on the LBR

~~~
sudo yum install -y nginx
~~~

Nginx was already present and was upgraded to the current repository version:

~~~
Upgraded:
  nginx-2:1.20.1-31.el9.x86_64
  nginx-core-2:1.20.1-31.el9.x86_64
  nginx-filesystem-2:1.20.1-31.el9.noarch

Complete!
~~~

Enable and start the service:

~~~
sudo systemctl enable --now nginx
~~~

> **Why:** `sudo` provides the administrative privileges required to install packages and manage services. `yum install` installs Nginx and its dependencies; `-y` confirms the package operation automatically. `systemctl enable --now` starts Nginx immediately and configures it to start automatically after a reboot. Running the package command is safe when Nginx is already installed because the package manager reports the existing state and applies available updates.

### 🔎 Step 3: Check Apache on App Server 1

Leave the LBR and connect to App Server 1:

~~~
exit
exit
ssh tony@stapp01
~~~

Check Apache's service state and listening port:

~~~
sudo systemctl status httpd --no-pager -l
sudo ss -tulnp | grep httpd
~~~

The result showed Apache running on port `6200`:

~~~
Active: active (running)
Server configured, listening on: port 6200
tcp LISTEN 0 511 *:6200 *:* users:(("httpd",pid=19739,fd=4), ...)
~~~

> **Why:** `systemctl status` displays the current state and recent messages for the `httpd` service. `--no-pager` keeps the output in the terminal, and `-l` shows complete lines. `ss` inspects network sockets; `-tulnp` selects TCP and UDP sockets, limits the output to listening sockets, keeps addresses numeric, and displays the owning process. `grep httpd` keeps the Apache socket line. The port must be discovered and reused exactly as configured; changing it would break the existing application setup.

### 🔎 Step 4: Check Apache on App Server 2

~~~
exit
exit
ssh steve@stapp02
sudo systemctl status httpd --no-pager -l
sudo ss -tulnp | grep httpd
~~~

App Server 2 was also healthy and used port `6200`:

~~~
Active: active (running)
Server configured, listening on: port 6200
tcp LISTEN 0 511 *:6200 *:* users:(("httpd",pid=19626,fd=4), ...)
~~~

> **Why:** The same checks confirm that the second backend is available before it is added to the load-balancing pool. No Apache configuration was changed.

### 🔎 Step 5: Check Apache on App Server 3

~~~
exit
exit
ssh banner@stapp03
sudo systemctl status httpd --no-pager -l
sudo ss -tulnp | grep httpd
~~~

App Server 3 was healthy and also used port `6200`:

~~~
Active: active (running)
Server configured, listening on: port 6200
tcp LISTEN 0 511 *:6200 *:* users:(("httpd",pid=19513,fd=4), ...)
~~~

> **Why:** Checking the third backend completes the pre-configuration health check. All three App Servers were active and shared the same existing Apache port, so the LBR can use `stapp01:6200`, `stapp02:6200`, and `stapp03:6200` without changing Apache.

### ⚙️ Step 6: Configure the upstream group in the main Nginx file

Return to the LBR:

~~~
exit
exit
ssh loki@stlb01
~~~

Edit only the required main configuration file:

~~~
sudo vi /etc/nginx/nginx.conf
~~~

Inside the existing `http { ... }` context, add the upstream group before the existing `server` block:

~~~nginx
upstream app_servers {
    server stapp01:6200;
    server stapp02:6200;
    server stapp03:6200;
}
~~~

Inside the existing HTTP `server` block, configure the root location to proxy requests to the upstream group:

~~~nginx
location / {
    proxy_pass http://app_servers;
}
~~~

The relevant structure should be:

~~~nginx
http {
    upstream app_servers {
        server stapp01:6200;
        server stapp02:6200;
        server stapp03:6200;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://app_servers;
        }
    }
}
~~~

Keep the other existing directives in `/etc/nginx/nginx.conf` unchanged. Save and exit `vi` with `Esc`, `:wq`, and `Enter`.

> **Why:** `/etc/nginx/nginx.conf` is the main Nginx configuration file required by the challenge. The `http` context is where HTTP-related upstream groups and server blocks are defined. `upstream app_servers` names a backend group, and each `server` entry identifies one App Server and its existing Apache port. `proxy_pass http://app_servers` forwards requests received by the LBR to that group. Nginx distributes requests between the listed servers using its default weighted round-robin method. No file in `/etc/nginx/conf.d/` was created or modified, and no Apache port was changed.

### 🧪 Step 7: Validate and reload the LBR configuration

Validate the configuration:

~~~
sudo nginx -t
~~~

The validation succeeded:

~~~
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
~~~

Apply the configuration without stopping Nginx:

~~~
sudo systemctl reload nginx
~~~

Check the service:

~~~
sudo systemctl status nginx --no-pager
~~~

Nginx remained active after the reload:

~~~
Active: active (running)
Reloaded The nginx HTTP and reverse proxy server.
~~~

> **Why:** `nginx -t` checks the configuration syntax and verifies that referenced files can be opened. `systemctl reload` applies the new configuration while keeping the service running, which avoids an unnecessary interruption. `systemctl status` confirms that the LBR remains active after the configuration change.

### ✅ Step 8: Verify the website through the LBR

Return to the Jump Host:

~~~
exit
~~~

Request the website through port `80` on the LBR:

~~~
curl http://stlb01:80
~~~

The request returned the application page:

~~~
Welcome to xFusionCorp Industries!
~~~

> **Why:** `curl` sends an HTTP request from the Jump Host. `http://stlb01:80` targets the LBR hostname and its HTTP listener. The returned application content confirms the complete path: the LBR accepted the request, selected an upstream App Server, forwarded the request to Apache on port `6200`, and returned the backend response.

## Best Practices

- **Discover backend ports before configuring the LBR.** Reusing the existing Apache port (`6200`) avoids breaking the App Servers.
- **Check backend health first.** Confirming `httpd` is active and listening on every App Server prevents adding an unavailable backend to the pool.
- **Use the required configuration file.** The challenge specifically requires updating only `/etc/nginx/nginx.conf`; no additional Nginx configuration file was created.
- **Keep upstream names descriptive.** `app_servers` clearly identifies the backend group used by `proxy_pass`.
- **Validate before reloading.** Always run `nginx -t` before applying a configuration change.
- **Reload instead of stopping the service.** A reload applies the new configuration while preserving the running Nginx process.
- **Verify through the load balancer.** Testing `curl http://stlb01:80` confirms the client-to-LBR-to-backend path.

### 📚 Official Documentation

- [Nginx documentation](https://nginx.org/en/docs/)
- [Nginx HTTP load balancing](https://nginx.org/en/docs/http/load_balancing.html)
- [Nginx upstream module](https://nginx.org/en/docs/http/ngx_http_upstream_module.html)
- [Nginx proxy module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
- [Nginx command-line parameters](https://nginx.org/en/docs/switches.html)
- [curl man page](https://curl.se/docs/manpage.html)
