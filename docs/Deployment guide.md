# Guide: Deploying New Services via Ansible on Home Server

This documentation describes standardized practice for adding Dockerized services managed with Ansible.

## 1. Preparation and Standards

- **Default user:** Use the default `ansible_user` unless the container specifically requires a dedicated system user.
- **Secrets/passwords:** Store all secrets in Ansible Vault (`vars/secrets.yml`). Never hard-code sensitive data in playbooks or templates.
- **Docker networks:**
  - All services join `home_net` (a bridge Docker network).
  - If the service depends on a database or other components, create a dedicated `<service_name>_net` bridge network for internal traffic.
- **Configuration files:** Store configuration files/templates in `/home/{ansible_user}/`.
- **Persistent data:** 
  - Store in `/data/<service_name>/`. All `/data/` is backed up.
  - Split it into subfolders if required. Like `/data/<service_name>/db`, `/data/<service_name>/storage/`, etc.
- **Resource limits:** No special resource limits are set for Docker containers.
- **Logging & monitoring:** Not configured for now.

## 2. Step-by-Step Deployment Guide

### Define Variables
1. Add any needed variables for your new service in the relevant `vars/` file (e.g., version, port, password).
2. Add secrets (like passwords) to `vars/secrets.yml` and encrypt with Ansible Vault.

### Prepare Directories and Users

1. **Configuration:** Ensure `/home/{ansible_user}/{service_name}` exists for configs if required.
2. **Data:** Make `/data/{service_name}` directory for persistence (included in backups) if required.
3. **User:** Create a new system user ONLY IF REQUIRED by the container or for file permissions. Otherwise, default user suffices.

### Set up Docker Networks

1. Ensure `home_net` exists.
2. If the service has internal dependencies (like a DB), create a `<service_name>_net` network and connect dependent containers to it.

### Configuration Files
  
If the service is configured by files:
  
1. Place all service templates or static config files under `./configs/`, then deploy them to `/home/{ansible_user}/{service_name}` via Ansible `copy:` or `template:` tasks.

These configuration files are stored in the repository in the `./configs/` directory and cannot contain secrets. Environment variables are preferred for configuration where applicable.

### Set up Docker Containers

1. Use the Ansible `docker_container` module.
2. Always connect the service container to `home_net`. If needed, also connect to `<service_name>_net` to access internal components (like db, custom services, cache, etc).
3. Map ports via `published_ports` as needed to access service.
4. Mount volumes:
   - `/home/{ansible_user}/{service_name}` for configs if required.
   - `/data/{service_name}` for persistent data.
5. Use the `env:` for environment variables, referencing secrets from Ansible Vault or to provide additional configuration.
  - If sensitive data is included in environment variables, references must correctly point to values within the Ansible Vault.

*Example snippet:*
```yaml
- name: Run <service> container
  docker_container:
    name: <service>
    image: <image>:{{ <service_version_variable>_version }}
    networks:
      - name: home_net
        aliases: <service>
      # If needed:
      # - name: <service>_net
      #   aliases: <service>
    published_ports:
      - "{{ <service_port> }}:<container_port>"
    volumes:
      - "/home/{{ ansible_user}}/<service_name>:/config"
      - "/data/<service_name>:/data"
    env:
      EXAMPLE_VAR: "{{ example_var }}"
      # EXAMPLE_INTERNAL_CONNECTION: "db.<service>_net:12345"
    restart_policy: unless-stopped
```

## 3. Full Example: Adding a New Service

Suppose you want to deploy `appX`, which needs no dedicated user, stores persistent data, and has a custom config.

1. **Create needed vars (in `vars/versions.yml` and `vars/ports.yml`)**  
   ```yaml
   appx_version: '1.2.3' # container_registry.net/appx
   appx_port: 8080
   ```

2. **Add secrets to `vars/secrets.yml`** (encrypted):  
   ```yaml
   appx_password: supersecret
   ```

3. **Create data directory:**
   ```yaml
   - name: Create data directory for appX
     file:
       path: "/data/appx"
       state: directory
   ```

4. **Optional: Prepare configs `configs/appx.conf` and deploy them:**
   ```yaml
   - name: Create config directory for appX
     file:
       path: "/home/{{ ansible_user }}/appx"
       state: directory


   - name: Deploy appX config
     template:
       src: ../configs/appx.conf # example config
       dest: "/home/{{ ansible_user }}/appx/appx.conf"
   ```

5. **Ensure the bridge network(s):**
   ```yaml
   - name: Create home_net
     docker_network:
       name: home_net
   ```

6. **Run the container:**
   ```yaml
   - name: Run appX container
     docker_container:
       name: appx
       image: appx/appx:{{ appx_version }}
       networks:
         - name: home_net
           aliases: appx
       published_ports:
         - "{{ appx_port }}:80"
       volumes:
         - "/home/{{ ansible_user }}/appx:/config"
         - "/data/appx:/data"
       env:
         APPX_PASSWORD: "{{ appx_password }}"
       restart_policy: unless-stopped
   ```

## 4. Post-Deployment Steps

After successfully deploying the new service, follow these steps to complete the setup:

### Configure Subdomain in Pi-hole

1. Access your Pi-hole admin interface.
2. Navigate to the **DNS Settings** section.
3. Add a new DNS record for the service, specifying:
   - **Subdomain**: For example, `appx.{{ domain }}`.
   - **IP Address**: Set this to the IP address of the server where the service is hosted.

### Configure Reverse Proxy for This Domain (Nginx)

1. Add record for new service in `./config/nginx.conf`
  *Example:*
  ```
  server {
      server_name <service_name>.{{ domain }};

      location / {
          proxy_pass      http://{{ pihost }}:{{ service_port }};
      }
  }
  ```
2. Reload nginx using `./playbooks/nginx.yml`.

### Place New Service in the Homepage

1. Update your homepage configuration files (`./configs/homepage/services`).
2. Add a new entry or link that leads to your newly deployed service:
   ```yml
    - Service:
        icon: http://<service_name>.{{ domain }}/favicon.ico
        href: http://<service_name>.{{ domain }}
        description: service description
   ```
3. Publish changes via `./playbooks/homepage.yml`

## 5. Additional Notes

- **For DB-dependent services:**  
  Use a `<service>_net` for the database and the main app, connect both containers.

- **Manual validation/removal:**  
  After deployment, check service health via UI, logs, or API.
  
- **Manual removal:**
  To remove, stop and remove containers plus their config/data directories manually.
