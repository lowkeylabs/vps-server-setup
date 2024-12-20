*i'm setting up a VPS running ubuntu 22.04.5.  The server is set up with docker and nginx. DNS will send different subdomains to the server (e.g., 409.cmsc-vcu.com or 427.cmsc-vcu.com).  I'd like NGINX to route these requests to an appropriate docker container that will provide the necessary services.  I need code to install nginx, then set up the external nginx on the server, then a template docker container to receive the appropriate subdomain requests, and a template container to catch all unmapped requests that lists the subdomain name requested.  I'd like nginx to use a map command to route requests to the appropriate docker container. further, I'd like each docker container to created with their own custom docker-compose file stored in separate folders for each scalability.*


To achieve your goal, we will follow these steps:

1. **Install and configure NGINX** on your VPS.
2. **Set up a dynamic NGINX configuration** using the `map` command to route subdomain requests to the appropriate Docker containers.
3. **Create template Docker containers** for mapped and unmapped subdomains.
4. **Use `docker-compose` for container management**, with each service in its own folder for scalability.

Here’s the detailed implementation:

---

### Step 1: Install NGINX

Run the following commands to install NGINX and start the service:

```bash
sudo apt update
sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```

---

### Step 2: Configure NGINX for Subdomain Routing

Create a new configuration file for NGINX routing logic.

#### NGINX Configuration (`/etc/nginx/conf.d/subdomain-router.conf`)

```nginx
map $host $upstream {
    default                "catchall:80";  # Default to the catchall container.
    408.cmsc-vcu.com       "cmsc408:80";   # Map specific subdomains to containers.
    427.cmsc-vcu.com       "cmsc427:80";
}

server {
    listen 80;
    server_name _;

    resolver 8.8.8.8 8.8.4.4 valid=300s;  # Define a DNS resolver with a cache timeout.
    resolver_timeout 5s;                  # Set the timeout for DNS resolution.

    location / {
        proxy_pass http://cmsc408:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Reload NGINX to apply the configuration:

```bash
sudo nginx -t
sudo systemctl reload nginx
```
---


To achieve this, we need to set up a directory structure under `/home/docker` and ensure that the permissions allow any user in the `docker` group to access and manage the containers. Here’s how to set it up:

---

### Step 3: Create a Shared Directory for Docker Containers

1. Create the `/home/docker` directory and set the correct permissions:

   ```bash
   sudo mkdir -p /home/docker
   sudo chown -R :docker /home/docker
   sudo chmod -R 775 /home/docker
   ```

   - `chown -R :docker /home/docker`: Changes the group ownership to the `docker` group.
   - `chmod -R 775 /home/docker`: Ensures that members of the `docker` group can read, write, and execute files within the directory.

2. Verify that the `docker` group exists (Docker typically creates it upon installation):

   ```bash
   grep docker /etc/group
   ```

   If the group does not exist, create it:

   ```bash
   sudo groupadd docker
   ```

3. Add all required users to the `docker` group:

   ```bash
   sudo usermod -aG docker <username>
   ```

   Replace `<username>` with the name of each user you want to grant access. Users need to log out and back in for this to take effect.

---

### Step 2: Update the Deployment Script

Modify the `deploy-container.sh` script to store all `docker-compose` configurations under `/home/docker`.

```bash
#!/bin/bash

SUBDOMAIN=$1
if [ -z "$SUBDOMAIN" ]; then
  echo "Usage: $0 <subdomain>"
  exit 1
fi

FOLDER="/home/docker/${SUBDOMAIN}"
mkdir -p $FOLDER/html

# Ensure correct permissions
chown -R :docker $FOLDER
chmod -R 775 $FOLDER

cat <<EOF > $FOLDER/docker-compose.yml
version: '3.9'
services:
  $SUBDOMAIN:
    image: nginx:alpine
    container_name: $SUBDOMAIN
    ports:
      - "80"
    volumes:
      - ./html:/usr/share/nginx/html
EOF

echo "<h1>Welcome to $SUBDOMAIN</h1>" > $FOLDER/html/index.html

# Ensure correct permissions on files
chown -R :docker $FOLDER
chmod -R 775 $FOLDER

