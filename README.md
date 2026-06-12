# Azure Resource Locks: Governance and Safety Guardrails

Step-by-Step Practical Lab Guide for Portal and CLI + Governance Training

## 1. Introduction and Core Concepts

In cloud environments, human error is one of the most common causes of system downtime. A single accidental deletion or misconfiguration can disrupt business continuity. Azure Resource Locks provide a mechanism to lock resources, preventing unauthorized modifications or deletions regardless of user permissions.

### Lock Types
- **CanNotDelete**: Authorized users can read and modify a resource, but cannot delete it. Ideal for production databases, key networks, and storage accounts.
- **ReadOnly**: Authorized users can only read a resource, but cannot delete or modify it. Behaves similarly to restricting all users to the Reader role.

### Lock Inheritance
Azure Resource Locks follow a hierarchical inheritance model. When you apply a lock at a parent scope (Subscription or Resource Group), all child resources within that scope automatically inherit the lock.

> **Important Note on RBAC**  
> Resource locks apply to the Azure Resource Manager control plane and override user permissions. Even a Subscription Owner must explicitly delete the lock first before modifying/deleting the resource.

## 2. Phase 1: Provisioning the Test Environment

### Portal Steps
#### Step 1: Create Resource Group
1. Azure Portal → 'Resource groups' → '+ Create'
2. Subscription: 'Azure subscription 1'
3. Name: `rg-locks-learning-prod`, Region: `West Europe`
4. 'Review + create' → 'Create'

#### Step 2: Provision Storage Account
1. 'Storage accounts' → '+ Create'
2. Resource Group: `rg-locks-learning-prod`
3. Name: `salearningprod` + 6 random digits
4. Performance: `Standard`, Redundancy: `LRS`
5. 'Review + create' → 'Create'

#### Step 3: Provision NSG
1. 'Network security groups' → '+ Create'
2. Resource Group: `rg-locks-learning-prod`
3. Name: `nsg-learning-prod`
4. 'Review + create' → 'Create'

#### Step 4: Provision VM
1. 'Virtual machines' → '+ Create' → 'Azure virtual machine'
2. Resource Group: `rg-locks-learning-prod`
3. VM Name: `vm-learning-prod`, Image: `Ubuntu Server 22.04 LTS`
4. Size: `Standard_D2s_v5`
5. Auth: `Password`, Username: `azureuser`
6. 'Review + create' → 'Create'

> **West Europe Capacity Warning**  
> If you encounter 'SkuNotAvailable' for B-series VM size, switch to D-series like `Standard_D2s_v5`.

## 3. Phase 2: Applying and Testing Resource-Level Locks

### Portal Method

#### Task 1: Apply 'CanNotDelete' Lock to Storage Account
1. Navigate to Storage Account → 'Settings' → 'Locks' → '+ Add'
2. Lock Name: `lock-sa-delete`, Lock Type: `Delete` → 'OK'

**Testing CanNotDelete:**
- **Modify**: Add tag `Project = AzureLocks` → Saves successfully
- **Delete**: Try delete → Portal error: scope is locked

#### Task 2: Apply 'ReadOnly' Lock to NSG
1. NSG `nsg-learning-prod` → 'Settings' → 'Locks' → '+ Add'
2. Lock Name: `lock-nsg-readonly`, Lock Type: `Read-only` → 'OK'

**Testing ReadOnly:**
- **Modify**: Add Inbound rule port `80` → Fails: cannot perform write operations
- **Delete**: Try delete NSG → Fails with locked scope notification

#### Task 3: Apply 'ReadOnly' Lock to VM
1. VM `vm-learning-prod` → 'Settings' → 'Locks' → '+ Add'
2. Lock Name: `lock-vm-readonly`, Lock Type: `Read-only` → 'OK'

**Testing VM Power States:**
- **Stop/Start**: Both fail. ReadOnly blocks ARM control plane writes
- **Data Plane**: SSH still works if VM was running. Data plane not blocked

