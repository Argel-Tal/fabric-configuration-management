# Microsoft Fabric Workspace access management automation

## Introduction

This tooling provides an mechanism to automatically remediate which users, groups and machine identities have access each Fabric workspaces, ensuring alignment with a desired security / RBAC configuration stored in an external configuration file.

It uses the Semantic Link Labs python package (which are Python wrappers for Fabric APIs), and depends on Utils notebooks which provide function abstraction.


### Problem statement

Access to Workspaces is managed via authorised security groups, and developers are typically granted the `Member` role is used when provisioning developers with access to Workspaces. 

However, `Members` can also [add other entities to Workspaces](https://learn.microsoft.com/en-us/power-bi/collaborate-share/service-roles-new-workspaces#workspace-roles), including groups, machine identities or users, at equal or lower levels (_Member, Contributor or Viewer)_, which would violate the access control authorisation & approval policies. As the ability of `Members` to add entities cannot be disabled, it is necessary to enforce this policy via a control. 

![PowerBI Workspace add members menu](https://learn.microsoft.com/en-us/power-bi/collaborate-share/media/service-roles-new-workspaces/power-bi-roles-access.png)

To eliminate manual intervention required to implement this control, an automated solution was sought. 

> Q: "Why aren't you just using the _Contributor_ role?"
> A: Given the `Contributor` role in Fabric Workspaces has [restricted ability to create, manage and share Fabric assets](https://learn.microsoft.com/en-us/power-bi/collaborate-share/service-roles-new-workspaces#workspace-roles), and requires post-workspace creation configuration it is not widely used.

### Solution

Fabric Capacities allow the execution of arbitrary Python / PySpark through Fabric Notebooks. This functionality has been combined with the [Semantic Link Labs Python package](https://semantic-link-labs.readthedocs.io/en/stable/sempy_labs.html) (_a Python wrapper for Fabric APIs_) to extract, evaluate and rectify [Workspace permissions](https://learn.microsoft.com/en-us/power-bi/collaborate-share/service-roles-new-workspaces#workspace-roles) for Fabric Workspaces. 

By using a desired state configuration file, the tooling only performs access enforcement on listed Workspaces, rather than deprovisioning & re-provisioning across the entire tenant, allowing for a progressive roll-out

### Architecture Diagram

![Architecture-diagram](/notebooks/01-utils/fabric-workspace-membership-validation-util.svg)

### Integrations

This tooling relies on 

Application | Entity | Notes
:----------:|--------|---------
Azure DevOps | Configuration file `.csv` | master copy
Azure DevOps | Pipeline | **Trigger:** Changes to configuration file subdirectory on Main branch _(via Pull Request Completion)_<br>**Action:** upload (override) Configuration files into an Azure Storage Account<br>**Action:** trigger notebook execution
Azure Storage Account | Configuration file `.csv` | Created / Updated by DevOps pipeline
EntraID | Security group for Service Principals allowed to call EDIT Fabric Admin APIs | 
EntraID| Service Principal with <br>- Workspace access <br>- access to storage containers <br>- Ability to call public Fabric APIs | Provides the ability to run notebooks without being tied to a user identity
Azure Keyvault | Service Principal secrets  | Accessed via Keyvault queries within Fabric Notebooks
Microsoft Fabric | Fabric Capacity | 
Microsoft Fabric | Workspace(s) bound to Fabric Capacity | Also connected to Git & Deployment pipelines for CI/CD


### Deployment

Deployment across environments uses Git Integration for Development and Staging environments, followed by a Fabric Deployment Pipeline to promote from Staging to Production.

This allows for the use of Fabric Library variables, which can be toggled between value-sets via Fabric Deployment Pipelines, enabling us to re-point Fabric assets to different entities as the assets are promoted through each Environment.

For lower environments {`DEV`, `STG`}, the default set is used, which points to a "development mode" shortcut, which contains a subset of workspaces all owned by ITS, to enable lower-risk testing. Once promoted to `PRD`, the full set of managed workspaces is references.

### Networking

#### Private Networking

To access the keyvault where the Admin service principal is located, the Fabric Workspace where the notebook is running requires a **[Managed Private Endpoint](https://learn.microsoft.com/en-us/fabric/security/security-managed-private-endpoints-create)** connected to the target keyvault.

Additionally, the entity invoking a Fabric Pipeline / Notebook requires reader rights to the keyvault (_either passively via a Service Principal when executing in a pipeline, or activated PIM for users running Notebook directly_). 

For scheduled runs an orchestrator service principal is used, as Workspace Identities cannot be the executing identity of a notebook, but App Registrations can. This Orchestrator principal is a workspace member, and used in the Pipeline via Notebook Connectors. 

## Installation

1. Download the `admin-management-module-install.ipynb` notebook
0. Upload it to a Fabric capacity backed workspace
0. Fill any placeholder values if using Private Networking need MPEs, or delete that section
0. Run the notebook
0. Set up the Azure Storage Account and Container, Service Principals...
0. Set up the manual items, i.e. Environments and Variable Libraries referencing ^
0. Have the MPEs approved if needed
0. Populate the configuration files into the Storage Containers
0. Run the execution notebooks

## Operating notes

1. Service Principals must be added via their Enterprise Application `Object ID`, not the details of their parent App Registration.
2. Pipeline runs resulting in `200` series "_Livy Session_" errors which contained `401` "_Artefact not found_" errors were found to be the result of Notebook metadata containing references to deleted Lakehouses. Resolved by deleting the corresponding section from the notebook contents file.