# Task Management App

This application consists of a _frontend built with React_ and a _backend built with Flask_. The backend uses a _MariaDB_ instance for storing data. It allows users to _manage tasks_, _create new tasks_, and _remove existing tasks_.

This project features a complete **DevOps pipeline** with **Infrastructure as Code (Terraform)** and **automated CI/CD (GitHub Actions)** for deployment to AWS.

![app-flow-diagram](app-flow-diagram.png)

## Features

- 🎯 **Task Management**: Full-featured task CRUD operations
- 🐳 **Dockerized**: Complete containerization with Docker Compose, including a DB healthcheck to avoid startup race conditions
- ☁️ **AWS Deployment**: Automated deployment to AWS EC2
- 🏗️ **Infrastructure as Code**: Terraform configuration for AWS resources
- 🚀 **CI/CD Pipeline**: GitHub Actions workflow for automated builds and deployments
- 📦 **Container Registry**: AWS ECR for Docker image management

## Prerequisites

### Local Development
- Node.js & npm - for frontend
- Python 3 - for backend
- Docker & Docker Compose - for containerized deployment (recommended; no need to install MariaDB/Node/Python locally if using Docker)

### AWS Deployment
- AWS Account with appropriate permissions
- Terraform >= 1.0
- AWS CLI configured
- GitHub repository with the following secrets:
  - `AWS_ACCESS_KEY_ID`
  - `AWS_SECRET_ACCESS_KEY`
  - `EC2_SSH_KEY`
  - `DB_PASSWORD`
  - `DB_ROOT_PASSWORD`
  - `DB_NAME`
- GitHub variables:
  - `PROJECT_NAME`
  - `AWS_REGION`

## Installation and Usage

### Local Development (Docker — recommended)

1. Create three `.env` files (not committed to git — see `.gitignore`):

   **`.env`** (project root):
   ```env
   MARIADB_ROOT_PASSWORD=your_root_password
   MARIADB_DATABASE=taskdb
   MARIADB_USER=taskuser
   MARIADB_PASSWORD=your_password
   ```

   **`backend/.env`**:
   ```env
   DB_USERNAME=taskuser
   DB_PASSWORD=your_password
   DB_HOST=db
   DB_NAME=taskdb
   ```

   **`frontend/.env`**:
   ```env
   REACT_APP_BACKEND_URL=http://localhost:5000
   ```

2. Build and start all services:
   ```sh
   docker-compose up -d --build
   ```

3. Visit `http://localhost:3000` to use the app.


### Manual Local Development (without Docker)

#### Frontend
- Configure environment variables in `.env`
- Install dependencies:
  ```sh
  npm install
  ```
- Launch on port 3000:
  ```sh
  npm start
  ```

#### Backend
- Configure environment variables in `.env`
- Install dependencies:
  ```sh
  pip install flask flask_sqlalchemy pymysql python-dotenv flask-cors
  ```
- Launch on port 5000:
  ```sh
  python main.py
  ```

#### Database
- Install MariaDB
- Create a database and user matching your `.env` values
- Run `backend/init.db/init.sql` to create the required tables

## Docker Deployment

### Local Docker Compose
```sh
docker-compose up -d --build
```

### Production Docker Compose (on AWS EC2)
```sh
docker-compose -f docker-compose.prod.yml up -d
```

## Infrastructure Setup (Terraform)

The `terraform/` directory contains complete Infrastructure as Code for AWS deployment:

### What Gets Provisioned
- **EC2 Instance**: t3.micro Ubuntu server with Docker pre-installed
- **ECR Repositories**: Container registries for frontend and backend images
- **Security Groups**: Rules for SSH (22), HTTP (80), and application ports (3000, 5000)
- **IAM Roles**: EC2 instance profile with ECR pull permissions, plus a dedicated IAM user for GitHub Actions with scoped ECR/EC2 permissions
- **SSH Key Pair**: Auto-generated for secure EC2 access

### Deploy Infrastructure

1. Navigate to the terraform directory:
   ```sh
   cd terraform
   ```
2. Initialize Terraform:
   ```sh
   terraform init
   ```
