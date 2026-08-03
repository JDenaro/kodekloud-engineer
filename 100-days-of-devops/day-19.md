# Day 19: Install and Configure Web Application

xFusionCorp Industries is planning to host two static websites on their infra in Stratos Datacenter. The development of these websites is still in-progress, but we want to get the servers ready. Please perform the following steps to accomplish the task:

a. Install httpd package and dependencies on app server 2.

b. Apache should serve on port 3000.

c. There are two website's backups /home/thor/news and /home/thor/apps on jump_host. Set them up on Apache in a way that news should work on the link http://localhost:3000/news/ and apps should work on link http://localhost:3000/apps/ on the mentioned app server.

d. Once configured you should be able to access the website using curl command on the respective app server, i.e curl http://localhost:3000/news/ and curl http://localhost:3000/apps/

## Specific Requirements:

1. Install httpd package and dependencies on App Server 2.
2. Configure Apache to serve on port 3000.
3. Configure the `news` backup to work at `http://localhost:3000/news/`.
4. Configure the `apps` backup to work at `http://localhost:3000/apps/`.
5. Verify both websites with `curl` on App Server 2.

## Solution

App Server 2 was `stapp02`, accessed with the `steve` user. Apache was installed with its dependencies, and the default `Listen 80` directive was changed to `Listen 3000`. The existing Apache `DocumentRoot` remained `/var/www/html`; each backup was placed in its own subdirectory so Apache could serve the two websites by path.

The `AH00558` message about the fully qualified domain name was only a warning. The configuration test returned `Syntax OK`, and Apache started successfully on port `3000`.

### 🔌 Step 1: Connect to App Server 2

From the Jump Host, connect to App Server 2:

~~~bash
ssh steve@stapp02
~~~

> **Why:** `ssh` opens a secure remote shell. `steve` is the sudo-enabled account for App Server 2, and `stapp02` is the target host for this challenge.

### 📦 Step 2: Install Apache and its dependencies

Install the `httpd` package:

~~~bash
sudo yum install -y httpd
~~~

The package operation completed successfully:

~~~text
Installed:
  apr-1.7.0-12.el9.x86_64
  apr-util-1.6.1-23.el9.x86_64
  httpd-2.4.62-14.el9.x86_64
  httpd-core-2.4.62-14.el9.x86_64
  httpd-filesystem-2.4.62-14.el9.noarch
  httpd-tools-2.4.62-14.el9.x86_64
  mod_http2-2.0.26-6.el9.x86_64
  mod_lua-2.4.62-14.el9.x86_64

Complete!
~~~

> **Why:** `sudo` provides the administrative privileges required to install system packages. `yum install` installs Apache and resolves its dependencies; `-y` automatically accepts the package manager's confirmation prompt. The `httpd` package provides the Apache HTTP Server, while packages such as `httpd-core`, `httpd-filesystem`, and `httpd-tools` provide the server core, standard filesystem layout, and administration utilities. The APR packages provide Apache Portable Runtime components used by Apache.

### ⚙️ Step 3: Configure Apache to listen on port 3000

Open Apache's main configuration file:

~~~bash
sudo vi /etc/httpd/conf/httpd.conf
~~~

Find this directive:

~~~apache
Listen 80
~~~

Change it to:

~~~apache
Listen 3000
~~~

Save and exit `vi` with `Esc`, `:wq`, and `Enter`.

Do not change the existing `DocumentRoot` or any website content.

> **Why:** `/etc/httpd/conf/httpd.conf` is Apache's main configuration file. The `Listen` directive controls the TCP port on which Apache accepts HTTP requests. Changing only the port from `80` to `3000` satisfies the task while preserving the default `DocumentRoot` of `/var/www/html`. The warning about the fully qualified domain name does not prevent Apache from starting and does not need to be changed for this lab.

### 🧪 Step 4: Test the Apache configuration

Validate the configuration before starting the service:

~~~bash
sudo httpd -t
~~~

The command returned:

~~~text
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.244.13.62. Set the 'ServerName' directive globally to suppress this message
Syntax OK
~~~

> **Why:** `httpd -t` checks Apache configuration syntax without starting the server. `Syntax OK` confirms that Apache can parse the configuration. `AH00558` is a non-fatal warning about the server name, so no additional change is required for this challenge.

### 📤 Step 5: Copy the website backups to App Server 2

Return to the Jump Host:

~~~bash
exit
exit
~~~

Copy both backup directories to a temporary location on App Server 2:

~~~bash
scp -r /home/thor/news /home/thor/apps steve@stapp02:/tmp/
~~~

The transfer completed successfully:

~~~text
index.html  100%  117  ... 
index.html  100%  117  ...
~~~

