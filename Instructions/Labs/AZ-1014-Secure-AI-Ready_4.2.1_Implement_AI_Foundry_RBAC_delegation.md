# AI Ready Security hands-on exercise: Configure authorization for Azure AI Foundry project

## Background information
Access management for cloud resources is a critical function for any organization that is using the cloud. Azure role-based access control (Azure RBAC) helps manage who has access to Azure resources, what they can do with those resources, and what areas they have access to. Azure RBAC is an authorization system built on Azure Resource Manager that provides fine-grained access management to Azure resources.

The way access is controlled is through role assignments, which define how permissions are enforced. A role assignment consists of three elements: security principal, role definition, and scope.
- Security principal: An object that represents a user, group, service principal, or managed identity requesting access to Azure resources. 
- Role definition: A collection of permissions (or "role") that defines which actions can be performed, such as read, write, or delete. Azure includes many built-in roles. You can also create custom roles tailored to your specific needs.
- Scope: The set of resources that the access applies to. Roles can be assigned at the management group, subscription, resource group, or resource level. Scopes form a hierarchy, and permissions assigned at a higher scope automatically apply to all lower scopes.

It's important to note that Azure RBAC is an allow-only and cumulative authorization model. This means that permissions are additive: if a user or identity has multiple role assignments, the effective permissions are the union of all assigned roles. Likewise, access granted at a higher scope (for example, subscription) applies to all lower scopes (such as resource groups and individual resources) within that hierarchy.

Within Azure AI Foundry, RBAC provides consistent, role-based authorization across both the account and its projects.
- The account hosts the shared infrastructure (such as virtual networks, customer-managed keys, managed identities, and policies).
- A project is a workspace within the account where AI assets—such as datasets, models, prompts, deployments, and experiments—are organized, developed, and managed. Projects serve as the primary collaboration and lifecycle management areas for building, fine-tuning, evaluating, and deploying AI solutions.

Azure AI Foundry includes built-in roles that govern access at both levels:
- Azure AI User – Grants read-only access to Foundry resources and projects, including permissions to view AI assets and perform limited data actions.
- Azure AI Project Manager – Allows management of project-level assets and resources, including building, developing, and assigning roles to other users within the project scope.
- Azure AI Account Owner – Provides full control over both Foundry resources and projects, including configuration, governance, and conditional role assignment capabilities.

These roles combine control-plane actions (for managing Foundry resources and projects) with data-plane permissions (for accessing AI assets such as models, datasets, and deployments). Managed identities used by Azure AI Foundry resources and projects can also be granted RBAC permissions to access dependent Azure services like Key Vault or Storage, enabling secure, keyless access using Microsoft Entra ID authentication.

## Scenario
Your company is a global financial analytics firm that develops AI-driven models to assess credit risk, detect fraud, and provide personalized financial insights for its clients. To support these workloads, the company will build a centralized AI development platform using Azure AI Foundry, enabling multiple teams to collaborate securely while maintaining strict access boundaries between projects.

To maintain strong governance and data protection controls, the company will implement Azure role-based access control (RBAC) to manage who can view, build, or manage AI assets. By leveraging built-in roles such as Azure AI Account Owner, Azure AI Project Manager, and Azure AI User, the company will be able to ensure that each team member has access appropriate to their responsibilities while adhering to the principle of least privilege. Because RBAC permissions are allow-only and cumulative, any access granted at the resource level will automatically apply to all projects created within the same Azure AI Foundry resource.

Within this model, one team member will serve as the Project Manager for the default project, with full management capabilities, including the ability to build and develop AI assets and manage access for other users. The same individual will hold only the Azure AI User role in other projects under the same Foundry resource, allowing visibility into shared data and configurations without the ability to modify them. This separation of duties will ensure clear accountability, prevent unauthorized changes, and provide a secure foundation for collaborative AI development across teams.

