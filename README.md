## Secure Edge Proxy Deployment (CYS 411)
Project Overview
This project demonstrates a secure, containerized web architecture using Traefik v3 as an edge proxy and NGINX as a backend service. The implementation focuses on SSL/TLS termination, automated traffic routing, and secure credential management.

Architecture
Edge Proxy: Traefik (Entry point for all HTTPS traffic).

Backend: NGINX (Static website server).

Security: SSL certificates handled via local mounting with traefik/certs.

Technical Walkthrough
## Prerequisites
Docker & Docker Compose installed.

OpenSSL for certificate generation.
## File system
## Directory Tree

├── docker-compose.yml             # Docker services orchestration

├── .gitignore                      # Git ignore rules (Security)

├── README.md                        # Project documentation

├── traefik/

│   ├── traefik.yml                  # Traefik main configuration

│   ├── routes.yml                   # Explicit routing rules

│   └── certs/

│       ├── cert.pem                 # SSL certificate (Ignored)

│       └── key.pem                  # Private key (Ignored)

├── nginx/

│   └── default.conf                    # NGINX server configuration

└── website/

    └── index.html              # Static website content

Generate Local Certificates:

Bash
``openssl req -x509 -newkey rsa:4096 -keyout traefik/certs/key.pem -out traefik/certs/cert.pem -sha256 -days 365 -nodes``

3. Deployment

Run the following command to launch the stack in detached mode:

Bash
``docker-compose up -d``

4. Verification

Traefik Dashboard: Accessible at http://localhost:8080/dashboard/.

Secure Site: Accessible at ``https://localhost.``
