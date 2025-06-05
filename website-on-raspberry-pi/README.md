# Self-Hosted Ghost Blog with Nginx Proxy Manager and Cloudflare DDNS

This repository contains a `docker-compose.yml` file to quickly deploy a self-hosted Ghost blogging platform, managed by Nginx Proxy Manager for reverse proxying and SSL, with Cloudflare DDNS for dynamic IP updates.

## Overview

This Docker Compose setup orchestrates the following services:

1.  **Nginx Proxy Manager (NPM):**
    * **Image:** `jlesage/nginx-proxy-manager:latest`
    * Provides an easy-to-use web interface for managing Nginx proxy hosts.
    * Handles SSL certificate generation and renewal via Let's Encrypt.
    * Exposes ports `8080` (HTTP), `4443` (HTTPS), and `8181` (Admin UI) on the host. **You will need to port forward external ports 80 and 443 on your router to these internal ports on your Docker host.**

2.  **Cloudflare DDNS:**
    * **Image:** `oznu/cloudflare-ddns:latest`
    * Automatically updates your Cloudflare DNS records (specifically an A record) with your home's dynamic public IP address. This ensures your domain name always points to your server.

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
* **Cloudflare API Token:** Required for the DDNS service.
* **Host Directories:** You need to create specific directories on your Docker host for persistent data storage (see "Volume Configuration" below).

## Configuration - IMPORTANT: Modifications Required!

Before deploying, you **MUST** edit the `docker-compose.yml` file to include your specific details.

### 1. Cloudflare DDNS Service (`cloudflare-ddns`):

* **`CF_API_TOKEN`**:
    * Replace `YOUR_CLOUDFLARE_API_TOKEN_HERE` with your actual Cloudflare API Token.
    * This token needs permissions: `Zone:Zone:Read` and `Zone:DNS:Edit` for the specific zone(s) you want to update.
* **`CF_ZONE_NAME`**:
    * Replace `yourdomain.com` with your actual domain name registered with Cloudflare (e.g., `example.com`).
* **`CF_SUBDOMAIN_NAME`**:
    * Set this to the A record you want to update.
        * `@` will update the root domain (e.g., `example.com`).
        * A specific subdomain (e.g., `blog`) will update `blog.example.com`.
* **`CF_PROXIED`**:
    * Set to `true` to proxy traffic through Cloudflare (orange cloud).
    * Set to `false` for DNS only (grey cloud).

### 2. Ghost Server Service (`ghost-server`):

* **`url`**:
    * Replace `https://test.campanale.com` with the full public URL where your blog will be accessible (e.g., `https://blog.yourdomain.com`). **This must match the domain you configure in Nginx Proxy Manager.**
* **`database__connection__password`**:
    * Change `SuperSecurePassword` to a strong, unique password. **This password MUST match the `MYSQL_ROOT_PASSWORD` in the `ghost-db` service.**

### 3. Ghost Database Service (`ghost-db`):

* **`MYSQL_ROOT_PASSWORD`**:
    * Change `SuperSecurePassword` to the same strong, unique password you set for `database__connection__password` in the `ghost-server` service.

### 4. Volume Configuration (Host Path Binds):

This setup uses host path binds for persistent storage. You **MUST ensure these directories exist on your Docker host machine** *before* starting the stack. If they don't exist, Docker might create them as files or with incorrect permissions, leading to issues.

* **Nginx Proxy Manager:**
    * `volumes: - /portainer/npm/config:/config`
    * Create the directory `/portainer/npm/config` on your host.
* **Ghost Server Content:**
    * `volumes: - /portainer/ghost/content:/var/lib/ghost/content`
    * Create the directory `/portainer/ghost/content` on your host.
* **Ghost Database Data:**
    * `volumes: - /portainer/ghost/mysql:/var/lib/mysql`
    * Create the directory `/portainer/ghost/mysql` on your host.

