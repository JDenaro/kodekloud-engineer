# Day 20: Configure Nginx + PHP-FPM Using Unix Sock

The Nautilus application development team is planning to launch a new PHP-based application, which they want to deploy on Nautilus infra in Stratos DC. The development team had a meeting with the production support team and they have shared some requirements regarding the infrastructure. Below are the requirements they shared:

a. Install nginx on app server 3 , configure it to use port 8092 and its document root should be /var/www/html.

b. Install php-fpm version 8.3 on app server 3, it must use the unix socket /var/run/php-fpm/default.sock (create the parent directories if don't exist).

c. Configure php-fpm and nginx to work together.

d. Once configured correctly, you can test the website using curl http://stapp03:8092/index.php command from jump host.

NOTE: We have copied two files, index.php and info.php, under /var/www/html as part of the PHP-based application setup. Please do not modify these files.

## Specific Requirements:

1. Install Nginx on App Server 3 and configure it to listen on port 8092 with document root /var/www/html.
2. Install PHP-FPM version 8.3 on App Server 3.
3. Configure PHP-FPM to use the Unix socket /var/run/php-fpm/default.sock, creating the parent directory if necessary.
4. Configure Nginx and PHP-FPM to work together.
5. Do not modify /var/www/html/index.php or /var/www/html/info.php.
6. Verify the application with curl http://stapp03:8092/index.php from the Jump Host.

## Solution

App Server 3 was stapp03, accessed with the banner user. PHP-FPM was installed from the explicitly enabled php:8.3 module stream, which installed version 8.3.19. PHP-FPM was configured to listen on the Unix socket /var/run/php-fpm/default.sock, and Nginx was configured to serve /var/www/html on port 8092 and forward PHP requests to that socket.

The application files index.php and info.php were left unchanged. The final request returned Welcome to xFusionCorp Industries!, confirming that Nginx passed the request to PHP-FPM and returned the generated response.

### 🔌 Step 1: Connect to App Server 3

From the Jump Host, connect to App Server 3:

~~~bash
ssh banner@stapp03
~~~

> **Why:** ssh opens a secure remote shell. banner is the sudo-enabled login account for App Server 3, and stapp03 is the server where the PHP application must be configured.

### 📦 Step 2: Select PHP 8.3 and install the packages

Select the PHP 8.3 module stream before installing PHP-FPM:

~~~bash
sudo yum module reset php -y
sudo yum module enable php:8.3 -y
~~~

Install Nginx and PHP-FPM:

~~~bash
sudo yum install -y nginx php-fpm
~~~

Confirm the installed PHP-FPM package version:

~~~bash
rpm -q php-fpm
~~~

The package operation installed the required versions:

~~~text
nginx-2:1.20.1-31.el9.x86_64
php-fpm-8.3.19-1.module_el9+1213+23b849bf.x86_64
~~~

> **Why:** yum module reset clears a previous PHP stream selection, while yum module enable php:8.3 selects the required PHP 8.3 stream. Selecting the stream before installation prevents yum from choosing an older default such as PHP 8.0. yum install installs Nginx and PHP-FPM with their dependencies; -y accepts the package manager prompts. Nginx receives HTTP requests, and PHP-FPM executes PHP scripts through FastCGI. rpm -q php-fpm queries the installed RPM package and confirms the exact version.

### 📁 Step 3: Create the PHP-FPM socket directory

Create the parent directory required by the task:

~~~bash
sudo mkdir -p /var/run/php-fpm
~~~

> **Why:** mkdir creates directories, and -p also creates missing parent directories without failing if the directory already exists. /var/run is the runtime directory used by services, and php-fpm is the directory that will contain the Unix socket.

### ⚙️ Step 4: Configure PHP-FPM to use the Unix socket

Open the default PHP-FPM pool configuration:

~~~bash
sudo vi /etc/php-fpm.d/www.conf
~~~

Set the active listen directive to:

~~~ini
listen = /var/run/php-fpm/default.sock
~~~

Keep the existing active ACL setting:

~~~ini
listen.acl_users = apache,nginx
~~~

Save and exit vi with Esc, :wq, and Enter.

Validate the PHP-FPM configuration:

~~~bash
sudo php-fpm -t
~~~

The validation succeeded:

~~~text
NOTICE: configuration file /etc/php-fpm.conf test is successful
~~~

> **Why:** /etc/php-fpm.d/www.conf defines the www PHP-FPM pool. The listen directive changes PHP-FPM from a network listener to a local Unix socket. A Unix socket keeps PHP-FPM communication on the same server and avoids opening a separate TCP port. listen.acl_users grants the web-server users access to the socket; the existing apache,nginx value includes Nginx. php-fpm -t checks the configuration without starting or restarting the service.

### ▶️ Step 5: Start PHP-FPM and confirm the socket

Enable PHP-FPM at boot and start it immediately:

~~~bash
sudo systemctl enable php-fpm --now
~~~

Check the service and socket:

~~~bash
sudo systemctl status php-fpm --no-pager -l
ls -l /var/run/php-fpm/default.sock
~~~

The service and socket were available:

~~~text
Active: active (running)
srw-rw----+ 1 root root 0 Aug  4 09:06 /var/run/php-fpm/default.sock
~~~

> **Why:** systemctl manages services through systemd. enable makes PHP-FPM start after future reboots, and --now starts it immediately. systemctl status displays the service state, while ls -l confirms that the requested Unix socket was created. The + on the socket permissions indicates that ACL entries are present.

### 🌐 Step 6: Configure Nginx for port 8092 and PHP-FPM

Create a dedicated Nginx server configuration:

~~~bash
sudo vi /etc/nginx/conf.d/php-app.conf
~~~

Add:

~~~nginx
server {
    listen 8092;
    server_name stapp03;

    root /var/www/html;
    index index.php index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_pass unix:/var/run/php-fpm/default.sock;
    }
}
~~~

