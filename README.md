# cst8922-task4 -- Azure RI & Savings Plan Optimization Function 🚀

This project automates the analysis and optimization of Reserved Instances (RI) and Savings Plans (SP) in a large Azure environment. It generates a monthly report flagging underutilized RIs and SPs to help reduce cloud costs. Built with Azure Functions (Python) and deployed using GitHub Actions.

---

## 📊 Features

- 🔍 Tracks utilization of all Reserved Instances and Savings Plans in your tenant
- 📉 Flags any usage < 70% and recommends actions
- 📁 Outputs unified CSV report for FinOps team
- ⚙️ Automatically runs on a schedule (1st of every month)
- 🔁 CI/CD pipeline using GitHub Actions

---

## 🧰 Tech Stack

- Azure Functions (Python)
- Azure AD App Registration (Service Principal)
- Microsoft Capacity & Consumption REST APIs
- GitHub Actions for CI/CD
- Optional: Logic App or SendGrid for notifications

---
## 📦 Directory Structure

azure-ri-optimizer/
├── .github/
│ └── workflows/
│ └── deploy-functionapp.yml # CI/CD pipeline
├── CheckRIUtilization/ # Function folder
│ ├── init.py # Main trigger
│ ├── function.json # Timer schedule
│ ├── ri_optimizer.py # RI + SP logic
├── requirements.txt
├── host.json
├── .funcignore
├── .gitignore
├── README.md

---

## 🛠️ Setup Instructions

## 🧾 Prerequisites

- ✅ Azure Subscription (e.g. Azure for Students)
- ✅ Python 3.10+
- ✅ Azure CLI + Functions Core Tools
- ✅ Visual Studio Code
- ✅ GitHub repository
- ✅ Cost & Reservation APIs enabled on Azure

### 2. Clone the Repo

```bash
git clone https://github.com/<your-username>/azure-ri-optimizer.git
cd azure-ri-optimizer
```
### 3. Create a Service Principal (Azure AD App)

```bash
az ad sp create-for-rbac --name "RITrackerApp" \
  --role Reader \
  --scopes /subscriptions/<YOUR_SUBSCRIPTION_ID> \
  --sdk-auth
```

📌 Save the JSON output for GitHub Actions:

```bash
{
  "clientId": "...",
  "clientSecret": "...",
  "subscriptionId": "...",
  "tenantId": "...",
  ...
}
```
Save the following:

`appId` → `CLIENT_ID`

`password` → `CLIENT_SECRET`

`tenant → TENANT_ID`
You’ll use these in the Azure Function Configuration.
---
## 🏗️ Setup Azure Function (Local Dev)
**1. Initialize the Function App**

```bash
func init ri-tracker --python
cd ri-tracker
func new --name CheckRIUtilization --template "Timer trigger"
```
**2. Edit CheckRIUtilization/__init__.py**

```bash
import logging
import azure.functions as func
from .ri_optimizer import generate_ri_recommendations

def main(mytimer: func.TimerRequest) -> None:
    logging.info('RI Optimizer Function Started')
    generate_ri_recommendations()
```
**3. Add ri_optimizer.py**

```bash
import os
import requests
import pandas as pd
from datetime import datetime

TENANT_ID = os.getenv("TENANT_ID")
CLIENT_ID = os.getenv("CLIENT_ID")
CLIENT_SECRET = os.getenv("CLIENT_SECRET")

def get_token():
    url = f"https://login.microsoftonline.com/{TENANT_ID}/oauth2/token"
    data = {
        "grant_type": "client_credentials",
        "resource": "https://management.azure.com/",
        "client_id": CLIENT_ID,
        "client_secret": CLIENT_SECRET
    }
    response = requests.post(url, data=data)
    return response.json()["access_token"]

def get_reservations(token):
    url = f"https://management.azure.com/providers/Microsoft.Capacity/reservationOrders?api-version=2022-03-01"
    headers = {"Authorization": f"Bearer {token}"}
    return requests.get(url, headers=headers).json()

def get_savings_plans(token):
    url = f"https://management.azure.com/providers/Microsoft.Consumption/savingsPlans?api-version=2022-10-01"
    headers = {"Authorization": f"Bearer {token}"}
    return requests.get(url, headers=headers).json()

def get_savings_plans_utilization(token, sp_id):
    url = f"https://management.azure.com{sp_id}/usages?api-version=2022-10-01"
    headers = {"Authorization": f"Bearer {token}"}
    return requests.get(url, headers=headers).json()

def generate_ri_recommendations():
    token = get_token()
    report_data = []

    # Reserved Instances
    ri_data = get_reservations(token)
    for order in ri_data.get("value", []):
        usage_url = f"https://management.azure.com{order['id']}/usages?api-version=2022-03-01"
        usage = requests.get(usage_url, headers={"Authorization": f"Bearer {token}"}).json()
        for item in usage.get("value", []):
            utilization = item.get("utilizationPercentage", 100)
            if utilization < 70:
                report_data.append({
                    "type": "Reserved Instance",
                    "name": item["name"],
                    "utilization": utilization,
                    "scope": item["properties"].get("scopeId", "unknown"),
                    "recommendation": "Consider SKU change, reassignment, or exchange"
                })

    # Savings Plans
    sp_data = get_savings_plans(token)
    for sp in sp_data.get("value", []):
        usage_data = get_savings_plans_utilization(token, sp["id"])
        for usage_item in usage_data.get("value", []):
            utilization = usage_item.get("utilizationPercentage", 100)
            if utilization < 70:
                report_data.append({
                    "type": "Savings Plan",
                    "name": sp["name"],
                    "utilization": utilization,
                    "scope": usage_item["properties"].get("scope", "unknown"),
                    "recommendation": "Investigate workloads; consider changes"
                })

    # Output
    df = pd.DataFrame(report_data)
    filename = f"/tmp/ri_sp_report_{datetime.now().strftime('%Y%m%d')}.csv"
    df.to_csv(filename, index=False)
    print(f"Saved: {filename}")
```
**4. function.json**

