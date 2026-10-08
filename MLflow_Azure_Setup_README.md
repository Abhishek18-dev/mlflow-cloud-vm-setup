# MLflow on Azure with S3-Compatible Azure Blob Storage

This guide reproduces a typical AWS MLflow setup on **Azure for Students** while keeping the MLflow application code mostly unchanged.

## Architecture

```text
Local Windows / VS Code
        |
        | MLFLOW_TRACKING_URI
        v
Azure Ubuntu VM
        |
        | MLflow Server :5000
        v
Flexify.IO (S3-compatible API)
        |
        v
Azure Blob Storage
        |
        v
Blob Container: mlflowtracking1
```

### AWS → Azure Mapping

| AWS Setup | Azure Setup |
|---|---|
| EC2 Ubuntu | Azure Ubuntu VM |
| S3 Bucket | Azure Blob Container |
| S3 API | Flexify.IO S3-compatible API |
| AWS Security Group | Azure NSG / Inbound Port Rule |
| EC2 Public DNS | Azure VM Public IP |
| AWS credentials | Flexify S3 credentials |
| `s3://mlflowtracking1` | Same S3 path through Flexify |
| MLflow Server | MLflow Server |

> **Important:** S3 is not created inside Azure. Azure provides Blob Storage. Flexify provides an S3-compatible API layer so existing S3-based applications can continue using the S3 API.

---

# 1. Prerequisites

You need:

- Azure for Students account
- Azure Portal access
- SSH client on Windows
- VS Code
- Python
- Your existing MLflow training project
- Flexify.IO S3-compatible API

Do **not** put Azure keys, Flexify keys, or passwords in GitHub.

---

# 2. Create Azure Ubuntu VM

### LOCATION: Azure Portal

Go to:

```text
Azure Portal
→ Virtual Machines
→ Create
→ Azure virtual machine
```

Suggested basic configuration:

```text
VM Name: mlflow-vm
Image: Ubuntu Server 22.04 LTS
Authentication: SSH public key
```

Choose a small VM suitable for learning.

After creation, note:

```text
VM Public IP
SSH username
```

Example:

```text
Public IP: 20.50.100.25
Username: azureuser
```

---

# 3. Open MLflow Port 5000

### LOCATION: Azure Portal

Go to:

```text
Virtual Machine
→ Networking
→ Inbound port rules
→ Create port rule
```

Add:

```text
Protocol: TCP
Destination Port: 5000
Action: Allow
```

Port 5000 is used by the MLflow tracking server.

> For a learning setup this can be temporarily exposed, but production systems should use proper authentication, HTTPS, and restricted network access.

---

# 4. Connect to Azure Ubuntu

### LOCATION: Local Windows PowerShell

Use the SSH private key downloaded/generated during VM creation.

```powershell
ssh -i "C:\path\to\your-key.pem" azureuser@YOUR_AZURE_VM_IP
```

Example:

```powershell
ssh -i "C:\Users\Abhishek\Downloads\mlflow-vm.pem" azureuser@20.50.100.25
```

If successful, you should see something similar to:

```text
azureuser@mlflow-vm:~$
```

You are now inside the Azure Ubuntu VM.

---

# 5. Install Python and MLflow

### LOCATION: Azure Ubuntu VM

Update packages:

```bash
sudo apt update
```

Install pip:

```bash
sudo apt install python3-pip -y
```

Install virtual environment support:

```bash
sudo apt install python3-venv -y
```

Create MLflow directory:

```bash
mkdir ~/mlflow
cd ~/mlflow
```

Create virtual environment:

```bash
python3 -m venv venv
```

Activate it:

```bash
source venv/bin/activate
```

Upgrade pip:

```bash
pip install --upgrade pip
```

Install MLflow and boto3:

```bash
pip install mlflow boto3
```

Check MLflow:

```bash
mlflow --version
```

Save the version because MLflow commands can differ between versions.

---

# 6. Create Azure Storage Account

### LOCATION: Azure Portal

Go to:

```text
Storage accounts
→ Create
```

Create a storage account, for example:

```text
Storage account name:
mlflowstorage12345
```

Storage account names must be globally unique and follow Azure naming rules.

---

# 7. Create Blob Container

### LOCATION: Azure Portal

Open:

```text
Storage Account
→ Data storage
→ Containers
→ Create
```

Create:

```text
Container name:
mlflowtracking1
```

Keep the container private.

Conceptually:

```text
AWS:
S3 Bucket
└── mlflowtracking1

Azure:
Storage Account
└── Blob Container
    └── mlflowtracking1
```

---

# 8. Get Azure Storage Credentials

### LOCATION: Azure Portal

Open:

```text
Storage Account
→ Security + networking
→ Access keys
```

You will see:

```text
Storage account name
Key1
Key2
```

Keep these private.

You will use the storage account credentials while configuring Flexify.

---

# 9. Deploy Flexify.IO

### LOCATION: Azure Portal / Marketplace

Search for:

```text
S3 API for Azure Blob Storage (Flexify.IO)
```

If available, use the appropriate **Community Edition / free-use option** rather than assuming that every "Free Trial" Marketplace plan is permanently free.

Deploy Flexify according to the Marketplace wizard.

After deployment, note the Flexify management endpoint/IP.

---

# 10. Configure Flexify with Azure Blob Storage

### LOCATION: Flexify Web UI

Open the Flexify management interface in your browser.

Configure an Azure Blob Storage backend using:

```text
Azure Storage Account Name
Azure Storage Account Key
```

Connect it to the Azure Storage Account created earlier.

The resulting architecture becomes:

```text
Flexify
   |
   v
Azure Storage Account
   |
   v
mlflowtracking1
```

---

# 11. Configure the S3-Compatible Endpoint

### LOCATION: Flexify Web UI

Create/configure an S3 endpoint for the Azure storage.

Flexify should provide S3-compatible connection information such as:

```text
S3 Endpoint
Access Key
Secret Key
```

Do not invent these values. Use the values shown by your Flexify deployment.

Conceptually:

```text
S3 application
      |
      | S3 API
      v
Flexify S3 Endpoint
      |
      | Azure Blob API
      v
Azure Blob Storage
```

---

# 12. Test S3 → Flexify → Azure Blob

### LOCATION: Azure Ubuntu VM

Activate the environment if necessary:

```bash
cd ~/mlflow
source venv/bin/activate
```

Set the Flexify credentials:

```bash
export AWS_ACCESS_KEY_ID="YOUR_FLEXIFY_ACCESS_KEY"
export AWS_SECRET_ACCESS_KEY="YOUR_FLEXIFY_SECRET_KEY"
```

Set the S3-compatible endpoint:

```bash
export MLFLOW_S3_ENDPOINT_URL="YOUR_FLEXIFY_ENDPOINT"
```

Now test with Python:

```bash
python
```

Run:

```python
import boto3

s3 = boto3.client(
    "s3",
    endpoint_url="YOUR_FLEXIFY_ENDPOINT"
)

print(s3.list_buckets())
```

Expected result should show the storage exposed through Flexify.

Exit:

```python
exit()
```

If this test fails, do not continue to MLflow. Fix the S3/Flexify connection first.

---

# 13. Start MLflow Server

### LOCATION: Azure Ubuntu VM

Make sure the environment variables are available:

```bash
export AWS_ACCESS_KEY_ID="YOUR_FLEXIFY_ACCESS_KEY"
export AWS_SECRET_ACCESS_KEY="YOUR_FLEXIFY_SECRET_KEY"
export MLFLOW_S3_ENDPOINT_URL="YOUR_FLEXIFY_ENDPOINT"
```

Check:

```bash
echo $MLFLOW_S3_ENDPOINT_URL
```

Then start MLflow.

The teacher's basic style is:

```bash
mlflow server \
    --host 0.0.0.0 \
    --port 5000 \
    --default-artifact-root s3://mlflowtracking1
```

> **Important:** Check your installed MLflow version first:
>
> ```bash
> mlflow --version
> ```
>
> Newer MLflow versions also support server-side artifact proxying with `--artifacts-destination`. If the `--default-artifact-root` setup gives an error or behaves differently, use the command appropriate for your installed MLflow version rather than randomly changing the configuration.

Keep the MLflow server terminal running.

---

# 14. Open MLflow UI

### LOCATION: Local Browser

Open:

```text
http://YOUR_AZURE_VM_IP:5000
```

Example:

```text
http://20.50.100.25:5000
```

If the MLflow UI opens, the connection is:

```text
Local Browser
      |
      v
Azure VM :5000
      |
      v
MLflow Server
```

---

# 15. Connect Local VS Code to Azure MLflow

### LOCATION: Local VS Code Git Bash

Do this only after the Azure MLflow server is running.

```bash
export MLFLOW_TRACKING_URI="http://YOUR_AZURE_VM_IP:5000"
```

Check it:

```bash
echo $MLFLOW_TRACKING_URI
```

Expected:

```text
http://YOUR_AZURE_VM_IP:5000
```

### PowerShell alternative

If using PowerShell instead of Git Bash:

```powershell
$env:MLFLOW_TRACKING_URI="http://YOUR_AZURE_VM_IP:5000"
```

Do not run both commands. They are alternatives.