3. Review the planned changes:
   ```sh
   terraform plan
   ```
4. Apply the infrastructure:
   ```sh
   terraform apply
   ```
5. Retrieve outputs:
   ```sh
   terraform output -raw private_key_pem > ../devops-ssh-key.pem
   chmod 400 ../devops-ssh-key.pem
   terraform output server_public_ip
   terraform output ecr_frontend_url
   terraform output ecr_backend_url
   terraform output -raw github_access_key_id
   terraform output -raw github_secret_access_key
   ```

### Terraform Files
- `provider.tf` - AWS provider configuration
- `variables.tf` - Input variables for customization
- `ec2.tf` - EC2 instance and SSH key configuration
- `ecr.tf` - ECR repositories for Docker images
- `security.tf` - Security groups and network rules
- `iam.tf` - IAM roles and policies for EC2 and GitHub Actions
- `outputs.tf` - Output values (IP, SSH key, ECR URLs, IAM credentials)
- `user_data.sh` - EC2 initialization script (installs Docker, AWS CLI)

## CI/CD Pipeline (GitHub Actions)

The `.github/workflows/deploy.yml` file implements a two-stage CI/CD pipeline:

### Workflow Trigger
- Runs automatically on push to `main`

### Pipeline Stages

**1. Build and Push**
- Checks out code
- Configures AWS credentials
- Logs into Amazon ECR
- Builds Docker images for frontend and backend
- Pushes images to ECR

**2. Deploy**
- Retrieves the EC2 instance's public IP
- Copies `docker-compose.prod.yml` to EC2 via SCP
- SSHes into EC2 and:
  - Verifies Docker is installed
  - Logs into ECR
  - Generates the production `.env` file from GitHub secrets
  - Pulls the latest images
  - Deploys via Docker Compose

### Setting Up CI/CD

1. Fork/clone this repository
2. Add required secrets under Settings → Secrets and variables → Actions
3. Add required variables (`PROJECT_NAME`, `AWS_REGION`)
4. Push to `main` to trigger deployment

## Architecture

```
┌─────────────────┐      ┌──────────────────┐
│  GitHub Actions │─────▶│   AWS ECR        │
│   (CI/CD)       │      │  (Image Registry)│
└─────────────────┘      └──────────────────┘
                                  │
                                  ▼
                         ┌──────────────────┐
                         │   AWS EC2        │
                         │  (Application)   │
                         │                  │
                         │  ┌────────────┐  │
                         │  │  Frontend  │  │
                         │  │  (React)   │  │
                         │  └────────────┘  │
                         │  ┌────────────┐  │
                         │  │  Backend   │  │
                         │  │  (Flask)   │  │
                         │  └────────────┘  │
                         │  ┌────────────┐  │
                         │  │  MariaDB   │  │
                         │  └────────────┘  │
                         └──────────────────┘
```

## Project Structure

```
.
├── .github/
│   └── workflows/
│       └── deploy.yml          # GitHub Actions CI/CD pipeline
├── backend/
│   ├── Dockerfile              # Backend container image
│   ├── main.py                 # Flask application
│   └── init.db/
│       └── init.sql            # DB schema (tasks table)
├── frontend/
│   ├── Dockerfile               # Frontend container image
│   ├── package.json
│   └── src/                    # React application
├── terraform/
│   ├── ec2.tf                  # EC2 instance configuration
│   ├── ecr.tf                  # ECR repositories
│   ├── iam.tf                  # IAM roles and policies
│   ├── security.tf             # Security groups
│   ├── provider.tf             # AWS provider setup
│   ├── variables.tf            # Input variables
│   ├── outputs.tf               # Output values
│   └── user_data.sh            # EC2 initialization script
├── docker-compose.yml           # Local development compose
├── docker-compose.prod.yml      # Production compose
└── README.md
```

## Environment Variables

### Frontend
- `REACT_APP_BACKEND_URL` - Backend API URL

### Backend
- `DB_HOST` - Database host (`db` for local Docker Compose)
- `DB_USERNAME` - Database user
- `DB_PASSWORD` - Database password
- `DB_NAME` - Database name


## License

This project is open source and available under the MIT License.