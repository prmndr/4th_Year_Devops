# Practical Lab 1: Setting up WordPress with Docker Compose and Terraform

## Objective
In this lab, you will learn how to set up a WordPress website using two different provisioning methods: Docker Compose and Terraform. This exercise will demonstrate different approaches to infrastructure as code and containerization.

## Prerequisites
- Docker Desktop installed on your Windows PC
- Terraform installed on your Windows PC
- Basic understanding of YAML and HCL syntax

## Estimated Time
1 hour

## Use Case
You're a DevOps engineer tasked with setting up a development environment for a WordPress website. You need to demonstrate two different methods of provisioning this environment to your team: one using Docker Compose and another using Terraform with Docker provider.

## Part 1: Using Docker Compose

### 1. Create a new directory for your Docker Compose project
```
mkdir wordpress-docker
cd wordpress-docker
```

### 2. Create a docker-compose.yml file
Create a file named `docker-compose.yml` with the following content:

```yaml
version: '3'

services:
  db:
    image: mysql:5.7
    volumes:
      - db_data:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: somewordpress
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress

  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    ports:
      - "8000:80"
    restart: always
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress

volumes:
  db_data: {}
```

### 3. Start the containers
```
docker-compose up -d
```

### 4. Access WordPress
Open a web browser and navigate to `http://localhost:8000`. You should see the WordPress installation page.

### 5. Stop and remove the containers
When you're done, stop and remove the containers:
```
docker-compose down --volumes
```

## Part 2: Using Terraform with Docker Provider

### 1. Create a new directory for your Terraform project
```
mkdir wordpress-terraform
cd wordpress-terraform
```

### 2. Create a main.tf file
Create a file named `main.tf` with the following content:

```hcl
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0.0"
    }
  }
}

provider "docker" {}

resource "docker_network" "wordpress_net" {
  name = "wordpress_net"
}

resource "docker_volume" "mysql_data" {
  name = "mysql_data"
}

resource "docker_container" "mysql" {
  name  = "wordpress_db"
  image = "mysql:5.7"
  restart = "always"
  network_mode = docker_network.wordpress_net.name

  volumes {
    volume_name    = docker_volume.mysql_data.name
    container_path = "/var/lib/mysql"
  }

  env = [
    "MYSQL_ROOT_PASSWORD=somewordpress",
    "MYSQL_DATABASE=wordpress",
    "MYSQL_USER=wordpress",
    "MYSQL_PASSWORD=wordpress"
  ]
}

resource "docker_container" "wordpress" {
  name  = "wordpress"
  image = "wordpress:latest"
  restart = "always"
  network_mode = docker_network.wordpress_net.name

  ports {
    internal = 80
    external = 8000
  }

  env = [
    "WORDPRESS_DB_HOST=${docker_container.mysql.name}:3306",
    "WORDPRESS_DB_USER=wordpress",
    "WORDPRESS_DB_PASSWORD=wordpress",
    "WORDPRESS_DB_NAME=wordpress"
  ]

  depends_on = [docker_container.mysql]
}
```

### 3. Initialize Terraform
```
terraform init
```

### 4. Apply the Terraform configuration
```
terraform apply
```

### 5. Access WordPress
Open a web browser and navigate to `http://localhost:8000`. You should see the WordPress installation page.

### 6. Destroy the infrastructure
When you're done, destroy the infrastructure:
```
terraform destroy
```

## Comparison and Discussion

After completing both parts of this lab, consider the following questions:

1. What are the main differences between using Docker Compose and Terraform for this task?
2. Which method do you find easier to understand and why?
3. How might each method scale for larger, more complex infrastructures?
4. What are the advantages and disadvantages of each approach?

## Conclusion
In this lab, you've learned how to set up a WordPress development environment using two different Infrastructure as Code (IaC) tools: Docker Compose and Terraform. Both methods achieve the same result but use different approaches and syntaxes.

## Additional Exercises
1. Modify both the Docker Compose and Terraform configurations to use environment variables for sensitive data like passwords.
2. Add a phpMyAdmin service to both setups for easier database management.
3. Research how to use Terraform to manage Docker Compose files, combining both approaches.

Remember to explore the documentation for Docker Compose and Terraform to learn about more advanced features and best practices.
