Analyze the following snippets and write a one-liner describing the vulnerabilities.

## Dockerfiles Snippets

a.)
  ```dockerfile
  FROM python:latest
  COPY ./src /app
  WORKDIR /app
  CMD ["python", "server.py"]
  ```
 As no USER is not being changed within the Dockerfile the container will run as ROOT user.  
 
 b).
  ```dockerfile
  FROM python
  COPY . /app
  WORKDIR /app
  USER root
  CMD ["python", "server.py"]
  ```
 Running the container as a ROOT user. 
 
 c).
  ```dockerfile
  FROM python
  RUN groupadd -r appuser && \
  useradd --no-log-init -r -g appuser appuser
  COPY . /app
  WORKDIR /app
  USER appuser
  CMD ["python", "server.py"]
  ```
  Using `COPY . /app` should be avoided as all the root level files will get copied to the container which means files such as `.git`, `.dockerignore` or any other sensitive file will also be added to the container and that should be avoided. 
  
  d).
  ```dockerfile
  FROM python
 RUN groupadd -r appuser && \
  useradd --no-log-init -r -g appuser appuser
  COPY . /app
  WORKDIR /app
  CMD ["python", "server.py"]
  ```
  A new user is created but before starting the server the USER is not switched to the newly created user. 
  
  e).
  ```dockerfile
  FROM python
  RUN groupadd -r appuser && \
  useradd --no-log-init -r -g appuser appuser
  COPY . /app
  WORKDIR /app
  CMD ["python", "server.py"]
  USER appuser
  ```
  In this snippet the USER is changed to appuser but after running the server. As a result, the server will run as a ROOT user and if the server is compromised attacker can get ROOT privileges to the system. 
  
  f).
  ```dockerfile
  FROM nginx:stable
  RUN rm /etc/nginx/conf.d/*
  COPY nginx.conf /etc/nginx/nginx.conf
  RUN groupadd -r www-data && \
  useradd --no-log-init -r -g www-data www-data \
  touch /var/run/nginx.pid && \
  chown -R www-data:www-data /var/run/nginx.pid && \
  chown -R www-data:www-data /var/cache/nginx
  USER www-data
  ```
 
g).
  ```dockerfile
  FROM python
  RUN groupadd -r appuser && \
  useradd --no-log-init -r -g appuser appuser
  RUN echo "appuser: psswrd" | chpasswd
  COPY ./src /app
  WORKDIR /app
  USER appuser
  CMD ["python", "server.py"]
  ```
  New user is created but the credentials of the user are hardcoded in the file itself which must be avoided. 
 
h).
  ```dockerfile
  FROM python
  RUN groupadd -r appuser && \
  useradd --no-log-init -r -g appuser appuser
  RUN echo "2123244f297a57awdwdfwwe4a801fc3" > /run/secrets/secret.txt
  COPY ./src /app
  WORKDIR /app
  USER appuser
  CMD ["python", "server.py"]
  ```
  
  Sensitive secret file is created and it's content is leaking through Dockerfile. 
  
i). 
  ```dockerfile
  FROM python
  ENV user_pass="p4ssw0rd"
  RUN groupadd -r appuser && \
  useradd --no-log-init -r -g appuser appuser
  RUN echo "appuser:$user_pass" | chpasswd
  COPY ./src /app
  WORKDIR /app
  USER appuser
  CMD ["python", "server.py"]
  ```
  User password is leaking through the Dockerfile and also getting stored in environment variable. 
  
## Terrform Snippets

a).
  ```tf
  variable "db_password" {
  description = "password of user."
  type = string
  default = "mdbpassword"
  }
...
  ```
  
  Default password value is leaking.
  
b).
  ```tf
  resource "google_bigquery_dataset" "assetonfire" {
  dataset_id                  = "e_dataset"
  friendly_name               = "fire"
  description                 = "This is description"
  location                    = "EU"
  default_table_expiration_ms = 360000

  labels = {
    env = "default"
  }

  access {
    role          = "OWNER"
    special_group = "allAuthenticatedUsers"
  }
  }
  ```
  
  All authenticated users are given owner permission to the resource. 
  
c).
  ```tf
  resource "google_storage_bucket_iam_member" "norton" {
  bucket = google_storage_bucket.default.name
  role = "roles/storage.admin"
  member = "allUsers"
  }

  resource "google_storage_bucket_iam_member" "express" {
  bucket = google_storage_bucket.default.name
  role = "roles/storage.admin"
  members = ["user:john@express.com","allAuthenticatedUsers"]
  }
  ```
  All the users if authenticated or unauthenticated are given admin privileges to the google storage bucket. 
  
d).
  ```tf
  provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "longitude23" {
  ami           = "ami-005e54dde23324e" # sg-south-2
  instance_type = "t2.micro"

  tags = {
    Name = "test"
  }

  user_data = <<EOF
#!/bin/bash
apt-get install -y awscli
export AWS_ACCESS_KEY_ID=access_key_id
export AWS_SECRET_ACCESS_KEY=secret_access_key
EOF

  credit_specification {
    cpu_credits = "unlimited"
  }
}
  ```

AWS secret key and secret access key are leaking. 

