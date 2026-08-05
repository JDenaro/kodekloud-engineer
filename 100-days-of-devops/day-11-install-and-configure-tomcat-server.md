# Day 11: Install and Configure Tomcat Server

The Nautilus application development team recently finished the beta version of one of their Java-based applications, which they are planning to deploy on one of the app servers in Stratos DC. After an internal team meeting, they have decided to use the tomcat application server. Based on the requirements mentioned below complete the task:

a. Install tomcat server on App Server 1.

b. Configure it to run on port 8087.

c. There is a ROOT.war file on Jump host at location /tmp.

Deploy it on this tomcat server and make sure the webpage works directly on base URL i.e curl http://stapp01:8087

## Specific Requirements:

1. Install Tomcat on App Server 1.
2. Configure Tomcat to listen on port `8087`.
3. Deploy `/tmp/ROOT.war` from the Jump Host.
4. Make the application available at `http://stapp01:8087`.

## Solution

Tomcat was installed on App Server 1 (`stapp01`) using the `tomcat` package. The default HTTP connector port was changed from `8080` to `8087` in `/etc/tomcat/server.xml`.

The WAR file was copied from the Jump Host to App Server 1 and placed in `/var/lib/tomcat/webapps/` with the name `ROOT.war`. The `ROOT` name makes the application available directly at the base URL rather than under an additional context path.

### 🔌 Step 1: Connect to App Server 1

From the Jump Host, connect as `tony`:

```bash
ssh tony@stapp01
```

The session opened on App Server 1:

```text
thor@jump-host ~$ ssh tony@stapp01
[tony@stapp01 ~]$
```

> **Why:** `ssh` opens a secure remote shell. `tony` is the login account for App Server 1, and `stapp01` is the server where Tomcat must be installed and configured.

### 📦 Step 2: Install Tomcat

```bash
sudo yum install -y tomcat
```

The package installation completed successfully:

```text
Installed:
  tomcat-1:9.0.117-2.el9.noarch
  tomcat-lib-1:9.0.117-2.el9.noarch
  tomcat-servlet-4.0-api-1:9.0.117-2.el9.noarch

Complete!
```

> **Why:** `sudo` provides administrative privileges, `yum` manages packages on the server, `install` requests the Tomcat package, and `-y` automatically confirms the package operation. The installation also added Java and Tomcat API dependencies required by the server.

### ⚙️ Step 3: Configure Tomcat to use port 8087

Open the Tomcat server configuration:

```bash
sudo vi /etc/tomcat/server.xml
```

Find the HTTP connector using `8080` and change only its port value:

```xml
<Connector port="8087" protocol="HTTP/1.1"
```

Save and exit `vi` with `Esc`, `:wq`, and `Enter`.

> **Why:** `server.xml` is Tomcat's main container configuration file. The HTTP `Connector` accepts web requests, and changing its `port` from the default `8080` to `8087` makes Tomcat listen on the port required by the challenge. Other connectors, such as the shutdown or AJP connectors, must not be changed.

### ▶️ Step 4: Enable and start Tomcat

```bash
sudo systemctl enable --now tomcat
```

The service was enabled and started:

```text
Created symlink /etc/systemd/system/multi-user.target.wants/tomcat.service → /usr/lib/systemd/system/tomcat.service.
```

> **Why:** `systemctl` controls services managed by `systemd`. `enable` configures Tomcat to start automatically after future reboots, while `--now` starts it immediately with the updated configuration.

### 📤 Step 5: Copy ROOT.war from the Jump Host

Exit the App Server 1 session:

```bash
exit
```

From `thor@jump-host`, copy the WAR file to a temporary location on App Server 1:

```bash
scp /tmp/ROOT.war tony@stapp01:/tmp/ROOT.war
```

The transfer completed successfully:

```text
ROOT.war  100% 4529  9.5MB/s  00:00
```

> **Why:** `scp` securely copies files over SSH. `/tmp/ROOT.war` is the source on the Jump Host, `tony@stapp01` identifies the destination login and server, and the second `/tmp/ROOT.war` is the temporary destination path. The `100%` transfer indicator confirms that the WAR file was copied completely.

### 📁 Step 6: Deploy ROOT.war to Tomcat

Reconnect to App Server 1:

```bash
ssh tony@stapp01
```

Move the WAR file into Tomcat's web application directory:

```bash
sudo mv /tmp/ROOT.war /var/lib/tomcat/webapps/ROOT.war
```

Restart Tomcat to load the deployed application:

```bash
sudo systemctl restart tomcat
```

> **Why:** `mv` moves the WAR into Tomcat's `webapps` directory, where Tomcat discovers web applications. The filename `ROOT.war` is special: it maps the application to the root context `/`. `sudo` is required because the Tomcat webapps directory is system-owned. `systemctl restart` stops and starts the service so the new application is loaded.

### ✅ Step 7: Verify the base URL

Exit back to the Jump Host:

```bash
exit
```

Request the application on the configured port:

```bash
curl http://stapp01:8087
```

The web page responded successfully:

```html
<!DOCTYPE html>
<html>
    <head>
        <title>SampleWebApp</title>
    </head>
    <body>
        <h2>Welcome to xFusionCorp Industries!</h2>
    </body>
</html>
```

> **Why:** `curl` sends an HTTP request from the Jump Host to App Server 1. The hostname `stapp01` identifies the server and `8087` identifies the configured Tomcat HTTP port. The returned HTML confirms that Tomcat is listening correctly and that `ROOT.war` is deployed at the base URL.

## Best Practices

- **Use the correct Tomcat configuration file.** On this package installation, `/etc/tomcat/server.xml` contains the connector configuration.
- **Preserve connector roles.** Change only the HTTP connector port required by the task; leave shutdown and AJP ports unchanged.
- **Use the ROOT context deliberately.** Naming the WAR `ROOT.war` makes the application available at `/`.
- **Deploy into Tomcat's webapps directory.** `/var/lib/tomcat/webapps/` is the application deployment directory created by the package.
- **Enable the service for future boots.** `systemctl enable --now tomcat` starts Tomcat now and configures automatic startup.
- **Test the exact required URL.** `curl http://stapp01:8087` validates the port and base context together.

### 📚 Official Documentation

- [Apache Tomcat 9 documentation](https://tomcat.apache.org/tomcat-9.0-doc/)
- [Apache Tomcat 9 introduction](https://tomcat.apache.org/tomcat-9.0-doc/introduction.html)
- [Tomcat web application deployment](https://tomcat.apache.org/tomcat-9.0-doc/deployer-howto.html)
- [Apache Tomcat security considerations](https://tomcat.apache.org/tomcat-9.0-doc/security-howto.html)
- [scp(1) OpenBSD manual page](https://man.openbsd.org/scp)