# Deploy the container
docker-compose -f $FOLDER/docker-compose.yml up -d
```

Save this script and make it executable:

```bash
chmod +x deploy-container.sh
```

---

### Step 3: Update the Catch-All Container

Ensure the catch-all container is also located under `/home/docker/catchall`:

1. Create the directory:

   ```bash
   mkdir -p /home/docker/catchall/html
   sudo chown -R :docker /home/docker/catchall
   sudo chmod -R 775 /home/docker/catchall
   ```

2. Use the updated `docker-compose.yml` and HTML files from the earlier instructions in `/home/docker/catchall`.

---

### Step 4: Automate Permissions for New Deployments

To ensure all new deployments inherit the correct permissions:

1. Set the `docker` group as the default group for `/home/docker`:

   ```bash
   sudo chgrp -R docker /home/docker
   sudo chmod g+s /home/docker
   ```

   The `chmod g+s` command ensures that new files and directories created under `/home/docker` inherit the `docker` group.

2. Test the setup by deploying a new subdomain:

   ```bash
   ./deploy-container.sh cmsc409
   ./deploy-container.sh cmsc427
   ```

---

### Step 5: Verification

1. Confirm that files are created under `/home/docker` and have the correct permissions:

   ```bash
   ls -l /home/docker
   ```

   You should see `docker` as the group for all files and directories.

2. Test that all users in the `docker` group can manage the containers:

   ```bash
   docker-compose -f /home/docker/cmsc409/docker-compose.yml up -d
   ```

Now, all Docker containers managed by `docker-compose` are stored in `/home/docker`, and permissions allow all users in the `docker` group to manage them.


---


#### Template for Unmapped Subdomains

1. Create a folder for the catchall container:

   ```bash
   mkdir -p ~/docker/catchall
   cd ~/docker/catchall
   ```

2. Create a `docker-compose.yml` file:

   ```yaml
   version: '3.9'
   services:
     catchall:
       image: nginx:alpine
       container_name: catchall
       ports:
         - "80"
       volumes:
         - ./html:/usr/share/nginx/html
   ```

3. Add a script to generate the HTML dynamically:

   ```bash
   mkdir html
   cat <<EOF > html/index.html.template
   <!DOCTYPE html>
   <html>
   <head><title>Unmapped Subdomain</title></head>
   <body>
   <h1>Subdomain Not Mapped</h1>
   <p>You requested: {{host}}</p>
   </body>
   </html>
   EOF

   cat <<EOF > html/update-index.sh
   #!/bin/bash
   HOSTNAME=\$(echo \$HOST)
   sed "s/{{host}}/\$HOSTNAME/" index.html.template > index.html
   EOF

   chmod +x html/update-index.sh
   ```

4. Add the script to the `docker-compose.yml`:

   ```yaml
   services:
     catchall:
       ...
       entrypoint: ["/bin/sh", "-c", "./html/update-index.sh && nginx -g 'daemon off;'"]
   ```

5. Start the container:

   ```bash
   docker-compose up -d
   ```

---

### Step 4: Automate Container Deployment

Create a helper script (`deploy-container.sh`) to streamline deployment:

```bash
#!/bin/bash

SUBDOMAIN=$1
if [ -z "$SUBDOMAIN" ]; then
  echo "Usage: $0 <subdomain>"
  exit 1
fi

FOLDER=~/docker/${SUBDOMAIN}
mkdir -p $FOLDER/html
cat <<EOF > $FOLDER/docker-compose.yml
version: '3.9'
services:
  $SUBDOMAIN:
    image: nginx:alpine
    container_name: $SUBDOMAIN
    ports:
      - "80"
    volumes:
      - ./html:/usr/share/nginx/html
EOF

echo "<h1>Welcome to $SUBDOMAIN</h1>" > $FOLDER/html/index.html
docker-compose -f $FOLDER/docker-compose.yml up -d
```

Run this script for each subdomain:

```bash
bash deploy-container.sh cmsc409
bash deploy-container.sh cmsc427
```

---

Now, NGINX will dynamically route subdomain requests to the appropriate containers, and the setup allows for easy scalability by managing each container with its own `docker-compose` configuration.