### Azure CLI Method

#### Prerequisites
```bash
az login
az account set --subscription "Azure subscription 1"
# CanNotDelete on Storage Account
az lock create \
  --name lock-sa-delete \
  --lock-type CanNotDelete \
  --resource-group rg-locks-learning-prod \
  --resource-name salearningprod899756 \
  --resource-type Microsoft.Storage/storageAccounts

# ReadOnly on NSG
az lock create \
  --name lock-nsg-readonly \
  --lock-type ReadOnly \
  --resource-group rg-locks-learning-prod \
  --resource-name nsg-learning-prod \
  --resource-type Microsoft.Network/networkSecurityGroups

# ReadOnly on VM
az lock create \
  --name lock-vm-readonly \
  --lock-type ReadOnly \
  --resource-group rg-locks-learning-prod \
  --resource-name vm-learning-prod \
  --resource-type Microsoft.Compute/virtualMachines
LIST AND DELETE LOCKS
# List all locks in RG
az lock list --resource-group rg-locks-learning-prod --output table

# Delete a lock to unlock
az lock delete --name lock-sa-delete --resource-group rg-locks-learning-prod
4. Phase 3: Verifying Lock Inheritance
Portal Method
Delete individual locks from Storage Account, NSG, and VMRG rg-locks-learning-prod → 'Settings' → 'Locks' → '+ Add'Lock Name: lock-rg-level, Type: Delete → 'OK'Go to Storage Account → 'Locks' → See inherited lock-rg-levelTry delete Storage Account → Blocked by inherited RG lockAzure CLI Methodbash# Apply lock at Resource Group level
az lock create \
  --name lock-rg-level \
  --lock-type CanNotDelete \
  --resource-group rg-locks-learning-prod

# Verify inheritance - check locks on storage account
az lock list \
  --resource-group rg-locks-learning-prod \
  --resource-name salearningprod899756 \
  --resource-type Microsoft.Storage/storageAccounts5. Phase 4: Cascading Protection and RBAC Overrides
Test Deletion
RG Overview → 'Delete resource group' → ConfirmObservation: Deletion fails due to RG-level lockEven as Subscription Owner, the lock blocks deletion. This proves locks are governance guardrails, not RBAC.Remove Lock with CLIbashaz lock delete --name lock-rg-level --resource-group rg-locks-learning-prod6. Advanced: Automating Locks with Azure Policy
Deploy locks at scale using deployIfNotExists policy:Portal → 'Policy' → 'Definitions' → '+ Policy definition'Rule: If resource has tag LockStatus = CanNotDelete, deploy Microsoft.Authorization/locksAssign policy to SubscriptionCLI to create policy assignment:bashaz policy assignment create \
  --name 'auto-apply-cannotdelete-locks' \
  --policy 'policy-definition-id' \
  --scope /subscriptions/<sub-id>7. Why ReadOnly Lock Blocks VM Start/Stop
Control Plane vs Data Plane
Control Plane: Managed by Azure Resource Manager API. Start/stop VM requires HTTP POST to update powerState, allocate/deallocate compute, change billing. ReadOnly blocks PUT, DELETE, POST.Data Plane: OS-level traffic like SSH, web requests. Not blocked by ReadOnly lock.Screenshots
Add portal screenshots here:![Storage Lock](./images/storage-lock.png)![NSG Lock](./images/nsg-lock.png)![VM Lock](./images/vm-lock.png)![RG Lock](./images/rg-lock.png)![Inherited Lock](./images/inherited-lock.png)Cleanupbash# Remove all locks first
az lock delete --name lock-rg-level --resource-group rg-locks-learning-prod

# Then delete resource group
az group delete --name rg-locks-learning-prod --yes --no-waitjavascript
**To download:**
1. **Mobile**: Long press the code block → Select All → Copy → Open Notes/Files app → Paste → Save as `README.md`
2. **Desktop**: Copy all → Open VS Code/Notepad → Paste → Save as `README.md`