**Note:** You can change these host paths (e.g., `/portainer/npm/config`) to any location you prefer on your Docker host. Just ensure the path before the colon (`:`) is updated and the directory exists.

## Deployment

### Using Portainer:

1.  Log into your Portainer instance.
2.  Go to "Stacks."
3.  Click "+ Add stack."
4.  Give your stack a name (e.g., `ghost-blog-stack`).
5.  Copy the entire modified `docker-compose.yml` content into the "Web editor."
6.  Click "Deploy the stack."

### Using Docker Compose CLI:

1.  Save the modified configuration as `docker-compose.yml` in a directory on your Docker host.
2.  Navigate to that directory in your terminal.
3.  Run the command: `docker-compose up -d`
    * The `-d` flag runs the containers in detached mode (in the background).

## Post-Deployment Steps

1.  **Access Nginx Proxy Manager:**
    * Open your web browser and go to `http://<your-docker-host-ip>:8181`.
    * Default login for `jlesage/nginx-proxy-manager` is often `admin@example.com` / `changeme` or similar (check the image documentation if unsure). **Change the default credentials immediately.**

2.  **Verify Cloudflare DDNS:**
    * Check the logs of the `cloudflare-ddns` container in Portainer or via `docker logs cloudflare-ddns`.
    * You should see messages indicating successful IP checks and updates (if your IP has changed).
    * Verify in your Cloudflare dashboard that the A record specified by `CF_SUBDOMAIN_NAME` is pointing to your current public IP address.

3.  **Configure Proxy Host in Nginx Proxy Manager for Ghost:**
    * In the NPM admin interface, go to "Hosts" -> "Proxy Hosts."
    * Click "Add Proxy Host."
    * **Domain Names:** Enter the public domain for your blog (e.g., `blog.yourdomain.com` – this MUST match the `url` in your `ghost-server` environment variables).
    * **Scheme:** `http`
    * **Forward Hostname / IP:** `ghost-server` (this is the service name of your Ghost container)
    * **Forward Port:** `2368` (this is the internal port Ghost listens on)
    * **Enable "Block Common Exploits."**
    * Go to the **SSL** tab:
        * Select "Request a new SSL certificate."
        * Enable "Force SSL" and "HTTP/2 Support."
        * Agree to the Let's Encrypt ToS.
        * Click "Save."

4.  **Access Your Ghost Blog:**
    * Wait a few minutes for the SSL certificate to be issued and for NPM to apply the settings.
    * Navigate to your configured Ghost URL (e.g., `https://blog.yourdomain.com`).
    * You should see the Ghost setup page. Complete the Ghost installation by creating your admin account.

5.  **Port Forwarding on Your Router:**
    * Log into your internet router.
    * Set up port forwarding rules:
        * Forward external TCP port `80` to your Docker host's IP address on port `8080`.
        * Forward external TCP port `443` to your Docker host's IP address on port `4443`.
    * This allows external traffic to reach Nginx Proxy Manager.

## Troubleshooting

* **Container not starting:** Check logs using `docker logs <container_name>` or via Portainer. Common issues include incorrect environment variables, missing host directories for volumes, or port conflicts.
* **SSL Certificate Failed:**
    * Ensure your domain's DNS (A record updated by Cloudflare DDNS) is correctly pointing to your public IP.
    * Verify that port forwarding (external 80/443 to internal 8080/4443) is correctly configured on your router and that your ISP isn't blocking these ports.
    * Check NPM logs for details from Let's Encrypt.
* **Ghost "Too many redirects" or incorrect styling:** This is often due to an incorrect `url` environment variable in the `ghost-server` configuration. Ensure it matches your public HTTPS URL *exactly*.
* **Database connection issues:** Double-check that `database__connection__password` in `ghost-server` matches `MYSQL_ROOT_PASSWORD` in `ghost-db`. Ensure `database__connection__host` is set to `ghost-db`.

## Contributing

Feel free to fork this repository and suggest improvements by creating a pull request.

## License

This configuration is provided as-is. You are free to use and modify it.