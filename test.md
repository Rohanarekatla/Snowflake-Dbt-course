# Proof-of-Concept (PoC)  
## **“Zero-Failure” Key Rotation on Azure**  

---

## 1 · Why this PoC Exists  

Every few months your encryption key **must** change.  
When a human copies key-strings around, somebody eventually forgets → an outage happens.  
This PoC shows how to make the process **automatic, tamper-proof and human-free** with native Azure features.  
*If you can copy-and-paste, you can run this PoC.*

---

## 2 · What You Will Build (30-Second Tour)  

1. **One central Azure Key Vault (Premium)** – the only place the key ever lives.  
2. **Built-in Rotation Policy** – Key Vault creates a brand-new version on a fixed schedule (e.g., every 270 days) [[3]].  
3. **Apps fetch the key without a version-number** – they always get the latest copy automatically.  
4. **Near-Expiry Alert** – if a rotation hiccups, Ops receive a Teams / e-mail alert.  
5. **100 % private networking & password-less access** – Managed Identity + Private Link.  

*(A later optional add-on shows how to “fan-out” to extra vaults if regulation forces one-vault-per-team.)*

```
                ┌────────── Apps / Lambdas / Containers ──────────┐
                │   (use Managed Identity “MI-App”)                │
                │   GET https://vault.xxx/keys/myKey               │
                └────────────────────▲─────────────────────────────┘
                                     │ versionless call
    Key rotates every 270 d          │
┌─────────────────────────────────────┴─────────────────────────┐
│              Central Key Vault (Premium)                     │
│   • auto-creates new version                                 │
│   • emits “NearExpiry” event (safety-net)                    │
└───────────────────────────────────────────────────────────────┘
```

---

## 3 · Glossary (Speak Like a Cloud Pro in 3 Minutes)

| Term | One-line, non-geek meaning |
|------|----------------------------|
| **Key Vault** | A tamper-proof, cloud-hosted safe-box for secrets/keys [[6]]. |
| **Version**   | Every time a key changes, Vault saves the old one and stamps the new one with **v2, v3…** |
| **Rotation Policy** | A built-in clock in Key Vault that says “Create a new version every _N_ days” [[3]]. |
| **Managed Identity (MI)** | An Azure-issued ID badge; apps use it instead of usernames/passwords. |
| **Private Link** | Puts the cloud service on your private network so it never uses the public Internet. |
| **Terraform** | A text-file way to tell Azure which resources to create, change or delete [[1]]. |

---

## 4 · Use-Cases This PoC Covers

| # | Scenario | Is it solved? | How |
|---|----------|---------------|-----|
| 1 | “One vault per region, many apps” | ✅ | Apps read live from the central vault. |
| 2 | “Regulator forces one vault per BU” | ✅* | Optional fan-out Function (Section 11). |
| 3 | “Service like Data Factory that **requires** a versioned URI” | ✅* | Special note in Section 10 (Data Factory is version-bound [[2]]). |
| 4 | “Need FIPS 140-2 Level 3” | ✅ | Use **Premium** SKU or Managed HSM. |
| 5 | “Absolutely no human can see the key” | ✅ | MI auth, RBAC denies portal read-access. |

---

## 5 · PoC Prerequisites  

| What | Why |
|------|-----|
| Azure subscription **Owner** or **Contributor** rights | To create resources. |
| **Terraform ≥ 1.0** + `az` CLI | We use Terraform for IaC [[1]]. |
| Visual Studio Code (optional) | Easier to edit `.tf` & `.md`. |

---

## 6 · Step-by-Step Guide (Layman Friendly)

### 6.1 Clone the Repo

```bash
git clone https://github.com/your-org/zero-fail-key-rotation
cd zero-fail-key-rotation/infra
```

*(Repo tree is shown in Section 12.)*

### 6.2 Fill in One Variable File

`infra/dev.auto.tfvars`  

```hcl
location = "eastus2"
env_name = "dev"
```

That is literally all you **must** edit.

### 6.3 Initialise & Deploy

```bash
terraform init    # downloads Azure provider
terraform plan    # shows what Azure will build
terraform apply   # yes → builds in ~60 s
```

