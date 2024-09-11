# Practical Lab 2: Simple Container Management with Ansible and Docker

## Objective
In this lab, you will learn how to use Ansible to manage and configure a simple Docker container running on your local machine. You'll use Ubuntu on Windows Subsystem for Linux (WSL) as your Ansible control node to manage containers on your Windows host.

## Prerequisites
- Docker Desktop installed on your Windows PC
- Ubuntu (WSL) installed on your Windows PC
- Basic understanding of YAML syntax and command-line operations

## Estimated Time
1 hour

## Use Case
You're a DevOps engineer tasked with creating a simple, reproducible development environment for your team's web application. You need to set up a solution that can quickly deploy and configure a web server container. Ansible will be used to automate this process, allowing for easy updates and maintenance.

## Steps

### 1. Set up the Ansible control node (Ubuntu WSL)

1. Open your Ubuntu (WSL) terminal.

2. Update the package list and install Ansible:
   ```
   sudo apt update
   sudo apt install -y ansible
   ```

3. Install the Docker SDK for Python (required for Ansible to interact with Docker):
   ```
   sudo apt install -y python3-pip
   pip3 install docker
   ```

### 2. Create the Ansible project structure

1. Create a new directory for your Ansible project:
   ```
   mkdir ~/ansible-docker-lab
   cd ~/ansible-docker-lab
   ```

2. Create the following directory structure:
   ```
   mkdir -p roles/web-server/tasks
   touch inventory.ini ansible.cfg playbook.yml
   ```

### 3. Configure Ansible

1. Edit `ansible.cfg`:
   ```
   [defaults]
   inventory = inventory.ini
   host_key_checking = False
   ```

2. Edit `inventory.ini`:
   ```
   [local]
   localhost ansible_connection=local
   ```

### 4. Create the Ansible role

1. Edit `roles/web-server/tasks/main.yml`:
   ```yaml
   ---
   - name: Ensure web server container is running
     docker_container:
       name: web-server
       image: nginx:alpine
       state: started
       restart_policy: always
       ports:
         - "8080:80"

   - name: Copy custom index.html to the container
     copy:
       content: |
         <html>
           <body>
             <h1>Hello from Ansible-managed Docker container!</h1>
           </body>
         </html>
       dest: /tmp/index.html

   - name: Update nginx container with custom index.html
     docker_container:
       name: web-server
       state: started
       restart: yes
       volumes:
         - /tmp/index.html:/usr/share/nginx/html/index.html
   ```

### 5. Create the Ansible playbook

Edit `playbook.yml`:
```yaml
---
- hosts: localhost
  roles:
    - web-server
```

### 6. Run the Ansible playbook

Execute the playbook:
```
ansible-playbook playbook.yml
```

### 7. Verify the setup

1. Check if the container is running:
   ```
   docker ps
   ```

2. Access the web server:
   Open a web browser and navigate to `http://localhost:8080`. You should see the custom "Hello from Ansible-managed Docker container!" message.

### 8. Modify the configuration

1. Edit the `roles/web-server/tasks/main.yml` file to change the web server port from 8080 to 8081.

2. Re-run the playbook:
   ```
   ansible-playbook playbook.yml
   ```

3. Verify that the web server is now accessible at `http://localhost:8081`.

### 9. Clean up

Create a new playbook called `cleanup.yml`:

```yaml
---
- hosts: localhost
  tasks:
    - name: Remove web server container
      docker_container:
        name: web-server
        state: absent
```

Run the cleanup playbook:
```
ansible-playbook cleanup.yml
```

## Conclusion
In this lab, you've learned how to use Ansible to manage a simple Docker container on your local machine. This approach allows for easy replication and modification of development environments, demonstrating the power of combining configuration management tools with containerization.

## Additional Exercises
1. Modify the role to use a different web server image (e.g., Apache).
2. Create Ansible variables for the container name and port, and use them in your role.
3. Extend the role to create a custom Dockerfile and build a custom image.
4. Implement Ansible handlers to restart the container only when the configuration changes.

Remember to explore the Ansible documentation, especially the Docker-related modules, for more advanced features and best practices.
