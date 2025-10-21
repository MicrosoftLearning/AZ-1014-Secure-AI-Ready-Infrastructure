# AI Ready Security hands-on exercise: Configure Azure AI Foundry hub network connectivity by using a managed network

## Background information
Securing network connectivity for Azure AI Foundry hubs is essential for maintaining data confidentiality, preventing unauthorized access, and complying with enterprise security policies. Azure AI Foundry provides built-in support for network isolation, enabling you to control both inbound and outbound traffic to the hub and its associated resources. This isolation ensures that compute instances, managed endpoints, and other AI workloads operate within a tightly controlled network boundary while maintaining connectivity to approved Azure services.

### Network isolation concepts

When configuring a hub-based Azure AI Foundry environment, there are two primary aspects of network isolation to address:

1. Inbound access restrictions control how clients access the hub. By disabling public network access, you prevent connections from the public internet. Instead, you enable access through private endpoints, which provide secure, private connectivity from your designated virtual network (VNet) or from your on-premises environment connected through VPN or ExpressRoute. These private endpoints use Azure Private Link to map the hub’s public service endpoint to a private IP address within your VNet.

2. Outbound access restrictions control how resources within the hub (such as compute instances, managed online endpoints, and serverless jobs) access external services. Outbound isolation is achieved through a managed virtual network (Managed VNet) that Azure AI Foundry automatically creates and maintains for the hub and all associated projects. The managed VNet ensures compute resources operate in a restricted network environment and that outbound traffic is routed only to explicitly approved destinations.

### Managed virtual network architecture

When a managed virtual network is enabled, Azure automatically provisions and manages a dedicated virtual network for the hub. All compute resources you create—such as Azure Machine Learning Compute Clusters, serverless compute, and online endpoints—are automatically attached to this network. The managed VNet can securely connect to dependent Azure resources (such as Azure Storage, Azure Key Vault, and Azure Container Registry) through private endpoints, ensuring that sensitive data, models, and credentials never traverse the public internet.

### Outbound access modes

A managed virtual network supports three outbound access modes, which determine how outbound traffic is handled:

| Mode                         | Description                                                                                                                                  | Typical Scenario                                                                                                     |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| Allow internet outbound      | Permits unrestricted outbound traffic to the internet.                                                                                           | Use when your workloads need to access public repositories (e.g., Python packages, pretrained models, or open datasets). |
| Allow only approved outbound | Restricts traffic to explicitly defined destinations using service tags, private endpoints, or fully qualified domain names (FQDNs). | Use when minimizing data exfiltration risk is a priority and you operate within a controlled network environment.        |
| Disabled                     | Disables managed outbound controls, leaving inbound and outbound access unrestricted.                                                            | Use only in scenarios where full public connectivity is required and network isolation is unnecessary.                   |

In allow only approved outbound mode, outbound rules define which destinations the managed VNet can access. For instance:

* Private endpoints are used to reach Azure resources like Storage, Key Vault, or Container Registry.
* Service tags can be used to allow traffic to specific Azure services (e.g., `AzureMonitor`, `MicrosoftContainerRegistry`).
* Optional FQDN-based rules can be added for non-Azure dependencies.

   **Note**: Using FQDN-based outbound rules incurs Azure Firewall costs, as the managed VNet deploys a firewall to enforce these rules.

### Default and custom rules

Azure AI Foundry automatically preconfigures essential default outbound rules when a managed VNet is created. These rules ensure that core Azure services required for model training and deployment remain reachable. You can then add custom outbound rules to grant access to specific external endpoints or Azure services, depending on your organization’s needs. However, adding new rules should be done cautiously, as each additional allowed endpoint may expand your potential data exfiltration surface.

### Private connectivity for dependent resources

When network isolation is enabled, all dependent Azure services that the hub relies on are accessible through private endpoints. This includes:

* Azure Storage Account – Hosts data, artifacts, and model files.
* Azure Key Vault – Stores secrets, encryption keys, and credentials.
* Azure Container Registry (ACR) – Manages Docker images and environment containers.
* Azure AI Services (e.g., Azure OpenAI or Cognitive Services) – Used for model APIs and inference endpoints.

## Scenario
Your company operates in the financial services industry, where strict regulatory requirements and data protection policies govern how sensitive information is accessed and processed. The organization plans to build a centralized AI platform on Azure AI Foundry to support the development of machine learning models used for fraud detection, credit risk assessment, and transaction analysis. Because these workloads involve highly confidential customer and financial data, ensuring network isolation and controlled data movement is a top priority.