## Prerequisites
- **Azure subscription**: If you don't have an Azure subscription, [create a free account](https://azure.microsoft.com/free/) before you begin.
- **Permissions**: To create Azure AI Services resources, you should have the Owner role assigned at the Azure subscription.
- **Familiarity with Azure AI Foundry resource types**: To learn more, refer to [Choose an Azure resource type for AI foundry](https://learn.microsoft.com/en-us/azure/ai-foundry/concepts/resource-types).
- **Familiarity with Azure role-based access control (RBAC)**: To learn more, refer to [What is Azure role-based access control (Azure RBAC)?](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview)

## Estimated duration
10 minutes

### Task 1: Create Azure AI Foundry resource and the default project

1. Start a web browser, navigate to the Azure portal at [https://portal.azure.com](https://portal.azure.com) and sign in by providing the credentials of a user account which has the Owner role assigned at the Azure subscription level.
1. In the Azure portal, use the **Search** text box at the top of the page to search for **AI Foundry** and, in the list of results, select **AI Foundry**.
1. On the **AI Foundry** page, in the vertical menu on the left side, select **Use with AI Foundry** and then select **Azure AI Foundry**.
1. On the **AI Foundry \| Azure AI Foundry** page, select **+ Create**.
1. On the **Basics** tab of the **Create an Azure AI Foundry resource** page, specify the following settings (leave others with their default values) and select **Next**:

   |Setting|Value|
   |---|---|
   |Subscription|The name of the Azure subscription you are using in this exercise|
   |Resource group|The name of a new resource group **rbac-project-RG**|
   |Name|**rbac-project-resource**|
   |Region|The name of the Azure region where you intend to create Azure AI Foundry project|
   |Default project name|**rbacproject01**|

1. On the **Network** tab of the **Create an Azure AI Foundry resource** page, ensure that the option **All networks, including the internet can access this resource** is selected and then select **Next**.
1. On the **Identity** tab, accept the default settings (with **System assigned** as the **Identity type**) and select **Next**.
1. On the **Encryption** tab, accept the default settings (encryption using Microsoft-managed keys) and select **Next**.
1. On the **Tags** tab, select **Next**.
1. On the **Review + create** tab, select **Create**.

   **Note**: Wait until the resource and project are provisioned. This might take about one minute. 

### Task 2: Create an additional Azure AI Foundry project

1. In the web browser displaying the Azure AI Foundry resource's deployment status page in the Azure portal, select **Go to resource** to navigate to the project's **Overview** page.
1. On the **rbac-project-resource \| Overview** page, in the vertical menu on the left side, in the **Resource Management** section, select **project**.
1. On the **rbac-project-resource \| project** page, select **+ New**.
1. On the **New project** pane, specify the following settings and then select **Create**:

   |Setting|Value|
   |---|---|
   |Name|**rbacproject02**|
   |Description|Non-default project 02|

   **Note**: Wait until the project is provisioned. This might take about one minute. 

### Task 3: Review the default RBAC assignments

1. In the web browser displaying the deployment page of the Azure AI Foundry project, use the **Search** text box at the top of the page to search for **AI Foundry** and, in the list of results, select **AI Foundry**.
1. On the **AI Foundry** page, in the vertical menu on the left side, select **Use with AI Foundry** and then select **Azure AI Foundry**.
1. On the **AI Foundry \| Azure AI Foundry** page, in the list of Azure AI Foundry resources, select **rbac-project-resource**.
1. On the **rbac-project-resource** page, in the vertical menu on the left side, select **Access control (IAM)**.
1. On the **rbac-project-resource \| Access control (IAM)** page, select the **Role assignments** tab, review the list of assignments, and note that your user account has both the **Azure AI User** and **Owner** role (inherited from the subscription level). The former, provides reader-level access on the data plane of the Azure AI Foundry resource and its projects. The latter grants you full access on the control plane, allowing you to manage access on both control and data plane.

   **Note**: To view specific actions and data actions associated with the **Azure AI User** role, select any of the **Azure AI User** links appearing next to your user account. This will open the **Azure AI User** page with the **Permissions** tab displaying the list of actions in the **Permission type** section. To view the list of data actions, select the **DataActions** button.

1. On the **rbac-project-resource \| Access control (IAM)** page, in the vertical menu on the left side, in the **Resource Management** section, select **Projects**.
1. On the **rbac-project-resource \| Projects** page, in the list of projects, select **rbacproject01**. 
1. On the **rbacproject01** page, in the vertical menu on the left side, select **Access control (IAM)**.
1. On the **rbacproject01 \| Access control (IAM)** page, select the **Role assignments** tab, review the list of assignments, and note that your user account has the same level of permissions on the project level as well (inherited from the subscription level).

### Task 4: Modify the default RBAC assignments

1. In the web browser displaying the **rbacproject01 \| Access control (IAM)** page, select **+ Add** and, in the drop-down menu, select **Add role assignment**.
1. On the **Role** tab of the **Add role assignment** page, under the **Job function roles** listing, in the **Search** text box, enter **Azure AI Project Manager**, in the list of search results, select **Azure AI Project Manager**, and select **Next**.

   **Note**: The **Azure AI Project Manager** grants the ability to build and develop in a project (data action), which is not part of the built-in **Owner** role assignment. 

1. On the **Members** tab of the **Add role assignment** page, in the **Assign access to** section, ensure that the **User, group, or service principal** is  selected and then click **+ Select members**. 
1. On the **Select members** pane, in the **Search** box enter your Microsoft Entra ID user name, select it in the list of search results, and then click **Select**.
1. Back on the **Members** tab of the **Add role assignment** page, select **Next**.
1. On the **Conditions** tab of the **Add role assignment** page, select **Review + assign**.
1. On the **Review + assign** tab, select **Review + assign**.
1. Back on the **rbacproject01 \| Access control (IAM)** page, review the list of assignments and verify that the assignments was successfully created.
1. In the web browser displaying the **rbacproject01 \| Access control (IAM)** page, navigate back to the **rbac-project-resource \| Projects** page and, in the list of projects, select **rbacproject02**. 
1. On the **rbacproject02** page, in the vertical menu on the left side, select **Access control (IAM)**.
1. On the **rbacproject02 \| Access control (IAM)** page, select the **Role assignments** tab, review the list of assignments, and note that your user account has the same level of permissions as those on the resource level (inherited from the subscription level), but the role assignment you created on the **rbacproject01** level did not affect in any way access to **rbacproject02**.

### Task 5: Perform cleanup

1. In the web browser displaying the Azure portal use the **Search** text box at the top of the page to search for **rbac-projects-RG** and, in the list of results, select **rbac-projects-RG**.
1. On the **rbac-projects-RG** page, select **Delete resource group**, on the **Delete a resource group** pane, in the **Enter resource group name to confirm deletion** text box, enter **rbac-projects-RG**, select **Delete**, and, when prompted for confirmation, select **Delete** again.