Save and exit vi with Esc, :wq, and Enter.

Test the Nginx configuration:

~~~bash
sudo nginx -t
~~~

The validation succeeded:

~~~text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
~~~

> **Why:** /etc/nginx/conf.d/php-app.conf is loaded by the main Nginx configuration and keeps this application configuration separate. listen 8092 makes Nginx accept HTTP requests on the required port. root sets the document root to /var/www/html, where the provided application files already exist. The PHP location matches requests ending in .php and sends them to PHP-FPM with fastcgi_pass over the requested Unix socket. fastcgi_param SCRIPT_FILENAME gives PHP-FPM the complete path to the requested script. try_files returns 404 for missing paths instead of forwarding arbitrary nonexistent paths to PHP. nginx -t validates the complete Nginx configuration before the service is started.

### ▶️ Step 7: Start Nginx

Enable Nginx at boot and start it:

~~~bash
sudo systemctl enable nginx --now
~~~

Check the service:

~~~bash
sudo systemctl status nginx --no-pager -l
~~~

Nginx was active and its configuration test passed during startup:

~~~text
Active: active (running)
nginx: configuration file /etc/nginx/nginx.conf test is successful
~~~

> **Why:** systemctl enable nginx configures Nginx to start automatically after a reboot, and --now starts it immediately. Nginx validates its configuration before starting through the service's startup checks. The service status confirms that the web server is running.

### ✅ Step 8: Verify PHP processing from the Jump Host

Exit the server and return to the Jump Host:

~~~bash
exit
exit
~~~

From thor@jump-host, request the PHP application:

~~~bash
curl http://stapp03:8092/index.php
~~~

The request returned:

~~~text
Welcome to xFusionCorp Industries!
~~~

> **Why:** curl sends an HTTP request from the Jump Host. The URL targets stapp03 on port 8092 and requests index.php. The response is generated application output rather than PHP source code, proving that Nginx served the request, forwarded the PHP script to PHP-FPM through the Unix socket, and returned the processed response.

## Best Practices

- **Select the PHP stream before installation.** Enabling php:8.3 before installing php-fpm avoids accidentally installing an older default stream.
- **Use a Unix socket for local FastCGI communication.** The socket keeps PHP-FPM off the network and limits communication to the local host.
- **Restrict socket access.** The existing listen.acl_users = apache,nginx setting grants access to the web-server users without making the socket world-readable.
- **Keep application files unchanged.** index.php and info.php were used as provided by the lab.
- **Protect diagnostic scripts.** info.php is useful for troubleshooting, but should be removed or restricted in production because it can expose environment details.
- **Validate both configurations before starting services.** php-fpm -t and nginx -t catch syntax errors before they cause runtime failures.
- **Use a dedicated Nginx server block.** Keeping the application in /etc/nginx/conf.d/php-app.conf avoids unnecessary changes to the main Nginx configuration.
- **Use an explicit PHP file path.** SCRIPT_FILENAME prevents PHP-FPM from guessing which file to execute.
- **Return 404 for missing paths.** try_files avoids routing arbitrary nonexistent paths to the PHP interpreter.
- **Enable services for persistence.** Enabling PHP-FPM and Nginx preserves the configuration across a reboot.

### 📚 Official Documentation

- [Nginx listen directive](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen)
- [Nginx root and try_files directives](https://nginx.org/en/docs/http/ngx_http_core_module.html)
- [Nginx FastCGI module](https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html)
- [Nginx Beginner's Guide](https://nginx.org/en/docs/beginners_guide.html)
- [PHP-FPM installation](https://www.php.net/manual/en/install.fpm.php)
- [PHP-FPM configuration](https://www.php.net/manual/en/install.fpm.configuration.php)
