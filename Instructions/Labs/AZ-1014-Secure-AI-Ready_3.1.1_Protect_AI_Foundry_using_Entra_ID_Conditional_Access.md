# AI Ready Security hands-on exercise: Protect Azure AI by using Microsoft Entra ID Conditional Access

## Background information
Modern security extends beyond the network perimeter and relies on identity-driven access control. Microsoft Entra Conditional Access evaluates multiple signals to make access decisions for users, devices, and applications. Signals include, for example, user's identity and group membership, device state and compliance, IP location, target application, and real-time risk assessment. Policies can combine these signals to implement complex access rules, such as requiring multifactor authentication in combination with restricting connections from trusted networks only or enforcing device's Entra hybrid join status when accessing sensitive apps.

Policies are applied after first-factor authentication and help organizations implement a Zero Trust model, reducing the risk of unauthorized access to sensitive resources. However, they can be set in report-only mode, allowing administrators to evaluate their impact before actually enforcing them.

Conditional Access policies can be applied to protect Azure AI Foundry resources. For example, administrators can assign policies to the following apps by their App IDs:
- Azure AI Studio App (cb2ff863-7f30-4ced-ab89-a00194bcf6d9) – controls access to the Azure AI Foundry portal
- Azure Machine Learning Web App (d7304df8-741f-47d3-9bc2-df0e24e2071f) – controls access to Azure ML studio
- Azure Machine Learning API (0736f41a-0425-bdb5-1563eff02385) – governs direct API access, such as SDK or REST API calls (hub-based projects rely on this API)

By leveraging these app IDs in Conditional Access assignments, administrators can target policies specifically to the Azure AI workloads, ensuring only authorized users using compliant devices from secure locations can access sensitive AI environments. This combination of signals and targeted enforcement enables organizations to maintain regulatory compliance, monitor access risk in real time, and protect critical AI workloads.

## Scenario
Your company is a financial services firm that handles highly sensitive customer and transactional data, including banking records, payment histories, credit assessments, and fraud detection metrics. To support its AI initiatives, the organization plans to deploy a centralized Azure AI Foundry platform, enabling teams to develop machine learning models for fraud detection, credit risk assessment, and transaction analysis. Given the sensitivity of these workloads, controlling access to the Azure AI Foundry portal is critical to maintain regulatory compliance and protect customer data.

To implement this, the company plans to use Microsoft Entra Conditional Access policies. Access will be restricted to a designated security group of users, ensuring only authorized personnel can reach the portal. Additional access controls will require multifactor authentication and devices to be Entra hybrid joined. Furthermore, access will be allowed exclusively from the company’s HQ IP range, blocking logins from untrusted networks. To validate these controls without disrupting business operations, the policies will initially be deployed in report-only mode, allowing administrators to monitor the effects before enforcing them. In addition, any users outside the designated group will be explicitly blocked from accessing the Azure AI Foundry portal, ensuring strict segregation and minimizing the risk of unauthorized access.