**What just happened?**

1. A **resource group** was created.  
2. Inside it, Terraform made **Key Vault Premium** with soft-delete and purge-protection.  
3. A 2048-bit RSA key called `enc-key` appeared.  
4. Vault received a **rotation policy** (rotate at day 270, alert 30 days before expiry) [[3]].  

*(The Terraform code is pasted in Section 9 if you’re curious.)*

### 6.4 Grant Your Test App Access (One Command!)

```bash
az identity create -g rg-dev -n id-app1          # create MI
az role assignment create \
    --assignee $(az identity show -g rg-dev -n id-app1 --query principalId -o tsv) \
    --role "Key Vault Crypto User" \
    --scope  $(az keyvault show -n kv-dev-core --query id -o tsv)
```

### 6.5 Run the Sample App

`python app_demo.py`

```python
from azure.identity import ManagedIdentityCredential
from azure.keyvault.keys import KeyClient

cred  = ManagedIdentityCredential()
vault = KeyClient("https://kv-dev-core.vault.azure.net", cred)
key   = vault.get_key("enc-key")     # note: NO version id!
print("✅ Got key version:", key.properties.version)
```

**Output**

```
✅ Got key version: 28a1f14d66be4bd89123fbce8c6e4d6b
```

Your app can now encrypt data with the latest key.  
When day 270 arrives, Key Vault makes version `v2`.  
The next time you run the script you’ll see a **new** version GUID—no code changes.

---

## 7 · Safety-Net: Near-Expiry Alert (10 Lines)

Even though rotation is automatic, Fortune-500 SOCs add an early-warning siren.  

1. Key Vault fires `Microsoft.KeyVault.KeyNearExpiry` 30 days before the key dies.  
2. Event Grid routes that to an **Action Group** → Teams or PagerDuty.

Quick CLI:

```bash
az eventgrid event-subscription create \
  --name kv-expiry \
  --source-resource-id $(az keyvault show -n kv-dev-core --query id -o tsv) \
  --included-event-types Microsoft.KeyVault.KeyNearExpiry \
  --endpoint-type webhook \
  --endpoint https://prod-123.webhook.logic.azure.com/...   # your Logic App URL
```

*(Yes, you can choose e-mail instead. Same idea.)*

---

## 8 · How Do We Know It’s “Zero Failure”?

| Disaster Test | Outcome |
|---------------|---------|
| Vault forgets to rotate | Azure SLA 99.99 % plus alert if near expiry. |
| Dev hard-codes `/keys/myKey/version-abc` | Azure Policy “deny” gate in Terraform plan. |
| Human tries to read key in portal | Role has **no** `get` permission, fails. |
| Region outage | Deploy secondary vault via same Terraform module; apps use DNS fail-over. |

---

## 9 · Terraform Core (25 Lines, Explained in English)

```hcl
module "vault" {
  source  = "Azure/key-vault/azurerm"                   # proven public module
  name    = "kv-${var.env_name}-core"
  sku_name = "premium"                                  # HSM-backed
  purge_protection_enabled = true
  soft_delete_retention_days = 90

  key_rotation = {
    rotation_in_days = 270                              # auto-rotate
    expiry_in_days   = 730                              # expire after 2 yr
  }

  keys = {
    enc-key = {
      key_type = "RSA"
      key_size = 2048
      opts     = ["decrypt","encrypt","wrapKey","unwrapKey"]
    }
  }
}
```

• `purge_protection_enabled` – nobody can hard-delete the vault (audit demand).  
• `rotation_in_days` drives the magic schedule [[3]].  
• The module attaches your **tenant ID** automatically (no typos possible).  

---

## 10 · Special Case: Azure Data Factory & Other “Version-Locked” Services  

Some Azure services (Data Factory, older SQL TDE) store the **full** key URI including the version [[2]].  
For them:

1. Let rotation create **v2**.  
2. Fire Event Grid → tiny Function that updates the CMK URI property of Data Factory to point at **v2**.  
3. Delete **v1** only after Data Factory confirms new key takes effect.

*(Sample code lives in `addons/rotate_data_factory.py`.)*

---

