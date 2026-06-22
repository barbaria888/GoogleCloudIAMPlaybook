# <div align="center"><img src="https://github.com/barbaria888/GoogleCloudIAMPlaybook/blob/main/images/Identity%20And%20Access%20Management.png" height="70"> Google Cloud IAM Custom Roles Guide </div>

Welcome to the **IAM Custom Roles** repository. This guide walks you through understanding, creating, updating, disabling, and deleting Custom Cloud IAM roles using the Google Cloud SDK (`gcloud`) and YAML definitions.

Custom roles allow you to enforce the **Principle of Least Privilege**, ensuring that users and service accounts in your organization have only the exact permissions required to perform their tasks.



---

## 🏗️ Repository Structure

All YAML configuration files are organized in the `roles/` folder:
- 📁 **`roles/`**
  - [`role-definition.yaml`](file:///d:/HP/Documents/coursera/GCP/GKE/MultiClusterGKE/custom-iam/roles/role-definition.yaml) - Initial definition for the Custom Role Editor.
  - [`new-role-definition.yaml`](file:///d:/HP/Documents/coursera/GCP/GKE/MultiClusterGKE/custom-iam/roles/new-role-definition.yaml) - Updated definition for the Custom Role Editor (with Storage permissions).
  - [`viewer.yaml`](file:///d:/HP/Documents/coursera/GCP/GKE/MultiClusterGKE/custom-iam/roles/viewer.yaml) - YAML definition used to update/restore the Custom Role Viewer.
- 📁 **`images/`** - Screenshots demonstrating terminal commands and console states.

---

## 🔑 Key Concepts

> [!IMPORTANT]
> - Custom roles can be created at the **Organization** or **Project** level. They **cannot** be created at the **Folder** level.
> - Permissions in IAM follow the format: `<service>.<resource>.<verb>` (e.g., `compute.instances.list`).
> - You cannot grant custom roles from one project or organization on a resource owned by another project or organization.

### Required Administrative Roles
To administer custom roles, you must have the `iam.roles.create` permission, typically granted via:
* **Organization Role Administrator** (`roles/iam.organizationRoleAdmin`)
* **IAM Role Administrator** (`roles/iam.roleAdmin`)

---

## 🚀 Step-by-Step Walkthrough

### Step 1: Querying IAM Environment
Before creating custom roles, check what permissions and roles are available:

1. **List all testable permissions** on a project:
   ```bash
   gcloud iam list-testable-permissions //cloudresourcemanager.googleapis.com/projects/$DEVSHELL_PROJECT_ID
   ```
2. **Describe a predefined role** to view its metadata and included permissions:
   ```bash
   gcloud iam roles describe roles/viewer
   ```
3. **List grantable roles** on a resource:
   ```bash
   gcloud iam list-grantable-roles //cloudresourcemanager.googleapis.com/projects/$DEVSHELL_PROJECT_ID
   ```

---

### Step 2: Creating Custom Roles

You can create a custom role using either a **YAML definition file** or **CLI flags**.

#### Method A: Using a YAML File
Define the role in a file like [`role-definition.yaml`](file:///d:/HP/Documents/coursera/GCP/GKE/MultiClusterGKE/custom-iam/roles/role-definition.yaml):
```yaml
title: "Role Editor"
description: "Edit access for App Versions"
stage: "ALPHA"
includedPermissions:
- appengine.versions.create
- appengine.versions.delete
```

Apply the definition:
```bash
gcloud iam roles create editor --project $DEVSHELL_PROJECT_ID --file roles/role-definition.yaml
```

#### Method B: Using CLI Flags
Create the role directly using flags:
```bash
gcloud iam roles create viewer --project $DEVSHELL_PROJECT_ID \
  --title "Role Viewer" \
  --description "Custom role description." \
  --permissions compute.instances.get,compute.instances.list \
  --stage ALPHA
```

---

### Step 3: Listing Custom Roles
To view the custom roles created in your project:
```bash
gcloud iam roles list --project $DEVSHELL_PROJECT_ID
```

![Custom Roles Listed in Project](/images/custom-roles-in-current-project.png)

---

### Step 4: Updating a Custom Role (Concurrency Control with Etags)

> [!NOTE]
> Cloud IAM uses `etag` properties to prevent concurrent modification conflicts. The `etag` value changes with every update, ensuring that two operators do not overwrite each other's changes.

#### Method A: Using a YAML File
1. Fetch the latest role metadata (which includes the current `etag`):
   ```bash
   gcloud iam roles describe editor --project $DEVSHELL_PROJECT_ID
   ```
2. Save the description to [`new-role-definition.yaml`](file:///d:/HP/Documents/coursera/GCP/GKE/MultiClusterGKE/custom-iam/roles/new-role-definition.yaml) and add new permissions (e.g., `storage.buckets.get` and `storage.buckets.list`):
   ```yaml
   description: Edit access for App Versions
   etag: BwVxIAbRq_I=
   includedPermissions:
   - appengine.versions.create
   - appengine.versions.delete
   - storage.buckets.get
   - storage.buckets.list
   name: projects/qwiklabs-gcp-04-d6ba91c0de6b/roles/editor
   stage: ALPHA
   title: Role Editor
   ```
3. Update the role:
   ```bash
   gcloud iam roles update editor --project $DEVSHELL_PROJECT_ID --file roles/new-role-definition.yaml
   ```

#### Method B: Using CLI Flags
To add or remove permissions inline:
```bash
gcloud iam roles update viewer --project $DEVSHELL_PROJECT_ID \
  --add-permissions storage.buckets.get,storage.buckets.list
```

![Roles Updated via Command Line and YAML](/images/roles-updated0using-yaml-as-well-as-commandline.png)

---

### Step 5: Disabling and Deleting Roles

* **Disable a role** (temporarily deactivates all policies bound to it):
  ```bash
  gcloud iam roles update viewer --project $DEVSHELL_PROJECT_ID --stage DISABLED
  ```
* **Delete a role** (deactivates the role and marks it for permanent deletion after 37 days):
  ```bash
  gcloud iam roles delete viewer --project $DEVSHELL_PROJECT_ID
  ```

---

## 🔍 Observed Behavior: Restoration via YAML Update

### 1. The Unexpected Discovery
When a custom role is deleted in Cloud IAM, it enters a soft-deleted/disabled state. The officially documented method to restore it is:
```bash
gcloud iam roles undelete viewer --project $DEVSHELL_PROJECT_ID
```

However, during testing, we observed an alternative behavior when attempting a restoration workflow:
1. The role `viewer` was deleted, transitioning its stage to `DISABLED`.
2. A configuration file [`viewer.yaml`](file:///d:/HP/Documents/coursera/GCP/GKE/MultiClusterGKE/custom-iam/roles/viewer.yaml) was prepared, setting the launch stage to `GA` and containing the matching `etag`.
3. Instead of running `undelete` first, the role was updated directly via:
   ```bash
   gcloud iam roles update viewer --project $DEVSHELL_PROJECT_ID --file roles/viewer.yaml
   ```
   *This update succeeded and transitioned the role's launch stage back to `GA`.*
4. When subsequently executing the official undelete command:
   ```bash
   gcloud iam roles undelete viewer --project $DEVSHELL_PROJECT_ID
   ```
   It threw a precondition error:
   ```
   ERROR: (gcloud.iam.roles.undelete) FAILED_PRECONDITION: A role that is not deleted cannot be undeleted
   ```

![Yaml Update to GA stage succeeded](/images/wrote-a-yaml-and-applied-to-viewer-role-again-it-got-GAstage.png)

---

### 2. Analysis & Verification
To verify this finding, we compared the outputs of `gcloud iam roles describe` before and after running the `update` command.

#### Before the YAML Update (Deleted State)
```yaml
description: Custom role description.
etag: BwZU0S4fPbc=
includedPermissions:
- compute.instances.get
- compute.instances.list
- storage.buckets.get
- storage.buckets.list
name: projects/qwiklabs-gcp-04-d6ba91c0de6b/roles/viewer
stage: DISABLED
title: Role Viewer
```

#### After the YAML Update (Restored State)
```yaml
description: Custom role description.
etag: BwZU0UEBVOg=
includedPermissions:
- compute.instances.get
- compute.instances.list
- storage.buckets.get
- storage.buckets.list
name: projects/qwiklabs-gcp-04-d6ba91c0de6b/roles/viewer
stage: GA
title: Role Viewer
```

As shown in the output, the YAML update successfully transitioned the launch stage from `DISABLED` back to `GA` and generated a new `etag` (`BwZU0UEBVOg=`), effectively reactivating the role. Because the role was now active, the subsequent `undelete` call returned a `FAILED_PRECONDITION` error.

![Undelete Precondition Failure Screenshot](/images/no-undelete-.png)

---

### 3. Conclusion & Best Practices
> [!WARNING]
> While updating a deleted custom role with an active launch stage via YAML successfully reactivated the role in this environment, this behavior is an **observed experimental finding** and is not officially documented or guaranteed as a primary restoration mechanism. It may be a side effect of how the API processes the state transition payload.
> 
> For production environments and reliable automation, the officially documented method for restoring custom roles remains:
> ```bash
> gcloud iam roles undelete
> ```

