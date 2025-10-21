# AI Ready Security hands-on exercise: Implement customer-managed keys (CMK) for AI services

## Background information
All Azure AI Foundry resources are encrypted at rest by default with Microsoft-managed keys. Organizations that must control the lifecycle of their encryption keys can instead enable customer-managed keys (CMKs). With CMKs, encryption and decryption operations use a key stored in Azure Key Vault, allowing the organization to define its own key-creation, rotation, and deletion policies and to satisfy compliance frameworks such as ISO 27001 or SOC 2.

### Managed identity access
Azure AI Foundry accesses the Key Vault key through a managed identity—either system-assigned (tied to the Foundry resource) or user-assigned (centrally managed or shared). This identity removes the need for stored credentials and is granted only the minimal permissions required to perform encryption:
- get  
- wrapKey  
- unwrapKey  
The Key Vault Crypto Service Encryption User built-in role provides exactly these rights.

### Key Vault configuration
To use CMKs:
- The Key Vault and Foundry hub must be in the same region and tenant.  
- The vault must have purge protection enabled (soft-delete is enforced on all newly created vaults).
- Only RSA or RSA-HSM 2048-bit keys are supported.  
- Network isolation can be maintained through Private Link or by allowing trusted Microsoft services to reach the vault.

### Key rotation and management
Key rotation can be manual or policy-based in Key Vault. When a new key version is created, the Foundry resource must be updated to reference it. Rotation does not re-encrypt existing data, and CMK-enabled resources cannot revert to Microsoft-managed keys.

### Benefits
Using CMKs strengthens control over encryption, enables auditability through Key Vault logging, and aligns AI Foundry deployments with enterprise and regulatory security requirements.

## Scenario
Your company is a financial services firm that handles highly sensitive customer and transactional data, including banking records, payment histories, credit assessments, and fraud detection metrics. It provides services such as credit risk evaluation, fraud monitoring, transaction analysis, and personalized financial recommendations. Given the sensitive nature of its operations, maintaining strict control over data encryption is essential for compliance with internal security policies and external regulatory requirements.

Your company plans to build a centralized AI platform on Azure AI Foundry to support these operations. To ensure full control over encryption, the organization intends to use customer-managed keys, allowing it to enforce key rotation policies and maintain auditability of key usage. System-assigned managed identities will be used for the Azure AI Foundry resources to securely access the Key Vault storing the CMKs, reducing administrative overhead and minimizing the risk of accidental key exposure. This approach allows the company to protect sensitive data while leveraging AI workloads without modifying application code.

