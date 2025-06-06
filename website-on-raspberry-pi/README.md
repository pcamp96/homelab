# Self-Hosted Ghost Blog with Nginx Proxy Manager and ddclient

This repository contains a `docker-compose.yml` file to quickly deploy a self-hosted Ghost blogging platform, managed by Nginx Proxy Manager for reverse proxying and SSL, with `ddclient` for dynamic IP updates.

## Overview

This Docker Compose setup orchestrates the following services:

1.  **Nginx Proxy Manager (NPM):**
    * **Image:** `jlesage/nginx-proxy-manager:latest`
    * Provides an easy-to-use web interface for managing Nginx proxy hosts.
    * Handles SSL certificate generation and renewal via Let's Encrypt.
    * Exposes ports `8080` (HTTP), `4443` (HTTPS), and `8181` (Admin UI) on the host. **You will need to port forward external ports 80 and 443 on your router to these internal ports on your Docker host.**

2.  **ddclient:**
    * **Image:** `ddclient/ddclient:latest`
    * A highly configurable Dynamic DNS (DDNS) client.
    * Automatically updates your Cloudflare DNS records with your home's dynamic public IP address.
    * Configuration is handled via a `ddclient.conf` file.

3.  **Ghost Server:**
    * **Image:** `ghost:5`
    * The Ghost blogging platform application.
    * Configured to use an external MySQL database.
    * Exposes port `2368` internally, which Nginx Proxy Manager will proxy to.

4.  **Ghost Database (MySQL):**
    * **Image:** `mysql:8`
    * A dedicated MySQL database server for your Ghost installation.

All services are connected via a custom Docker network (`proxy-network`), allowing them to communicate with each other by their service names.

## Prerequisites

* **Docker and Docker Compose:** Must be installed on your host machine (e.g., Raspberry Pi, Linux server).
* **Domain Name:** You need to own a domain name.
* **Cloudflare Account:** Your domain should be managed through Cloudflare for DNS.
* **Cloudflare API Token:** Required for the `ddclient` service.
* **Host Directories & Files:** You need to create specific directories and configuration files on your Docker host for persistent data storage (see "Volume Configuration" below).

## Configuration - IMPORTANT: Modifications Required!

Before deploying, you **MUST** create and edit the required configuration files.

### 1. ddclient Service (`ddclient.conf`):

This service is configured via a file, not environment variables.

1.  Create the directory `/portainer/ddclient` on your host machine.
2.  Inside that directory, create a new file named `ddclient.conf`.
3.  Paste the following content into `ddclient.conf` and edit it with your details:

```ini
# /portainer/ddclient/ddclient.conf

# Check for IP change every 5 minutes (300 seconds)
daemon=300

# General configuration
ssl=yes
use=web
protocol=cloudflare

# --- Your Cloudflare Details ---
# zone = your actual domain name
# password = your Cloudflare API Token
zone=yourdomain.com
password=YOUR_CLOUDFLARE_API_TOKEN

# --- Your Record(s) to Update ---
# List the A record(s) you want to update, one per line.
# Use '@' for the root domain (yourdomain.com).
# Use a subdomain name for a subdomain (e.g., blog for blog.yourdomain.com).
@
blog
www