e).
  ```tf
  provider "aws" {
  region = "us-east-1"
  }

  resource "aws_instance" "onemoreinstance" {
  ami           = "ami-005e54dee72cc1d00"
  instance_type = "t2.micro"
  tags = {
    Name = "test"
  }

  user_data_base64 = var.init_aws_cli

  credit_specification {
    cpu_credits = "unlimited"
  }
  }
  ```
  
f).
  ```tf
  resource "aws_iam_role_policy" "localone" {
  name = "apigateway-cloudwatch-logging"
  role = aws_iam_role.apigateway_cloudwatch_logging.id

  policy = <<EOF
  {
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["*"],
      "Resource": "*"
    }
  ]
  }
  EOF
  }
  ```
  The policy is allowing every action to every resources. Instead, everything should be treated as blacklist and allow actions as required. 
  
g).
```tf
provider "aws" {
  region = "us-east-1"
}

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.0"
    }
  }
}

resource "aws_network_acl" "local5678" {
  vpc_id = aws_vpc.main.id

  egress = [
    {
      protocol   = "tcp"
      rule_no    = 200
      action     = "allow"
      cidr_block = "10.3.0.0/18"
      from_port  = 443
      to_port    = 443
    }
  ]

  ingress = [
    {
      protocol   = "tcp"
      rule_no    = 100
      action     = "allow"
      cidr_block = "0.0.0.0/0"
      from_port   = 3389
      to_port     = 3389
    }
  ]

  tags = {
    Name = "main"
  }
}
```
Remote Desktop port (3389) is allowed to be accessed by anyone from the internet. 

h).
```tf
resource "azurerm_network_security_rule" "tester1667" {
     name                        = "tester1667one"
     priority                    = 100
     direction                   = "Inbound"
     access                      = "Allow"
     protocol                    = "TCP"
     source_port_range           = "*"
     destination_port_range      = "3389"
     source_address_prefix       = "*"
     destination_address_prefix  = "*"
     resource_group_name         = azurerm_resource_group.glints.name
     network_security_group_name = azurerm_network_security_group.glints.name
}
```
Remote Desktop port (3389) is allowed to be accessed by anyone from the internet. 

i).
```tf
resource "aws_s3_bucket" "mybugs" {
  bucket = "my-bucket"
  acl    = "public-read"

  tags = {
    Name        = "My bag"
    Environment = "Dev"
  }

  versioning {
    enabled = true
  }
}
```
S3 bucket is readable by anyone reagardless of authorization. 

j).
```tf
resource "google_sql_database_instance" "postdbql" {
  name             = "master-instance"
  database_version = "POSTGRES_11"
  region           = "us-central1"

  settings {

    tier = "db-f1-micro"
  }
}

resource "google_sql_database_instance" "pgdbql2" {
  name             = "postgres-instance"
  database_version = "POSTGRES_11"

  settings {
    tier = "db-f1-micro"

    ip_configuration {

      authorized_networks {
        name  = "pub-network"
        value = "0.0.0.0/0"
      }
    }
  }
}

resource "google_sql_database_instance" "pgdbql4" {
  name             = "master"
  database_version = "POSTGRES_11"
  region           = "us-central1"

  settings {
    # Second-generation instance tiers are based on the machine
    # type. See argument reference below.
    tier = "db-f1-micro"

    ip_configuration {
        ipv4_enabled = true
    }
  }
}

resource "google_sql_database_instance" "dbql5" {
  name             = "master-instance"
  database_version = "POSTGRES_11"
  region           = "us-central1"

  settings {

    tier = "db-f1-micro"

    ip_configuration {}
  }
}
```

k).
```tf
resource "google_compute_instance" "vmone" {
  name         = "test"
  machine_type = "e2-medium"
  zone         = "us-central1-a"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-9"
    }
  }

  network_interface {
    network = "default"
    access_config {
      // Ephemeral IP
    }
  }

  service_account {
    scopes = ["userinfo-email", "compute-ro", "storage-ro", "cloud-platform"]
  }
}
```

l).
```tf
resource "azurerm_app_service" "cyborg" {
  name                = "service"
  location            = azurerm_resource_group.glints.location
  resource_group_name = azurerm_resource_group.glints.name
  app_service_plan_id = azurerm_app_service_plan.glints.id

  site_config {
    dotnet_framework_version = "v4.0"
    scm_type                 = "LocalGit"
    min_tls_version = 1.1
  }
}
```
Minimum TLS version is lower than the recommended lowest TLS version (TLS 1.2)

m).
```tf
provider "aws" {
  region = "us-east-1"
}

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.0"
    }
  }
}

resource "aws_network_acl" "lookout" {
  vpc_id = aws_vpc.main.id

  egress = [
    {
      protocol   = "tcp"
      rule_no    = 200
      action     = "allow"
      cidr_block = "10.3.0.0/18"
      from_port  = 443
      to_port    = 443
    }
  ]

  ingress = [
    {
      protocol   = "tcp"
      rule_no    = 100
      action     = "allow"
      cidr_block = "0.0.0.0/0"
      from_port   = 22
      to_port     = 22
    }
  ]

  tags = {
    Name = "main"
  }
}
```

SSH port is exposed to internet and anyone can connect to it.