```bash
{
  "scriptFile": "__init__.py",
  "bindings": [
    {
      "name": "mytimer",
      "type": "timerTrigger",
      "direction": "in",
      "schedule": "0 0 1 * * *"
    }
  ]
}
```
---
## 📦 Dependencies
**requirements.txt**

```bash
azure-functions
pandas
requests
```
---
## 🚀 GitHub Actions Deployment
**Step 1: Add Secrets to GitHub**
Name	                      Description
`AZURE_CREDENTIALS`	        Output from `az ad sp create-for-rbac --sdk-auth`
`FUNCTIONAPP_NAME`	        Your deployed Azure Function name
`AZURE_REGION`	            Your Azure region (e.g. canadacentral)
`RESOURCE_GROUP`	          Your Azure RG name

**Step 2: Add Workflow File**
`.github/workflows/deploy-functionapp.yml`

```bash
name: Deploy Azure Function

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    - name: Setup Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.10'

    - name: Install Azure Functions Core Tools
      run: npm install -g azure-functions-core-tools@4 --unsafe-perm true

    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Deploy Function App
      run: func azure functionapp publish ${{ secrets.FUNCTIONAPP_NAME }} --python
```
---
## ✅ Test It Locally

```bash
func start
```
✅ CSV report generated at `/tmp/ri_sp_report_YYYYMMDD.csv`
---
## ✉️ (Optional) Add Logic App Email Workflow
1. Trigger: When new blob (CSV) is uploaded

2. Action: SendGrid or Office 365 Outlook email

3. Include CSV as attachment

---
### Step-by-Step Guide: Sending Blob Files via Email Using Azure Logic Apps
#### Prerequisites
Before we begin, ensure you have the following:
  - Azure Storage Account with a blob container (e.g., reports) where your Azure Function saves the CSV reports.
  - Azure Logic App (Consumption Plan) created in the same region as your storage account.
  - Email Service: Either an Office 365 Outlook account or a SendGrid account for sending emails.

### Step 1: Create a Logic App
1. Navigate to the Azure Portal: https://portal.azure.com

2. Create a New Logic App:
  - Search for "Logic Apps" in the top search bar.
  - Click on "Create" and fill in the required details:
    - Subscription: Select your Azure subscription.
    - Resource Group: Choose an existing one or create a new one.
    - Logic App Name: e.g., SendBlobReportEmail.
    - Region: Same as your storage account.
    - Plan Type: Consumption.
    - Click "Review + create" and then "Create".

### Step 2: Define the Trigger
1. Open the Logic App Designer:
  - Once the Logic App is deployed, click on "Go to resource".
  - Under "Development Tools", click on "Logic App Designer".

2. Choose a Trigger:
  - In the search box, type "blob".
  - Select "When a blob is added or modified (V2)".

3. Configure the Trigger:
  - Storage Account Connection: Create a new connection using the storage account name and key.
  - Container Name: Select the container where your Azure Function saves the CSV reports (e.g., reports).
  - Blob Path Begins With: Leave blank unless you want to filter specific folders or filenames.
  - How often do you want to check for items?: Set the interval (e.g., every 1 hour).
    
### Step 3: Get Blob Content
1. Add a New Step:
  - Click on "New step".
  - Search for "Get blob content (V2)" and select it.

2. Configure the Action:
  - Storage Account Connection: Use the same connection as before.
  - Blob: Click on the field and select "Path" from the dynamic content (this refers to the blob that triggered the Logic App).

### Step 4: Send Email with Attachment
1. Add a New Step:
  - Click on "New step".
  - Depending on your email service, choose one of the following:
      - Office 365 Outlook: Search for "Send an email (V2)".
      - SendGrid: Search for "Send email (V4)".

2. Configure the Email Action:
  - To: Enter the recipient's email address.
  - Subject: e.g., "Monthly Azure Cost Optimization Report".
  - Body: e.g., "Please find the attached monthly report on Reserved Instances and Savings Plans utilization."
  - Attachments:
      - Click on "Add new parameter" and select "Attachments".
      - For Attachment Name, use the expression:
        ```bash
        last(split(triggerOutputs()?['headers']['x-ms-file-last-modified'],'/'))
        ```
        This will extract the filename from the blob path.
      - For Attachment Content, select "File Content" from the dynamic content (output from the "Get blob content" action).
   
### Step 5: Save and Test the Logic App
1. Save the Logic App:
  - Click on "Save" at the top of the designer.

2. Test the Workflow:
  - Upload a new CSV report to the specified blob container.
  - The Logic App should trigger, retrieve the blob content, and send an email with the CSV file attached.
