# Home Services Setup

## Overview

This setup provides a secure and flexible reverse proxy solution using **Traefik**, integrated with **Tailscale** for network access, and managed partly through **Portainer**. It runs within a Docker Compose stack on an Ubuntu 24.04 LTS virtual machine hosted on Proxmox and secured with Tailscale.

The primary goals are:

1.  **Secure Access:** Expose web services only over HTTPS using Let's Encrypt certificates.
2.  **Tailscale Integration:** Make services accessible from anywhere via the Tailscale network without exposing ports directly to the public internet.
3.  **Centralized Routing:** Use Traefik to route traffic based on hostnames (e.g., `service.yourdomain.com`) to various backend services.
4.  **Automatic Certificates:** Leverage Traefik's Let's Encrypt integration with Cloudflare's DNS challenge for automatic certificate acquisition and renewal.
5.  **Manageability:** Use Portainer to manage Docker containers and Traefik's file provider for routing to non-Docker services.

## Architecture

The setup involves several layers:

1.  **Proxmox Host:** The base hypervisor.
2.  **Ubuntu 24.04 LTS VM:** Hosts the Docker environment.
3.  **Docker & Docker Compose:** Runs the containerized services.
4.  **Core Docker Stack (explained below)**
5.  **Backend Services:** These can be:
    *   Other Docker containers on the same host (managed by Portainer or other Compose files).
    *   Web services running on other machines within the local network.

## Core Stack Components

*   **Tailscale:**
    *   Connects to your Tailnet using an Auth Key.
    *   Provides the network interface (`tailscale0`) and IP address used by Traefik.
    *   Exposes ports 80 and 443 to the Tailscale network, receiving traffic destined for Traefik.
*   **Traefik:**
    *   Uses `network_mode: service:tailscale` to share the Tailscale container's network stack.
    *   Listens on ports 80 (for HTTP->HTTPS redirect) and 443 (for HTTPS) on the Tailscale IP.
    *   Manages Let's Encrypt certificates using Cloudflare DNS challenge (configured via environment variables).
    *   Routes traffic based on `Host` rules defined in Docker labels or dynamic configuration files.
    *   Its own dashboard is exposed via `traefik.yourdomain.com` (defined by labels on the Traefik service itself).
*   **Portainer:**
    *   Provides a web UI for Docker management.
    *   Exposed via `portainer.yourdomain.com` (defined by labels on the Portainer service).
    *   Connects to the `traefik-net` Docker network.

## Configuration Files

*   `docker-compose.yml`: Defines the core Tailscale, Traefik, and Portainer services, volumes, networks, and labels.
*   `.env`: Stores sensitive information like Cloudflare API Token, Tailscale Auth Key, domain name, and email address. **(DO NOT COMMIT TO GIT)**
*   `traefik-data/traefik.yml`: Traefik's static configuration (entrypoints, certificate resolvers, providers).
*   `traefik-data/dynamic_conf/config.yml`: Traefik's dynamic configuration for file-based routing (used for non-Docker services). Traefik watches this file for changes.
*   `letsencrypt/acme.json`: Stores the obtained Let's Encrypt certificates. **(Backup recommended, restrict permissions)**

## Prerequisites

*   Proxmox host with an Ubuntu 24.04 LTS VM.
*   Docker and Docker Compose installed on the Ubuntu VM.
*   A domain name managed through Cloudflare.
*   A Cloudflare API Token with DNS edit permissions for your domain zone.
*   A Tailscale account and an Auth Key.

## Setup Summary

1.  Clone or create the directory structure (`traefik-data`, `dynamic_conf`, `letsencrypt`).
2.  Create the `traefik.yml`, `dynamic_conf/config.yml`, and `docker-compose.yml` files as per the configuration.
3.  Create the `.env` file and populate it with your credentials and domain info.
4.  Create the external Docker network: `docker network create traefik-net`.
5.  Set permissions for `acme.json`: `chmod 600 letsencrypt/acme.json`.
6.  Start the stack: `docker compose up -d`.

## Accessing Services

1.  **DNS:** Ensure that the DNS records (CNAME, or A and/or AAAA) for all desired hostnames (`traefik.yourdomain.com`, `portainer.yourdomain.com`, etc.) point to the **Tailscale IP address of the `tailscale` container**. Make sure you are not using Cloudflare proxying.
2.  **Tailscale Connection:** Connect the device you are browsing from to your Tailscale network.
3.  **Access:** Navigate to `https://service.yourdomain.com` in your browser.

## Adding New Services to Traefik

There are two main ways to add services:

### Method 1: File Provider (for Non-Docker Services or Services on Other Hosts)

1.  **Identify Backend:** Note the IP address and port of the service (e.g., `http://192.168.2.31:3000` for Homepage). Determine if it uses HTTP or HTTPS.
2.  **Edit Dynamic Config:** Open `traefik-data/dynamic_conf/config.yml`.
3.  **Add Router:** Under `http.routers:`, add a new router section:
    ```yaml
      routers:
        proxmox-rtr:
          rule: "Host(`prox.soehlert.com`)"
          entryPoints:
            - websecure
          service: proxmox-svc
        homepage-rtr:
          rule: "Host(`homepage.soehlert.com`)"
          entryPoints:
            - websecure
          service: homepage-svc
    ```
4.   **Add Service:** Under `http.services:`, add a new service section:
     ```yaml
      services:
        proxmox-svc:
          loadBalancer:
            servers:
              - url: "https://192.168.2.21:8006"
            passHostHeader: true
            serversTransport: proxmox-transport
            homepage-svc:
          loadbalancer:
            servers:
              - url: "http://192.168.2.31:3000"
     ```
5.   **Add Transport:** Under `http.serversTransports`, add a new transport section (OPTIONAL - YOU MAY NOT NEED THIS):
     ```yaml
     serversTransports:
       proxmox-transport:
         insecureSkipVerify: true
     ```
### Method 2: Portainer
1.   **Add Stack:** Use the web editor to add a docker compose file like this:
     ```yaml
     services:
       whoami:
         image: traefik/whoami:latest
         container_name: whoami-app
         networks:
           - traefik-net
         restart: unless-stopped
         labels:
           # --- Traefik Labels ---
           - "traefik.enable=true"

           # Router definition
           - "traefik.http.routers.whoami-rtr.rule=Host(`whoami.soehlert.com`)"
           - "traefik.http.routers.whoami-rtr.entrypoints=websecure"
           - "traefik.http.routers.whoami-rtr.service=whoami-svc "

           # Service definition (points to the container's intern al port)
           - "traefik.http.services.whoami-svc.loadbalancer.server.port=80"
     networks:
       traefik-net:
         external: true
     ```