## Prerequisites
- **Azure subscription**: If you don't have an Azure subscription, [create a free account](https://azure.microsoft.com/free/) before you begin.
- **Permissions**: To create Azure AI Services resources, you should have the Owner role assigned at the Azure subscription.
- **Familiarity with Azure AI Foundry resource types**: To learn more, refer to [Choose an Azure resource type for AI foundry](https://learn.microsoft.com/en-us/azure/ai-foundry/concepts/resource-types).

## Estimated duration
25 minutes

### Task 1: Create an Azure Key Vault instance

1. Start a web browser, navigate to the Azure portal at [https://portal.azure.com](https://portal.azure.com) and sign in by providing the credentials of a user account which has the Owner role assigned at the Azure subscription level.
1. In the Azure portal, use the **Search** text box at the top of the page to search for **Key vaults** and, in the list of results, select **Key vaults**.
1. On the **Key vaults** page, select **+ Create**.
1. On the **Basics** tab of the **Create a key vault** page, specify the following settings (leave others with their default values) and select **Next**:

   |Setting|Value|
   |---|---|
   |Subscription|The name of the Azure subscription you are using in this exercise|
   |Resource group|**aifoundry-cmkencryption-RG**|
   |Key vault name|Any unique name between 3 and 24 alphanumeric characters in length, starting with a letter|
   |Region|The name of the Azure region where you intend to create the Azure AI Foundry resource|
   |Pricing tier|**Standard**|
   |Soft-delete|**Enabled**|
   |Days to retain deleted vaults|**7**|
   |Purge protection|**Enable purge protection (enforce a mandatory retention period for deleted vaults and vault objects)**|

   **Note**: The value of **Days ot retain deleted vaults** is set to the minimum (7 days) strictly to facilitate the use of your lab environment. In real-world scenarios, you should consider increasing this value (depending on your preferences or needs).

   **Note**: Purge protection cannot be disabled (once enabled).

1. On the **Access configuration** tab of the **Create a key vault** page, ensure that the **Permission model** is set to **Azure role-based access control** and select **Next**:
1. On the **Networking** tab, ensure that the **Enable public access** checkbox and **All networks** option are enabled, and then select **Review + create**.

   **Note**: Public access is enabled strictly for the sake of simplicity. In real-world scenarios, you should consider disabling it and allowing connectivity exclusively from the networks you designate. In such cases, you need to also enable the setting **Allow trusted Microsoft services to bypass this firewall**.

1. Once you selected **Review + create**, wait for the validation to complete and then select **Create**.

   **Note**: Wait for the provisioning process to complete. This should take about 1 minute.

### Task 2: Generate an encryption key in the Azure Key Vault instance

1. In the web browser displaying the deployment page of the Azure Key Vault instance, select **Go to resource**.
1. On the Azure Key Vault **Overview** page, in the vertical menu on the left side, select **Access control (IAM)**.
1. On the **Access control (IAM)** page, select **+ Add** and, in the drop-down menu, select **Add role assignment**.
1. On the **Role** tab of the **Add role assignment** page, under the **Job function roles** listing, in the **Search** text box, enter **Key Vault Administrator**, in the list of search results, select **Key Vault Administrator**, and select **Next**.
1. On the **Members** tab of the **Add role assignment** page, in the **Assign access to** section, ensure that the **User, group, or service principal** option is selected and then click **+ Select members**. 
1. On the **Select members** pane, in the search text box, enter your user account name, select it in the list of results, and then click **Select**.
1. Back on the **Members** tab of the **Add role assignment** page, select **Next**.
1. On the **Conditions** tab of the **Add role assignment** page, select **Review + assign**.
1. On the **Review + assign** tab, select **Review + assign**.
1. Back on the Azure Key Vault **Access control (IAM)** page, in the vertical menu on the left side, in the **Objects** section, select **Keys**.
1. On the Azure Key Vault **Keys** page, select **+ Generate/Import**.
1. On the **Create a key** page, perform the following actions:

- Specify the following settings (leave others with their default values) and select **Create**:

   |Setting|Value|
   |---|---|
   |Options|**Generate**|
   |Name|**aifoundry-encryption-key**|
   |Key type|**RSA**|
   |RSA key size|**2048**|
   |Enabled|**Yes**|

- Next to the **Set key rotation policy** entry, select **Not configured**.
- On the **Rotation policy** pane, in the **Expiry time** text box, enter **210** and verify that **days** appears in the drop-down list next to it.
- On the **Rotation policy** pane, in the **Rotation** section, set **Enable auto rotation** to **Enabled**, verify that the **Rotation option** is set to **Automatically renew at a given time after creation**, set **Rotation time** to **6** **months** and then select **OK**.

   **Note**: The rotation time defines when a new version of a key is automatically generated, while the expiry time specifies how long that specific version remains valid before it becomes unusable. Effectively, rotation time determines the interval or trigger for renewal (for example, every 60 days), ensuring that credentials are refreshed regularly, whereas expiry time defines the lifetime of each version (for example, expires after 90 days). 

- Back on the **Create a key** page, select **Create**.

### Task 3: Create an Azure AI Foundry resource

1. In the web browser displaying the **Keys** page of the Azure Key Vault instance, use the **Search** text box at the top of the page to search for **AI Foundry** and, in the list of results, select **AI Foundry**.
1. On the **AI Foundry** page, in the vertical menu on the left side, select **Use with AI Foundry** and then select **Azure AI Foundry**.
1. On the **AI Foundry \| Azure AI Foundry** page, select **+ Create**.
1. On the **Basics** tab of the **Create an Azure AI Foundry resource** page, specify the following settings (leave others with their default values) and select **Next**:

   |Setting|Value|
   |---|---|
   |Subscription|The name of the Azure subscription you are using in this exercise|
   |Resource group|**aifoundry-cmkencryption-RG**|
   |Name|**aifoundry-cmkencryption-resource**|
   |Region|The name of the Azure region where you created the virtual network earlier in this exercise|
   |Default project name|**aifoundry-cmkencryption-project01**|

1. On the **Network** tab of the **Create an Azure AI Foundry resource** page, ensure that the option **All networks, including the internet can access this resource** is selected and then select **Next**.
1. On the **Identity** tab, accept the default settings (with **System assigned** as the **Identity type**) and select **Next**.
1. On the **Encryption** tab, accept the default settings (encryption using Microsoft-managed keys) and select **Next**.

   **Note**: You will use the managed identity of the Azure AI Foundry resource to access the key you generated in Azure Key Vault, so you have to create the resource first with encryption using a Microsoft-managed key.

1. On the **Tags** tab, select **Next**.
1. On the **Review + create** tab, select **Create**.

   **Note**: Wait for the provisioning process to complete. This should take about 1 minute.

### Task 4: Grant Azure Key Vault access to the Azure AI Foundry managed identity

1. In the web browser displaying the deployment page of the Azure Storage account, use the **Search** text box at the top of the page to search for **Key vaults** and, in the list of results, select **Key vaults**.
1. On the **Key vaults** page, select the entry representing the Azure Key Vault instance you created earlier in this exercise.
1. On the Azure Key Vault **Overview** page, in the vertical menu on the left side, in the **Objects** section, select **Keys**.
1. On the Azure Key Vault **Keys** page, select **aifoundry-encryption-key**.
1. On the **aifoundry-encryption-key** page, in the vertical menu on the left side, select **Access control (IAM)**.
1. On the **Access control (IAM)** page, select **+ Add** and, in the drop-down menu, select **Add role assignment**.
1. On the **Role** tab of the **Add role assignment** page, under the **Job function roles** listing, in the **Search** text box, enter **Key Vault Crypto Service Encryption User**, in the list of search results, select **Key Vault Crypto Service Encryption User**, and select **Next**.

   **Note**: Azure AI Foundry resource system-assigned managed identity requires **get**, **wrapKey**, **unwrapKey** key permissions. The **Key Vault Crypto Service Encryption User** role is the least-privileged built-in role that grants these permissions.

1. On the **Members** tab of the **Add role assignment** page, in the **Assign access to** section, select **Managed identity** and then click **+ Select members**. 
1. On the **Select members** pane, in the **Managed identity** drop-down list, select **Azure AI Foundry** entry, in the list of search results, select **aifoundry-cmkencryption-resource**, and then click **Select**.
1. Back on the **Members** tab of the **Add role assignment** page, select **Next**.
1. On the **Conditions** tab of the **Add role assignment** page, select **Review + assign**.
1. On the **Review + assign** tab, select **Review + assign**.

### Task 5: Configure the Azure AI Foundry resource to use the customer-managed encryption key

1. In the web browser displaying the **aifoundry-encryption-key** page, use the **Search** text box at the top of the page to search for **AI Foundry** and, in the list of results, select **AI Foundry**.
1. On the **AI Foundry** page, in the vertical menu on the left side, select **Use with AI Foundry** and then select **Azure AI Foundry**.
1. On the **AI Foundry \| Azure AI Foundry** page, select **aifoundry-cmkencryption-resource**.
1. On the **aifoundry-cmkencryption-resource** page, in the **Resource Management** section, select **Encryption**.
1. On the **aifoundry-cmkencryption-resource \| Encryption** page, set the **Encryption type** option to **Customer Managed Keys**, set they **Encryption key** option to **Select from Key Vault** and then click **Select a KeyVault and Key for encryption**.
1. On the **Select a key** page, ensure that the name of the Azure subscription you are using in this exercise appears in the **Subscription** drop-down list, and then set the **Key store type** to **Key vault**.
1. In the **Key vault** drop-down list, select the Azure Key Vault instance you created earlier in this exercise, in the **Key** drop-down list, select **aifoundry-encryption-key** and then click **Select**.
1. Back on the **aifoundry-cmkencryption-resource \| Encryption** page, select **Save**.

### Task 6: Perform cleanup

1. Open another tab in the web browser displaying the Azure AI Foundry portal, navigate to the Azure portal at [https://portal.azure.com](https://portal.azure.com) and, if prompted, sign in by providing the same credentials you have been using throughout this exercise.
1. In the Azure portal, use the **Search** text box at the top of the page to search for **aifoundry-cmkencryption-RG** and, in the list of results, select **aifoundry-cmkencryption-RG**.
1. On the **aifoundry-cmkencryption-RG** page, select **Delete resource group**, on the **Delete a resource group** pane, in the **Enter resource group name to confirm deletion** text box, enter **aifoundry-cmkencryption-RG**, select **Delete**, and, when prompted for confirmation, select **Delete** again.