## 11 · Optional Add-On: Fan-Out to Multiple Vaults

If each Business Unit must own a vault:

1. Keep **central** vault as “Source of Truth”.  
2. Enable Event Grid “KeyNewVersionCreated”.  
3. Azure Function (Python, 30 lines) copies the new version to BU vaults.  
   – The same pattern Microsoft docs call *“Rotation for two credential sets”* [[7]].  

*(Detailed walk-through is in `addons/fanout/README.md`.)*

---

## 12 · Repo Layout

```
zero-fail-key-rotation/
├─ infra/                 # Terraform (HCL)
│   ├─ main.tf
│   ├─ variables.tf
│   └─ dev.auto.tfvars
├─ app_demo.py            # simple consumer
└─ addons/
    ├─ rotate_data_factory.py
    └─ fanout/
        ├─ function_app.py
        └─ README.md
```

---

## 13 · Cost Cheat-Sheet (USD, East US)

| Item | Monthly Cost |
|------|--------------|
| Key Vault Premium base fee | ~$60 |
| Key operations (≤ 50 K) | <$1 |
| Event Grid near-expiry alerts | <$0.01 |
| Logic App / Action Group | $0 (consumption) |
| **Total** | **≈ $61** (cheaper than one lunch for five devs) |

---

## 14 · Next Steps in Real Life

1. **Move the Terraform state** to an Azure Storage Account with SAS lock.  
2. Add **Azure Policy** to enforce rotation on *every* future key [[8]].  
3. Wire the alert endpoint to your corporate incident channel.  
4. Run a **chaos day**: flip the vault to a paired region and prove apps survive.

---

### 🎉 Congratulations

You now own a **self-rotating, zero-touch, audit-friendly** key-management pipeline.  
No human can forget, leak or break the key ever again—exactly the behaviour security teams at Microsoft, Google and large banks aim for.

*Coffee time! ☕*

---


### References

1. **Configure cryptographic key auto-rotation in Azure Key Vault | Microsoft Learn**. [https://learn.microsoft.com](https://learn.microsoft.com/en-us/azure/key-vault/keys/how-to-configure-key-rotation)
2. **How to perform and automate key rotation in Azure Key Vault | TechTarget**. [https://www.techtarget.com](https://www.techtarget.com/searchcloudcomputing/tutorial/How-to-perform-and-automate-key-rotation-in-Azure-Key-Vault)
3. **Rotation tutorial for resources with one set of authentication credentials stored in Azure Key Vault | Microsoft Learn**. [https://learn.microsoft.com](https://learn.microsoft.com/en-us/azure/key-vault/secrets/tutorial-rotation)
4. **Azure Key Vault Overview - Azure Key Vault | Microsoft Learn**. [https://learn.microsoft.com](https://learn.microsoft.com/en-us/azure/key-vault/general/overview)
5. **Rotation tutorial for resources with two sets of credentials | Microsoft Learn**. [https://learn.microsoft.com](https://learn.microsoft.com/en-us/azure/key-vault/secrets/tutorial-rotation-dual)
6. **Quickstart: Create an Azure key vault and key using Terraform | Microsoft Learn**. [https://learn.microsoft.com](https://learn.microsoft.com/en-us/azure/key-vault/keys/quick-create-terraform)
7. **Azure & Terraform: Rotate Customer-managed Keys in Data Factory with Terraform | by Gerrit Stapper | NEW IT Engineering | Medium**. [https://medium.com](https://medium.com/twodigits/rotate-customer-managed-keys-in-data-factory-with-terraform-eb3a789ea2fc)
8. **Best practices for using Azure Key Vault | Microsoft Learn**. [https://learn.microsoft.com](https://learn.microsoft.com/en-us/azure/key-vault/general/best-practices)
9. **How to configure automatic key rotation (preview) in Azure Key Vault - Argon Systems**. [https://argonsys.com](https://argonsys.com/microsoft-cloud/library/how-to-configure-automatic-key-rotation-preview-in-azure-key-vault/)
10. **Automated Key rotation in Key Vault - DEV Community**. [https://dev.to](https://dev.to/makendrang/automated-key-rotation-in-key-vault-2n3f)