---

# 16. Python Training Code

### LOCATION: Local VS Code

You do **not** need to convert your MLflow code to the Azure Blob SDK.

Your existing code can continue using:

```python
with mlflow.start_run():

    mlflow.log_param("alpha", alpha)
    mlflow.log_param("l1_ratio", l1_ratio)

    mlflow.log_metric("rmse", rmse)
    mlflow.log_metric("mae", mae)
    mlflow.log_metric("r2", r2)

    mlflow.sklearn.log_model(
        lr,
        "model",
        registered_model_name="ElasticNetModel"
    )
```

Because the tracking URI is already supplied through the environment variable.

### Remove duplicate tracking URI configuration

Do not keep both:

```python
os.environ["MLFLOW_TRACKING_URI"] = "..."
```

and:

```python
mlflow.set_tracking_uri("...")
```

if you are using the terminal environment-variable method.

The clean approach is:

```text
Local terminal:
MLFLOW_TRACKING_URI
        ↓
Python
        ↓
MLflow Server
```

---

# 17. Run the Training Script

### LOCATION: Local VS Code terminal

Example:

```bash
python train.py
```

If your script takes parameters:

```bash
python train.py 0.5 0.5
```

Your flow becomes:

```text
Local train.py
      |
      | MLflow Tracking URI
      v
Azure MLflow Server
      |
      | S3 API
      v
Flexify
      |
      | Azure Blob API
      v
Azure Blob Container
```

---

# 18. What Goes Where?

| Task | Location |
|---|---|
| Create Azure VM | Azure Portal |
| Create Storage Account | Azure Portal |
| Create Blob Container | Azure Portal |
| Open port 5000 | Azure Portal |
| Deploy Flexify | Azure Portal / Marketplace |
| Configure Azure storage in Flexify | Flexify UI |
| Configure S3 endpoint | Flexify UI |
| SSH into VM | Local PowerShell |
| `sudo apt ...` | Azure Ubuntu |
| `pip install ...` | Azure Ubuntu |
| S3 connection test | Azure Ubuntu |
| Start MLflow | Azure Ubuntu |
| Open MLflow UI | Local Browser |
| `export MLFLOW_TRACKING_URI=...` | Local VS Code terminal |
| `python train.py` | Local VS Code terminal |
| Training code | Local VS Code |

---

# 19. Important Concepts

### `MLFLOW_TRACKING_URI`

This tells your local MLflow client:

```text
"Which MLflow server should I talk to?"
```

Example:

```bash
export MLFLOW_TRACKING_URI="http://20.50.100.25:5000"
```

It is **NOT** the S3 endpoint.

### S3 Endpoint

This tells boto3/MLflow:

```text
"Where should S3 API requests go?"
```

In this setup:

```text
S3 API
   ↓
Flexify
   ↓
Azure Blob
```

### `s3://mlflowtracking1`

This identifies the artifact location from the application's S3 perspective.

The actual physical storage is Azure Blob Storage.

---

# 20. Cost / Safety Checklist

Before using Azure resources:

- Use Azure for Students where eligible.
- Prefer free-use/Community Flexify options when available.
- Check the Marketplace pricing page before deployment.
- Do not assume "Free Trial" means permanently free.
- Stop/deallocate the Azure VM when you are finished.
- Keep the Blob container private.
- Never commit credentials to GitHub.
- Do not put access keys directly into Python source code.
- Check Azure Cost Management regularly.

---

# 21. Quick Start Checklist

```text
[ ] Azure for Students active
[ ] Ubuntu VM created
[ ] VM public IP noted
[ ] SSH working
[ ] Port 5000 opened
[ ] Storage Account created
[ ] Blob Container mlflowtracking1 created
[ ] Flexify deployed
[ ] Flexify connected to Azure Blob
[ ] Flexify S3 endpoint created
[ ] S3 connection tested
[ ] MLflow installed
[ ] MLflow server running
[ ] MLflow UI opens on :5000
[ ] Local MLFLOW_TRACKING_URI set
[ ] Training script executed
[ ] MLflow run visible
[ ] Model artifact visible in Azure Blob
```

## Final Architecture

```text
                  LOCAL
             VS Code / Python
                    |
                    | MLFLOW_TRACKING_URI
                    v
             Azure Ubuntu VM
                    |
                    | MLflow :5000
                    v
               Flexify.IO
                    |
                    | S3 API
                    v
            Azure Blob Storage
                    |
                    v
            mlflowtracking1
```

**Key idea:** MLflow and your application continue speaking the **S3 API**. Flexify handles the translation to Azure Blob Storage, so the main MLflow training code does not need to be rewritten for Azure.