> **Why:** `scp` securely copies files over SSH. `-r` recursively copies each directory and its contents. The two source paths are the backups stored on the Jump Host, and `steve@stapp02:/tmp/` is the temporary destination on App Server 2. Using a temporary location avoids requiring the non-root SSH account to write directly into Apache's protected document root.

### 📁 Step 6: Place each website under the Apache document root

Reconnect to App Server 2:

~~~bash
ssh steve@stapp02
~~~

Create the two website directories:

~~~bash
sudo mkdir -p /var/www/html/news /var/www/html/apps
~~~

Copy the contents of the backups into their corresponding directories:

~~~bash
sudo cp -r /tmp/news/. /var/www/html/news/
sudo cp -r /tmp/apps/. /var/www/html/apps/
~~~

Check that both index files are present:

~~~bash
sudo ls -l /var/www/html/news /var/www/html/apps
~~~

The result showed:

~~~text
/var/www/html/apps:
-rw-r--r-- 1 root root 117 Aug  3 15:00 index.html

/var/www/html/news:
-rw-r--r-- 1 root root 117 Aug  3 15:00 index.html
~~~

> **Why:** `mkdir -p` creates both destination directories and does not report an error if they already exist. `cp -r` recursively copies directory contents; the `/.` source notation copies the contents, including possible hidden files, without creating an extra nested directory. Apache maps `/news/` to `/var/www/html/news/` and `/apps/` to `/var/www/html/apps/` because the default `DocumentRoot` is `/var/www/html`. The `644`-style permissions shown by `ls` allow Apache to read the static files without making them executable.

### ▶️ Step 7: Enable and restart Apache

Validate the configuration again, enable Apache at boot, and restart it so it reads the new port:

~~~bash
sudo httpd -t
sudo systemctl enable httpd
sudo systemctl restart httpd
sudo systemctl status httpd --no-pager -l
~~~

The service became active and reported port `3000`:

~~~text
Syntax OK
Active: active (running)
Status: "Started, listening on: port 3000"
Server configured, listening on: port 3000
~~~

> **Why:** Running `httpd -t` again confirms that the final configuration is valid. `systemctl enable` configures Apache to start automatically after a reboot. `systemctl restart` stops and starts the service so it rereads `httpd.conf`; this is required after changing the `Listen` directive. `systemctl status` displays the resulting service state and recent startup messages. The service was not exposed through an additional application process; it serves only the requested static files through Apache.

### ✅ Step 8: Verify both websites

From App Server 2, request both paths:

~~~bash
curl http://localhost:3000/news/
curl http://localhost:3000/apps/
~~~

The first request returned the news page:

~~~html
<!DOCTYPE html>
<html>
<body>

<h1>KodeKloud</h1>

<p>This is a sample page for our news website</p>

</body>
</html>
~~~

The second request returned the apps page:

~~~html
<!DOCTYPE html>
<html>
<body>

<h1>KodeKloud</h1>

<p>This is a sample page for our apps website</p>

</body>
</html>
~~~

> **Why:** `curl` sends HTTP requests from the App Server. `localhost:3000` targets Apache on the local host and the required port. The trailing slash identifies each target as a directory, allowing Apache's directory handling to serve the corresponding `index.html`. Receiving different page content from `/news/` and `/apps/` confirms that both backups are mapped correctly.

## Best Practices

- **Keep the default document root.** Using `/var/www/html` and separate subdirectories keeps the two static sites isolated by URL path.
- **Change only the required listener.** Updating `Listen 80` to `Listen 3000` satisfies the requirement without changing website content or unrelated Apache directives.
- **Validate before restarting.** Run `httpd -t` before applying configuration changes so syntax errors are caught safely.
- **Use a temporary transfer location.** Copying backups to `/tmp` first avoids granting the SSH user unnecessary write access to Apache's document root.
- **Use read-only web content permissions.** The `644` permissions allow Apache to read the files while preventing ordinary users from modifying them.
- **Preserve static content boundaries.** Keep `news` and `apps` in separate directories so a path intended for one site cannot accidentally serve the other site's files.
- **Avoid unnecessary configuration changes.** The `ServerName` warning was non-fatal and did not need to be changed for the lab.

### 📚 Official Documentation

- [Apache HTTP Server: Binding to Addresses and Ports](https://httpd.apache.org/docs/2.4/bind.html)
- [Apache HTTP Server: Configuration Files](https://httpd.apache.org/docs/2.4/configuring.html)
- [Apache HTTP Server: Core Features and Directives](https://httpd.apache.org/docs/2.4/mod/core.html)
- [Apache HTTP Server: mod_dir Directory Indexing](https://httpd.apache.org/docs/2.4/mod/mod_dir.html)
- [Apache HTTP Server: Starting Apache](https://httpd.apache.org/docs/2.4/invoking.html)
- [curl Documentation](https://curl.se/docs/)

