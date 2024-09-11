# Practical Lab 6: Configuration Management with Puppet using Docker

## Objective
In this lab, you will learn how to use Puppet for configuration management in a containerized environment. You'll set up a Puppet master and a Puppet agent using Docker containers, and use Puppet to manage the configuration of a web server.

## Prerequisites
- Docker and Docker Compose installed on your local machine
- Basic understanding of Docker and containerization concepts
- Familiarity with command-line operations

## Estimated Time
1 hour

## Use Case
You're a DevOps engineer tasked with setting up a reproducible development environment that demonstrates Puppet's configuration management capabilities. You need to create a solution that can quickly deploy a Puppet master and agent, allowing for easy experimentation and testing of Puppet manifests.

## Steps

### 1. Set up the project structure

1. Create a new directory for your project:
   ```
   mkdir puppet-docker-lab
   cd puppet-docker-lab
   ```

2. Create the following file structure:
   ```
   puppet-docker-lab/
   ├── docker-compose.yml
   ├── puppetmaster/
   │   ├── Dockerfile
   │   └── manifests/
   │       └── site.pp
   └── puppetagent/
       └── Dockerfile
   ```

### 2. Create Docker Compose file

Create `docker-compose.yml`:

```yaml
version: '3'
services:
  puppetmaster:
    build: ./puppetmaster
    hostname: puppetmaster
    ports:
      - "8140:8140"
    volumes:
      - ./puppetmaster/manifests:/etc/puppetlabs/code/environments/production/manifests

  puppetagent:
    build: ./puppetagent
    hostname: puppetagent
    depends_on:
      - puppetmaster
    environment:
      - PUPPET_SERVER=puppetmaster
```

### 3. Set up Puppet Master

1. Create `puppetmaster/Dockerfile`:

```Dockerfile
FROM puppet/puppetserver

COPY manifests/site.pp /etc/puppetlabs/code/environments/production/manifests/

CMD ["puppetserver", "foreground"]
```

2. Create `puppetmaster/manifests/site.pp`:

```puppet
node default {
  package { 'nginx':
    ensure => installed,
  }
  
  service { 'nginx':
    ensure  => running,
    enable  => true,
    require => Package['nginx'],
  }
  
  file { '/usr/share/nginx/html/index.html':
    ensure  => file,
    content => '<html><body><h1>Hello from Puppet-managed container!</h1></body></html>',
    require => Package['nginx'],
  }
}
```

### 4. Set up Puppet Agent

Create `puppetagent/Dockerfile`:

```Dockerfile
FROM puppet/puppet-agent

RUN apt-get update && apt-get install -y nginx

CMD ["puppet", "agent", "--verbose", "--no-daemonize", "--summarize"]
```

### 5. Build and run the containers

1. Build and start the containers:
   ```
   docker-compose up --build -d
   ```

2. Check if the containers are running:
   ```
   docker-compose ps
   ```

### 6. Configure Puppet

1. Enter the Puppet master container:
   ```
   docker-compose exec puppetmaster /bin/bash
   ```

2. Inside the container, generate a certificate for the agent:
   ```
   puppetserver ca generate --certname puppetagent
   ```

3. Exit the Puppet master container:
   ```
   exit
   ```

4. Enter the Puppet agent container:
   ```
   docker-compose exec puppetagent /bin/bash
   ```

5. Inside the container, run Puppet agent:
   ```
   puppet agent --test
   ```

   This should apply the configuration defined in the manifest.

6. Exit the Puppet agent container:
   ```
   exit
   ```

### 7. Verify the Configuration

1. Enter the Puppet agent container again:
   ```
   docker-compose exec puppetagent /bin/bash
   ```

2. Check if Nginx is installed and running:
   ```
   service nginx status
   ```

3. Check the content of the index.html file:
   ```
   cat /usr/share/nginx/html/index.html
   ```

4. Exit the container:
   ```
   exit
   ```

### 8. Make a Configuration Change

1. Modify the `puppetmaster/manifests/site.pp` file to change the content of index.html:

   ```puppet
   file { '/usr/share/nginx/html/index.html':
     ensure  => file,
     content => '<html><body><h1>Hello from updated Puppet-managed container!</h1></body></html>',
     require => Package['nginx'],
   }
   ```

2. Enter the Puppet agent container and run Puppet again:
   ```
   docker-compose exec puppetagent /bin/bash
   puppet agent --test
   ```

3. Verify that the change has been applied by checking the index.html file again.

### 9. Clean up

When you're done experimenting, stop and remove the containers:

```
docker-compose down
```

## Conclusion
In this lab, you've learned how to use Puppet for configuration management in a containerized environment. You've set up a Puppet master and agent using Docker, created a basic manifest to manage a web server configuration, and seen how changes in the manifest are propagated to the agent.

## Additional Exercises
1. Add another service to the Docker Compose file (e.g., a database container) and manage its configuration with Puppet.
2. Use Puppet to manage users and groups in the agent container.
3. Implement Puppet Hiera to separate data from code in your manifests.
4. Set up multiple Puppet agent containers and use node definitions to apply different configurations to different containers.

Remember to explore the Puppet documentation for more advanced features and best practices in configuration management.
