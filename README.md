# azure-backup-system
An automated backup setup on Azure Blob Storage with file versioning, lifecycle-based cost tiering, and a scheduled workflow that emails a confirmation of what is protected.

![Backup confirmation email]
<img width="1365" height="641" alt="Screenshot 2026-10-08 213732" src="https://github.com/user-attachments/assets/fda767da-e23e-452e-8dc6-826f3377dd88" />


## The problem

Many small businesses back up by occasionally copying files to an external drive. When the person responsible is away or forgets, nothing gets backed up, and when something is deleted or overwritten, it is simply gone. Even when backups do run, nobody finds out whether they worked.

## What I built

A backup environment that answers three questions automatically:

1. **Can I get an older copy back?** Blob versioning keeps every version of a file.
2. **Is it costing more than it should?** A lifecycle rule moves old files to cheaper storage tiers.
3. **Did it work?** A scheduled Logic App checks the storage and emails a confirmation.

### Architecture

```
Azure Storage Account (LRS, South Africa North)
 ├── documents/          ← versioning enabled
 ├── database-exports/
 └── app-files/
        │
        ├── Lifecycle rule: Cool after 30 days, Archive after 90 days
        │
        └── Logic App (Consumption)
              Recurrence trigger (daily)
                → Lists blobs (V2) in "documents"
                → Send an email (V2) via Outlook.com
```

| Component | Configuration |
|---|---|
| Storage account | `stportfoliobackup0001`, Standard performance, LRS, South Africa North |
| Containers | `documents`, `database-exports`, `app-files` (all private) |
| Data protection | Blob versioning on, keep all versions |
| Lifecycle rule | Cool after 30 days, Archive after 90 days since last modification |
| Automation | Logic App `logic-backup-confirm`, Consumption (multi-tenant) plan |
| Notification | Outlook.com connector, count of blobs in `documents` |
| Resource group | `rg-portfolio` |

## Build steps

### 1. Storage account and containers

Created a Standard, locally redundant storage account in `rg-portfolio` with Blob Storage as the primary service, then three private containers to separate data by type. Separating documents, database exports and application files makes a restore under pressure much faster, because you know where to look.

![Storage account review]
<img width="1362" height="689" alt="Screenshot 2026-10-08 191938" src="https://github.com/user-attachments/assets/0e570a95-14ca-4f65-9243-27fd1e1bb345" />

![Containers list]
<img width="1365" height="646" alt="Screenshot 2026-10-08 192928" src="https://github.com/user-attachments/assets/1b8c12bf-b38f-439b-a619-29123701f409" />


I uploaded a few harmless sample files into each container so there was something to protect.

### 2. Versioning

Enabled **versioning for blobs** under Data protection, keeping all versions.

To test it, I uploaded `report.txt`, overwrote it with different content, and confirmed two versions appeared. I then restored the older one using **Make current version**. Restoring does not rewind the file; it copies the old version forward as a new one, so the history ended with **three versions** and the current content matching the original.

![Versioning enabled]
<img width="1365" height="657" alt="Screenshot 2026-10-08 194404" src="https://github.com/user-attachments/assets/27ec5ce4-160c-4c67-bf3d-115645c2f447" />

![Two versions listed]
<img width="1365" height="646" alt="Screenshot 2026-10-08 195414" src="https://github.com/user-attachments/assets/f11adb79-f241-4697-a9da-c1bfcc5eca7e" />

![After restore: three versions]
<img width="1365" height="651" alt="Screenshot 2026-10-08 215531" src="https://github.com/user-attachments/assets/61ce1427-73b8-4ed6-a8b6-d2cba9b3b2e0" />

### 3. Lifecycle management

Added a rule, `archive-old-files`, that moves blobs to **Cool** storage after 30 days without modification and to **Archive** after 90 days. The data stays safe and retrievable while the cost per GB drops over time. Nothing moves yet because the files are new; the configuration is the deliverable.

![Lifecycle rule]
<img width="1365" height="654" alt="Screenshot 2026-10-08 200030" src="https://github.com/user-attachments/assets/f2d07383-9b90-4f1f-b9d8-01c90edd48f6" />

### 4. Daily confirmation workflow

Built a Logic App on the Consumption plan, which bills per action and includes a monthly free allowance, so it costs nothing at this scale.