## Prerequisites
- **Microsoft Entra tenant**: If you don't have an Microsoft Entra tenant, [create a free account](https://azure.microsoft.com/free/) before you begin.
- **Licensing**: Microsoft Entra Conditional Access requires at minimum Microsoft Entra ID P1 licensing.
- **Permissions**: To implement Microsoft Entra Conditional Access, you should have the Global Administrator or Security Administrator role in the Microsoft Entra tenant.
- **Familiarity with Microsoft Entra ID**: To learn more, refer to [Microsoft Entra fundamentals documentation](https://learn.microsoft.com/en-us/entra/fundamentals/).

## Estimated duration
15 minutes

### Task 1: Create a Microsoft Entra security group

1. Start a web browser, navigate to the Microsoft Entra admin center at [https://entra.microsoft.com](https://entra.microsoft.com) and sign in by providing the credentials of a user account which has the Global Administrator or Security Administrator role assigned at the Microsoft Entra tenant level.
1. In the Microsoft Entra admin center, in the vertical menu on the left side, in the **Entra ID** section, select **Groups**.  
1. On the **Groups \| Overview** page, select **New group**.
1. On the **New Group** page, perform the following actions:

- Ensure that the **Security** entry appears in the **Group type** drop-down list.
- In the **Group name** text box, enter **CA - Azure AI Foundry Users**.
- In the **Group description** text box, enter **Users allowed to access Azure AI Foundry (subject to CA constraints)**.
- Ensure that the **Microsoft Entra roles can be assigned to the group** switch is set to **No**.
- Ensure that the **Membership type** is set to **Assigned**.
- Select **No owners selected**, on the **Add owners** pane, in the **Search** text box, enter the name of your Entra ID user account, select it from the list of results, and click **Select**.
- Select **No members selected**, on the **Add members** pane, in the **Search** text box, enter the name of your Entra ID user account, select it from the list of results, and click **Select**.
- Back on the **New Group** page, select **Create**.

### Task 2: Create a Microsoft Entra Conditional Access named location

1. In the web browser displaying the **Groups \| Overview** page of the Microsoft Entra admin center, in the vertical menu on the left side, in the **Entra ID** section, select **Conditional Access**.  
1. On the **Conditional Access \| Overview** page, in the vertical menu on the left side, in the **Manage** section, select **Named locations**.
1. On the **Conditional Access \| Named locations** page, select **+ IP ranges location**.
1. On the **New location (IP ranges)** pane, perform the following actions:

- In the **Name** text box, enter **HQ**.
- Select the **+** symbol and, in the **Enter a new IPv4 or IPv6 range** text box, enter **13.66.0.0/28**.
- Select **Create**.

### Task 3: Create a Microsoft Entra Conditional Access policy that allows access to Azure AI Foundry portal

1. Back on the **Conditional Access \| Named locations** page, in the vertical menu on the left side, select **Policies**.
1. On the **Conditional Access \| Policies** page, select **+ New policy**.
1. On the **New** page, perform the following actions:

- In the **Name** text box, enter **Allowing access to Azure AI Foundry**.
- In the **Assignments** section, select **Users**, on the **Include** tab, click **Select users and groups** option button, and then select the **Users and groups** checkbox.
- On the **Select users and groups** pane, in the **Search** text box, enter **CA - Azure AI Foundry Users**, in the list of results, select **CA - Azure AI Foundry Users**, and then click **Select**.
- Back on the **New** page, directly below the **Users** entry, click the link **No target resources selected**.
- Ensure that **Resources (formerly cloud apps)** appears in the **Select what this policy applies to** drop-down list.
- On the **Include** tab, click **Select resources**.
- Click the **None** link directly below the **Select** label.
- On the **Select** pane, in the **Search** text box, enter **Azure AI Studio App**, select **Azure AI Studio App** in the list of results, and then click **Select**.

   **Note**: In case **Azure AI Studio App** in the list of results, run the following script from the PowerShell pane of Azure Cloud Shell:

   ```
   # Requires Microsoft.Graph.Authentication module
   # Run in the security context of a user that has the Global Administrator role
   Connect-MgGraph -Scopes 'Application.ReadWrite.All'

   # Azure AI Foundry App ID
   $foundryAppId = 'cb2ff863-7f30-4ced-ab89-a00194bcf6d9'

   # Check if the service principal already exists
   $spUri = "https://graph.microsoft.com/v1.0/servicePrincipals?`$filter=appId eq '$foundryAppId'"
   $spExists = [bool](Invoke-MgGraphRequest -Method GET -Uri $spUri -ContentType 'PSObject' -OutputType PSObject).value.Count -gt 0

   if (-not $spExists) {
      # Create the service principal with a friendly display name
      $body = @{
         appId = $foundryAppId
         displayName = "Azure AI Studio App"
      } | ConvertTo-Json

      Invoke-MgGraphRequest -Method POST -Uri 'https://graph.microsoft.com/v1.0/servicePrincipals' -Body $body -ContentType 'application/json'
      Write-Host -ForegroundColor Green 'Azure AI Foundry App service principal created successfully.'
   } else {
      Write-Host -ForegroundColor Green 'Azure AI Foundry App service principal already exists.'
   }
   ```

- Back on the **New** page, directly below the **Network** entry, click the link **0 conditions selected**.
- In the list of conditions, directly under the **Locations** label, select **Not configured**, set the **Configure** switch to **Yes**, on the **Include** tab, and click **Selected network and locations**.
- On the **Select network**, in the **Search** text box, enter **HQ**, in the list of results, select **HQ** and then select **Save**. 
- Back on the **New** page, in the **Access controls** section, directly below the **Grant** label, click the link **0 controls selected**.
- On the **Grant** pane, ensure that the **Grant access** option is selected, select the **Require multifactor authentication** and **Require Microsoft Entra hybrid joined devices**, and then click **Select**.
- Set the **Enable policy** switch to **Report-only** and then select **Create**.  

### Task 4: Create a Microsoft Entra Conditional Access policy that blocks access to Azure AI Foundry portal

1. Back on the **Conditional Access \| Policies** page, select **+ New policy**.
1. On the **New** page, perform the following actions:

- In the **Name** text box, enter **Blocking access to Azure AI Foundry**.
- In the **Assignments** section, select **Users**, on the **Include** tab, click **All users** option button, select the **Exclude** tab, and then select the **Users and groups** checkbox.
- On the **Select users and groups** pane, in the **Search** text box, enter **CA - Azure AI Foundry Users**, in the list of results, select **CA - Azure AI Foundry Users**, and then click **Select**.
- Back on the **New** page, directly below the **Users** entry, click the link **No target resources selected**.
- Ensure that **Resources (formerly cloud apps)** appears in the **Select what this policy applies to** drop-down list.
- On the **Include** tab, click **Select resources**.
- Click the **None** link directly below the **Select** label.
- On the **Select** pane, in the **Search** text box, enter **Azure AI Studio App**, select **Azure AI Studio App** in the list of results, and then click **Select**.
- Back on the **New** page, in the **Access controls** section, directly below the **Grant** label, click the link **0 controls selected**.
- On the **Grant** pane, select the **Block access** option and then click **Select**.
- Set the **Enable policy** switch to **Report-only** and then select **Create**.  

### Task 5: Perform cleanup

1. In the web browser displaying the Microsoft Entra admin center, in the vertical menu on the left side, in the **Entra ID** section, select **Conditional Access**.  
1. On the **Conditional Access \| Overview** page, in the vertical menu on the left side, select **Policies**.
1. On the **Conditional Access \| Policies** page, in the **Search** text box, enter **Allowing access to Azure AI Foundry** and, in the list of results, select **Allowing access to Azure AI Foundry**.
1. On the **Allowing access to Azure AI Foundry** page, select **Delete** and, when prompted for the confirmation, select **Yes**.
1. Back on the **Conditional Access \| Policies** page, in the **Search** text box, enter **Blocking access to Azure AI Foundry** and, in the list of results, select **Blocking access to Azure AI Foundry**.
1. On the **Blocking access to Azure AI Foundry** page, select **Delete** and, when prompted for the confirmation, select **Yes**.
1. Back on the **Conditional Access \| Policies** page of the Microsoft Entra admin center, in the vertical menu on the left side, in the **Entra ID** section, select **Groups**.  
1. On the **Groups \| Overview** page, in the vertical menu on the left side, select **All groups**.
1. On the **Groups \| All groups** page, in the **Search** text box, enter **CA - Azure AI Foundry Users** and, in the list of results, select **CA - Azure AI Foundry Users**.
1. On the **CA - Azure AI Foundry Users** page, select **Delete** and, when prompted for the confirmation, select **OK**.