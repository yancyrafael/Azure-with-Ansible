# Azure VM Deployment with Terraform + Ansible + GitHub Actions

Fully automated infrastructure provisioning and application deployment pipeline. A single `git push` creates a VM in Azure, installs Docker, and deploys a fullstack coffee shop application — no manual steps required.

## Architecture

```
  git push
     │
     ▼
  GitHub Actions
     │
     ├── Job 1: Terraform
     │   └── Creates VM, VNet, NSG, Public IP in Azure
     │
     └── Job 2: Ansible (waits for Job 1)
         └── Connects via SSH → Installs Docker → Deploys app
                                                      │
                                                      ▼
                                              ┌──────────────┐
                                              │  Azure VM    │
                                              │ ┌──────────┐ │
                                              │ │  Nginx   │ │  ← Port 8090
                                              │ │ Frontend │ │
                                              │ └──────────┘ │
                                              │ ┌──────────┐ │
                                              │ │ Flask API│ │  ← Port 5000
                                              │ └──────────┘ │
                                              │ ┌──────────┐ │
                                              │ │  MySQL   │ │  ← Port 3306
                                              │ └──────────┘ │
                                              └──────────────┘
```

## How It Works

### Job 1: Terraform (Create Infrastructure)

Terraform provisions the following Azure resources:

| Resource                 | Purpose                             |
|--------------------------|-------------------------------------|
| Resource Group           | Logical container for all resources |
| Virtual Network + Subnet | Private network for the VM          |
| Public IP (Standard SKU) | External access to the VM           |
| Network Security Group   | Firewall rules (SSH, HTTP, API, Web)|
| Network Interface        | Connects VM to the network          |
| Linux VM (Ubuntu 22.04)  | Server where the app runs           |

### Job 2: Ansible (Configure and Deploy)

After Terraform finishes, Ansible connects to the VM via SSH and runs the following tasks in order:

| Step                  | What it does                                                   |
|-----------------------|----------------------------------------------------------------|
| Update packages       | Runs apt update and upgrade on the VM                          |
| Install dependencies  | Installs ca-certificates, curl, gnupg, git                     |
| Add Docker GPG key    | Adds Docker's official GPG key for package verification        |
| Add Docker repository | Configures the Docker apt repository                           |
| Install Docker        | Installs docker-ce, docker-cli, containerd, and compose plugin |
| Start Docker          | Enables and starts the Docker service                          |
| Clone repository      | Clones the Coffee Shop app from GitHub                         |
| Deploy with Compose   | Runs docker compose up -d --build                              |
| Health check          | Polls the API until it responds with status 200                |
| Show status           | Displays running containers                                    |

## Project Structure

```
Azure-with-Ansible/
├── .github/
│   └── workflows/
│       └── deploy.yml          # GitHub Actions pipeline
├── terraform/
│   ├── main.tf                 # Azure resources (VM, VNet, NSG, IP)
│   ├── variables.tf            # Variable declarations
│   ├── terraform.tfvars        # Variable values
│   └── outputs.tf              # Exports VM public IP for Ansible
├── ansible/
│   └── playbook.yml            # Server configuration and app deployment
└── README.md
```

## Ports

| Port | Service   | Purpose                              |
|------|-----------|--------------------------------------|
| 22   | SSH       | Ansible connects to configure the VM |
| 80   | HTTP      | Reserved for future use              |
| 5000 | Flask API | Reservation management endpoints     |
| 8090 | Nginx     | Coffee shop frontend                 |

## API Endpoints

| Method | Route                    | Description          |
|--------|--------------------------|----------------------|
| GET    | `/api/health`            | Health check         |
| GET    | `/api/reservations`      | List all reservations|
| POST   | `/api/reservations`      | Create a reservation |
| DELETE | `/api/reservations/<id>` | Delete a reservation |

## Prerequisites

### Terraform Backend (one-time setup)

```bash
az group create --name rg-terraform-state --location eastus
az storage account create --name tfstateyancy --resource-group rg-terraform-state --sku Standard_LRS
az storage container create --name tfstate --account-name tfstateyancy
```

### GitHub Secrets

Configure these in **Settings → Secrets and variables → Actions**:

| Secret | Description |
|--------|-------------|
| `AZURE_AD_CLIENT_ID` | Azure Service Principal client ID |
| `AZURE_AD_CLIENT_SECRET` | Azure Service Principal secret |
| `AZURE_AD_TENANT_ID` | Azure tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription ID |

## Deploy

Push to main and the pipeline runs automatically:

```bash
git add .
git commit -m "Deploy"
git push
```

Or trigger manually from GitHub → Actions → **Run workflow**.

## Access the Application

After deployment, get the VM's public IP:

```bash
az vm show --name coffeeshop-vm --resource-group rg-ansible-vm --show-details --query publicIps -o tsv
```

Then open:
- Frontend: `http://<IP>:8090`
- API: `http://<IP>:5000/api/health`
- Reservations: `http://<IP>:5000/api/reservations`

## Cleanup

To destroy all Azure resources:

```bash
cd terraform
terraform destroy -var-file="terraform.tfvars"
```

## Tech Stack

| Tool           | Role                                                      |
|----------------|-----------------------------------------------------------|
| Terraform      | Infrastructure as Code — creates Azure resources          |
| Ansible        | Configuration as Code — installs software and deploys app |
| GitHub Actions | CI/CD — orchestrates the full pipeline                    |
| Docker Compose | Container orchestration on the VM                         |
| Azure          | Cloud provider                                            |

## Author

**Yency Christopher**