1. **Recurrence** trigger, once per day
2. **Lists blobs (V2)** against the `documents` container, using an access-key connection to the storage account
3. **Send an email (V2)** through the Outlook.com connector, with the body built from the expression:

```
length(body('Lists_blobs_(V2)')?['value'])
```

Ran it manually to test. Both runs succeeded, and the email arrived reading **"Files currently protected in documents: 4"**.

![Logic App designer]
<img width="1365" height="630" alt="Screenshot 2026-10-08 213638" src="https://github.com/user-attachments/assets/c1d512b8-a738-4f81-b59e-99c26071f5c9" />

![Run history: succeeded]
<img width="1365" height="650" alt="Screenshot 2026-10-08 213545" src="https://github.com/user-attachments/assets/5be31d47-35b8-4459-9118-55d3516243db" />


## Results

| Check | Result |
|---|---|
| Files in `documents` at test time | 4 |
| Versions of `report.txt` after overwrite and restore | 3 |
| Lifecycle tiers | Cool at 30 days, Archive at 90 days |
| Logic App test runs | 2 of 2 succeeded |
| Confirmation email delivered | Yes |
| Total cost | Under US$0.01 (Logic Apps and Storage combined) |

## Problems I hit and how I fixed them

**Upload failed with "The requested file could not be read."** The error was about the file on my PC, not Azure. The browser could not read it reliably from its original location. Saving a fresh copy into a plain local folder and uploading from there fixed it.

**Almost created the wrong "container" resource.** Searching for "container" in the portal surfaces Container Instances, which is a compute service that runs apps and can cost money. The containers this project needs live *inside* the storage account, under Data storage. Worth knowing because the names are easy to confuse.

**Workflow validation failed on the email step.** The expression referenced `List_blobs_(V2)`, but the action is actually named `Lists blobs (V2)`. Logic App expressions refer to actions by their exact name with spaces replaced by underscores, so one missing letter broke validation. Correcting the name in the formula fixed it.

**The connection form would not submit.** The "Create new" button stayed greyed out because the storage account name field was empty. Required fields on connector forms do not always flag themselves clearly.

## Decisions I made and why

**LRS instead of the portal's recommendation.** The portal's default recommendation for this workload was read-access geo-redundant storage (RA-GRS). I changed it to locally redundant storage (LRS), which keeps three copies in one data centre. For a practice environment with no real data, the extra cost of geo-replication is not justified. For a real business backup, I would use geo-redundancy so the data survives the loss of an entire region.

**Versioning over relying on soft delete alone.** Soft delete protects against deletion, but versioning also protects against the more common failure: a file being overwritten with bad data. Versioning covers both.

**Consumption plan for the Logic App.** Billing per action rather than per hour means a once-a-day workflow costs essentially nothing, compared with the dedicated Standard plans.

## What I would improve

- **Replace the access key with a managed identity.** The Logic App connects with a storage account key, which is powerful and has to be guarded. A managed identity with a role assignment removes the secret entirely.
- **Control version costs.** Keeping all versions forever grows storage over time. I would add a lifecycle rule to move or delete old *versions* after a set period.
- **Make the confirmation more meaningful.** The email counts files, but it does not verify that anything changed or that backups are fresh. A better check would flag when no new or modified blobs have appeared within an expected window.
- **Cover all three containers** in the daily report, not just `documents`.
- **Add failure alerting.** A failed workflow run currently sends nothing. An alert on run failures would catch a silently broken backup check.
- **Deploy it as code** with Bicep or Terraform so the whole environment can be rebuilt in one command.

## Cost control

- Free-tier-friendly choices throughout: Standard LRS storage, Consumption Logic App
- A monthly budget with alerts at 50%, 80% and 100% set before building anything
- Everything in one resource group, deleted after testing
- Confirmed afterwards in Cost analysis: actual cost under US$0.01 for the month against a US$20 budget

![Cost analysis after teardown]
<img width="1365" height="697" alt="Screenshot 2026-10-10 125313" src="https://github.com/user-attachments/assets/7abd29c5-8f9b-4112-bdc6-605f48b5ed07" />

## Skills demonstrated

Azure Blob Storage, blob versioning and recovery, storage lifecycle management, access tiers (Hot, Cool, Archive), Azure Logic Apps, workflow automation, connector configuration, Azure Cost Management, troubleshooting, technical documentation.