To meet internal security policies and industry compliance standards (such as ISO 27001 and SOC 2), the company plans to deploy the Azure AI Foundry hub within a managed virtual network. This managed network will isolate all AI resources from the public internet and enforce strict outbound connectivity rules to prevent accidental data exposure. Private endpoints will be used to connect securely to dependent Azure services, including Storage accounts, Key Vault, and the Azure Container Registry, while private DNS zones will ensure reliable name resolution within the isolated environment.

## Prerequisites
- **Azure subscription**: If you don't have an Azure subscription, [create a free account](https://azure.microsoft.com/free/) before you begin.
- **Permissions**: To create Azure AI Services resources, you should have the Owner role assigned at the Azure subscription.
- **Familiarity with Azure AI Foundry resource types**: To learn more, refer to [Choose an Azure resource type for AI foundry](https://learn.microsoft.com/en-us/azure/ai-foundry/concepts/resource-types).

## Estimated duration
25 minutes

**Note**: The exercise involves reviewing inbound and outbound access settings of an Azure AI hub without the actual implementation of its configuration options. This is intentional in order to minimize the duration and cost of the implementation. Azure AI Foundry defers creating the managed virtual network until a compute resource is created. With automatic creation, it can take about 30 minutes to create the first compute resource because it also provisions the network. In addition, adding outbound FQDN rules in the allow only approved outbound mode triggers deployment of Azure Firewall, with the corresponding charges added to your Azure subscription bill. 

### Task 1: Create a virtual network to host resources that will be part of the Azure AI hub environment

**Note**: The virtual network you are creating in this task is intended for connecting to Azure AI Foundry hub and for hosting resources that Azure AI Foundry hub compute resources will be connecting to.

1. Start a web browser, navigate to the Azure portal at [https://portal.azure.com](https://portal.azure.com) and sign in by providing the credentials of a user account which has the Owner role assigned at the Azure subscription level.
1. In the Azure portal, use the **Search** text box at the top of the page to search for **Virtual networks** and, in the list of results, select **Virtual networks**.
1. On the **Network foundation \| Virtual networks** page, with the **Virtual networks** entry selected in the vertical menu on the left side, select **+ Create**
1. On the **Basics** tab of the **Create virtual network** page, specify the following settings and select **Next**:

   |Setting|Value|
   |---|---|
   |Subscription|The name of the Azure subscription you are using in this lab|
   |Resource group|The name of a new resource group **aifoundry-isolated-RG**|
   |Virtual network name|**aifoundry-access-vnet**|
   |Region|The name of the Azure region where you want to deploy the Azure AI Foundry hub and its resources|

1. On the **Security** tab, accept the default settings and select **Next**.
1. On the **IP addresses** tab, apply the following settings (modify the default if needed):

   |Setting|Value|
   |---|---|
   |IP address space|**10.0.0.0/20**|

1. Select the edit (pencil) icon next to the **default** subnet entry, on the **Edit** pane, specify the following settings (leave others with their existing values) and select **Save**:

   |Setting|Value|
   |---|---|
   |Name|**aifoundry-access-subnet**|
   |Starting address|**10.0.0.0**|
   |Size|**/24 (256 addresses)**|
   |Enable private subnet (no default outbound access)|Disabled|

1. Back on the **IP addresses** tab, select **+ Add a subnet**.
1. On the **Add a subnet** pane, specify the following settings (leave others with their existing values) and select **Save**:

   |Setting|Value|
   |---|---|
   |Name|**aifoundry-private-endpoint-subnet**|
   |Starting address|**10.0.1.0**|
   |Size|**/24 (256 addresses)**|
   |Enable private subnet (no default outbound access)|Disabled|

1. Back on the **IP addresses** tab, select **Review + create** and then, on the **Review + create** tab, select **Create**.

    > **Note**: Wait for the provisioning process to complete. This should take less than 1 minute.

### Task 2: Create an Azure AI Services instance accessible via a private endpoint

1. In the web browser displaying the deployment page of the Azure virtual network, use the **Search** text box at the top of the page to search for **AI Foundry** and, in the list of results, select **AI Foundry**.
1. On the **AI Foundry** page, in the vertical menu on the left side, select **Use with AI Foundry** and then select **Azure AI Foundry**.
1. On the **AI Foundry \| Azure AI Foundry** page, select **+ Create**.
1. On the **Basics** tab of the **Create an Azure AI Foundry resource** page, specify the following settings (leave others with their default values) and select **Next**:

   |Setting|Value|
   |---|---|
   |Subscription|The name of the Azure subscription you are using in this exercise|
   |Resource group|**aifoundry-isolated-RG**|
   |Name|**aifoundry-isolated-resource**|
   |Region|The name of the Azure region where you created the virtual network earlier in this exercise|
   |Default project name|**aifoundry-isolated-project01**|

1. On the **Network** tab of the **Create an Azure AI Foundry resource** page, perform the following tasks:

- In the **Inbound access** section, select **Disabled, no network can access this resource. You could configure private endpoint connections that will be the exclusive way to access this resource**.
- In the **Private endpoint** section, select **+ Add Private Endpoint**.
- On the **Create private endpoint** pane, specify the following settings and select **OK**.

   |Setting|Value|
   |---|---|
   |Subscription|The name of the Azure subscription you are using in this lab|
   |Resource group|**aifoundry-isolated-RG**|
   |Location|The name of the Azure region where you created the virtual network earlier in this exercise|
   |Name|**aifoundry-resource-pe**|
   |Target sub-resource|**Cognitive Services**|
   |Virtual network|**aifoundry-access-vnet**|
   |Subnet|**aifoundry-private-endpoint-subnet**|
   |Integrate with private DNS zone|**Yes**|
   |Private DNS Zone|**privatelink.cognitiveservices.azure.com**|

1. Back on the **Networking** tab, select **Next**.
1. On the **Identity** tab, accept the default settings (with **System assigned** as the **Identity type**) and select **Next**.
1. On the **Encryption** tab, accept the default settings (encryption using Microsoft-managed keys) and select **Next**.
1. On the **Tags** tab, select **Next**.
1. On the **Review + create** tab, select **Create**.

   **Note**: Do not wait for the provisioning process to complete but instead proceed to the next task. The provisioning might take about 3 minutes.

### Task 3: Create an Azure Storage account accessible via a private endpoint

1. In the web browser displaying the deployment page of the Azure AI Services instance, use the **Search** text box at the top of the page to search for **Storage accounts** and, in the list of results, select **Storage accounts**.
1. On the **Storage accounts \| Storage accounts (Blobs)** page, select **+ Create**.
1. On the **Basics** tab of the **Create storage account** page, specify the following settings (leave others with their default values) and select **Next**:

   |Setting|Value|
   |---|---|
   |Subscription|The name of the Azure subscription you are using in this exercise|
   |Resource group|**aifoundry-isolated-RG**|
   |Storage account name|Any globally unique name between 3 and 24 characters in length consisting of lower case letters and digits, starting with a letter|
   |Region|The name of the Azure region where you created the virtual network earlier in this exercise|
   |Preferred storage type|**Azure Blob Storage or Azure Data Lake Storage Gen 2**|
   |Performance|**Standard**|
   |Redundancy|**Locally redundant storage (LRS)**|

   >**Note**: Record the storage account name. You'll need it later in this exercise.

1. On the **Advanced** tab, clear the **Enable storage account key access** checkbox and then select **Next**.
1. On the **Networking** tab, perform the following tasks:

- In the **Public network access** section, select **Disable**.
- In the **Private endpoint** section, select **+ Create a private endpoint**.
- On the **Create a private endpoint** pane, specify the following settings and select **OK**.

   |Setting|Value|
   |---|---|
   |Subscription|The name of the Azure subscription you are using in this lab|
   |Resource group|**aifoundry-isolated-RG**|
   |Region|The name of the Azure region where you created the virtual network earlier in this exercise|
   |Name|**aifoundry-sa-pe**|
   |Target sub-resource|**blob**|
   |Virtual network|**aifoundry-access-vnet**|
   |Subnet|**aifoundry-private-endpoint-subnet**|
   |Integrate with private DNS zone|**Yes**|
   |Private DNS Zone|**(New) privatelink.blob.core.windows.net**|

1. Back on the **Networking** tab, select **Review + create**.
1. Once you selected **Review + create**, wait for the validation to complete and then select **Create**.

   **Note**: Do not wait for the provisioning process to complete but instead proceed to the next task. The provisioning might take about 3 minutes.

### Task 4: Create an Azure Key Vault instance accessible via a private endpoint

1. In the web browser displaying the deployment page of the Azure Storage account, use the **Search** text box at the top of the page to search for **Key vaults** and, in the list of results, select **Key vaults**.
1. On the **Key vaults** page, select **+ Create**.
1. On the **Basics** tab of the **Create a key vault** page, specify the following settings (leave others with their default values) and select **Next**:

   |Setting|Value|
   |---|---|
   |Subscription|The name of the Azure subscription you are using in this exercise|
   |Resource group|**aifoundry-isolated-RG**|
   |Key vault name|Any unique name between 3 and 24 alphanumeric characters in length, starting with a letter|
   |Region|The name of the Azure region where you created the virtual network earlier in this exercise|
   |Pricing tier|**Standard**|

1. On the **Access configuration** tab of the **Create a key vault** page, ensure that the **Permission model** is set to **Azure role-based access control** and select **Next**:
1. On the **Networking** tab, perform the following tasks:

- Disable the **Enable public access** checkbox.
- In the **Private endpoint** section, select **+ Create a private endpoint**.
- On the **Create a private endpoint** pane, specify the following settings and select **OK**.

   |Setting|Value|
   |---|---|
   |Subscription|The name of the Azure subscription you are using in this lab|
   |Resource group|**aifoundry-isolated-RG**|
   |Region|The name of the Azure region where you created the virtual network earlier in this exercise|
   |Name|**aifoundry-kv-pe**|
   |Target sub-resource|**Vault**|
   |Virtual network|**aifoundry-access-vnet**|
   |Subnet|**aifoundry-private-endpoint-subnet**|
   |Integrate with private DNS zone|**Yes**|
   |Private DNS Zone|**(New) privatelink.vaultcore.azure.net**|

1. Back on the **Networking** tab, select **Review + create**.
1. Once you selected **Review + create**, wait for the validation to complete and then select **Create**.

   **Note**: Do not wait for the provisioning process to complete but instead proceed to the next task. The provisioning might take about 3 minutes.

### Task 5: Create an Azure Container Registry instance accessible via a private endpoint

1. In the web browser displaying the Azure Key Vault deployment page, use the **Search** text box at the top of the page to search for **Container registries** and, in the list of results, select **Container registries**.
1. On the **Container registries** page, select **+ Create**.
1. On the **Basics** tab of the **Create container registry** page, specify the following settings (leave others with their default values) and select **Next: Networking**:

   |Setting|Value|
   |---|---|
   |Subscription|The name of the Azure subscription you are using in this exercise|
   |Resource group|**aifoundry-isolated-RG**|
   |Registry name|Any unique name between 5 and 50 alphanumeric characters in length, starting with a letter|
   |Region|The name of the Azure region where you created the virtual network earlier in this exercise|
   |Pricing tier|**Premium**|

   **Note**: The Premium pricing tier is required to support private access.

1. On the **Networking** tab, perform the following tasks:
 
- In the **Connectivity configuration** section, select **Private access (Recommended)**.
- Select **+ Create a private endpoint connection**.
- On the **Create a private endpoint** pane, specify the following settings and select **OK**.

   |Setting|Value|
   |---|---|
   |Subscription|The name of the Azure subscription you are using in this lab|
   |Resource group|**aifoundry-isolated-RG**|
   |Region|The name of the Azure region where you created the virtual network earlier in this exercise|
   |Name|**aifoundry-cr-pe**|
   |Target sub-resource|**registry**|
   |Virtual network|**aifoundry-access-vnet**|
   |Subnet|**aifoundry-private-endpoint-subnet**|
   |Integrate with private DNS zone|**Yes**|
   |Private DNS Zone|**(New) privatelink.azurecr.io**|

1. Back on the **Networking** tab, select **Review + create**.
1. Once you selected **Review + create**, wait for the validation to complete and then select **Create**.

   **Note**: Wait for the provisioning process to complete. This might take about 2 minutes.

### Task 6: Review inbound access configuration of an Azure AI hub

1. In the web browser displaying the Azure Container Registry instance deployment page, use the **Search** text box at the top of the page to search for **AI Foundry** and, in the list of results, select **AI Foundry**.
1. On the **AI Foundry** page, in the vertical menu on the left side, select **Use with AI Foundry** and then select **AI Hubs**.
1. On the **AI Foundry \| AI Hubs** page, select **+ Create** and, in the drop-down menu, select **Hub**.
1. On the **Basics** tab of the **Azure AI hub** page, specify the following settings (leave others with their default values) and select **Next: Storage**:

   |Setting|Value|
   |---|---|
   |Subscription|The name of the Azure subscription you are using in this exercise|
   |Resource group|**aifoundry-isolated-RG**|
   |Region|The name of the Azure region where you intend to create Azure AI Foundry hub|
   |Name|**aifoundry-isolated-hub**|
   |Friendly name|**Isolated hub**|
   |Default project resource group|**Same as hub resource group**|
   |Connect AI Services incl. OpenAI|**aifoundry-isolated-resource**|

1. On the **Storage** tab of the **Azure AI hub** page, specify the following settings and select **Next: Inbound Access**:

   |Setting|Value|
   |---|---|
   |Storage account|The name of the storage account you provisioned earlier in this exercise|
   |Credential store|**Azure key vault**|
   |Key vault|The name of the Key Vault instance you provisioned earlier in this exercise|
   |Application insights|**None**|
   |Container registry|The name of the Azure Container Registry instance you provisioned earlier in this exercise|

1. On the **Inbound Access** tab, perform the following tasks:

- Set the **Public network access** option to **Disabled**.
- In the **Workspace inbound access** section, select **+ Add**.
- On the **Create private endpoint** pane, specify the following settings and select **OK**.

   |Setting|Value|
   |---|---|
   |Subscription|The name of the Azure subscription you are using in this lab|
   |Resource group|**aifoundry-isolated-RG**|
   |Region|The name of the Azure region where you created the virtual network earlier in this exercise|
   |Name|**aifoundry-hub-pe**|
   |Target sub-resource|**azuremlworkspace**|
   |Virtual network|**aifoundry-access-vnet**|
   |Subnet|**aifoundry-private-endpoint-subnet**|
   |Integrate with private DNS zone|**Yes**|
   |Configuration name|**privatelink-api-azureml-ms**|
   |Resource group|**aifoundry-isolated-RG**|
   |Private DNS zone|**(new) privatelink.api.azureml.ms**|
   |Configuration name|**privatelink-notebooks-azure-net**|
   |Resource group|**aifoundry-isolated-RG**|
   |Private DNS zone|**(new) privatelink.notebooks.azure.net**|

1. Back on the **Inbound access** tab, select **Next: Outbound Access**.
1. On the **Outbound Access** tab, first select the **Allow Internet Outbound** option.
1. On the **Outbound Access** tab, in the **Workspace Outbound access** section, select the **Provision managed virtual network** checkbox and then expand the **Required outbound rules** section.
1. In the **Required outbound rules** section, review the listing of the outbound rules and note that they target the Azure resources you provisioned earlier in this exercise and configured to be accessible via private endpoints.
1. While on the **Outbound Access** tab, next select the **Allow Only Approved Outbound** option.
1. On the **Outbound Access** tab, in the **Workspace Outbound access** section, note the additional setting which allows you to choose between the **Standard** and **Basic** options of the **Azure Firewall SKU**. 
1. Expand the **Required outbound rules** section and review again the listing of the outbound rules. Note that this time they target not only the Azure resources you provisioned earlier in this exercise and configured to be accessible via private endpoints, but also include connectivity to a number of other endpoints designated by using **ServiceTag** destination type.
1. On the **Outbound Access** tab, select **Next: Encryption**.
1. On the **Encryption** tab, accept the default settings and select **Next: Identity**.
1. On the **Identity** tab, accept the default settings (with **System assigned identity** as the **Identity type** and **Credential-based access** as the **Storage account access type**) and select **Review + create**.
1. Once you selected **Review + create**, wait for the validation to complete but **do not** select **Create**.

**Note**: As mentioned earlier, this is intended in order to minimize the duration and cost of this exercise. Azure AI Foundry defers creating the managed virtual network until a compute resource is created. With automatic creation, it can take about 30 minutes to create the first compute resource because it also provisions the network. In addition, adding outbound FQDN rules in the allow only approved outbound mode triggers deployment of Azure Firewall, with the corresponding charges added to your Azure subscription bill. 

### Task 7: Perform cleanup

1. Open another tab in the web browser displaying the Azure AI Foundry portal, navigate to the Azure portal at [https://portal.azure.com](https://portal.azure.com) and, if prompted, sign in by providing the same credentials you have been using throughout this exercise.
1. In the Azure portal, use the **Search** text box at the top of the page to search for **aifoundry-isolated-RG** and, in the list of results, select **aifoundry-isolated-RG**.
1. On the **aifoundry-isolated-RG** page, select **Delete resource group**, on the **Delete a resource group** pane, in the **Enter resource group name to confirm deletion** text box, enter **aifoundry-isolated-RG**, select **Delete**, and, when prompted for confirmation, select **Delete** again.