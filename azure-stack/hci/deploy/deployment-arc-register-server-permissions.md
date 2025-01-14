--- 
title: Register your Azure Stack HCI servers with Azure Arc and assign permissions for deployment 
description: Learn how to Register your Azure Stack HCI servers with Azure Arc and assign permissions for deployment. 
author: alkohli
ms.topic: how-to
ms.date: 02/29/2024
ms.author: alkohli
ms.subservice: azure-stack-hci
ms.custom: devx-track-azurepowershell
---

# Register your servers and assign permissions for Azure Stack HCI, version 23H2 deployment

[!INCLUDE [applies-to](../../includes/hci-applies-to-23h2.md)]

This article describes how to register your Azure Stack HCI servers and then set up the required permissions to deploy an Azure Stack HCI, version 23H2 cluster.

## Prerequisites

Before you begin, make sure you've completed the following prerequisites:

- Satisfy the [prerequisites and complete deployment checklist](./deployment-prerequisites.md).
- Prepare your [Active Directory](./deployment-prep-active-directory.md) environment.
- [Install the Azure Stack HCI, version 23H2 operating system](./deployment-install-os.md) on each server.

- Make sure to register your subscription with the following resource providers. You would need to be an owner or contributor on your subscription to perform this registration. Run the following PowerShell commands to register the resource providers:

    ```powershell
    - Register-AzResourceProvider -ProviderNamespace "Microsoft.HybridCompute"
    - Register-AzResourceProvider -ProviderNamespace "Microsoft.GuestConfiguration"
    - Register-AzResourceProvider -ProviderNamespace "Microsoft.HybridConnectivity"
    - Register-AzResourceProvider -ProviderNamespace "Microsoft.AzureStackHCI"
    ```

- If you're registering the servers as Arc resources, make sure that you have the following permissions on the resource group where the servers were provisioned:

    - [Azure Connected Machine Onboarding](/azure/azure-arc/servers/onboard-service-principal#azure-portal)
    - [Azure Connected Machine Resource Administrator](/azure/azure-arc/servers/security-overview#identity-and-access-control)

    To verify that you have these roles, follow these steps in the Azure portal:

    1. Go to the subscription that you'll use for the Azure Stack HCI deployment.
    1. Go to the resource group where you're planning to register the servers.
    1. In the left-pane, go to **Access Control (IAM)**.
    1. In the right-pane, go the **Role assignments**. Verify that you have the **Azure Connected Machine Onboarding** and **Azure Connected Machine Resource Administrator** roles assigned.

    <!--:::image type="content" source="media/deployment-arc-register-server-permissions/contributor-user-access-administrator-permissions.png" alt-text="Screenshot of the roles and permissions assigned in the deployment subscription." lightbox="./media/deployment-arc-register-server-permissions/contributor-user-access-administrator-permissions.png":::-->

## Register servers with Azure Arc

> [!IMPORTANT]
> Run these steps on every Azure Stack HCI server that you intend to cluster.

1. Install the [Arc registration script](https://www.powershellgallery.com/packages/AzSHCI.ARCInstaller) from PSGallery.

    ```powershell
    #Register PSGallery as a trusted repo
    Register-PSRepository -Default -InstallationPolicy Trusted
    
    #Install Arc registration script from PSGallery 
    Install-Module AzsHCI.ARCinstaller

    #Install required PowerShell modules in your node for registration
    Install-Module Az.Accounts -Force
    Install-Module Az.ConnectedMachine -Force
    Install-Module Az.Resources -Force
    ```

    Here's a sample output of the installation:

    ```output
    PS C:\Users\SetupUser> Install-Module -Name AzSHCI.ARCInstaller                                           
    NuGet provider is required to continue                                                                                  
    PowerShellGet requires NuGet provider version '2.8.5.201' or newer to interact with NuGet-based repositories. The NuGet  provider must be available in 'C:\Program Files\PackageManagement\ProviderAssemblies' or
    'C:\Users\SetupUser\AppData\Local\PackageManagement\ProviderAssemblies'. You can also install the NuGet provider by
    running 'Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force'. Do you want PowerShellGet to install
    and import the NuGet provider now?
    [Y] Yes  [N] No  [S] Suspend  [?] Help (default is "Y"): Y
    PS C:\Users\SetupUser>
    
    PS C:\Users\SetupUser> Install-Module Az.Accounts -Force
    PS C:\Users\SetupUser> Install-Module Az.ConnectedMachine -Force
    PS C:\Users\SetupUser> Install-Module Az.Resources -Force
    ```

1. Set the parameters. The script takes in the following parameters:

    |Parameters  |Description  |
    |------------|-------------|
    |`SubscriptionID`    |The ID of the subscription used to register your servers with Azure Arc.         |
    |`TenantID`          |The tenant ID used to register your servers with Azure Arc. Go to your Microsoft Entra ID and copy the tenant ID property.       |
    |`ResourceGroup`     |The resource group precreated for Arc registration of the servers. A resource group is created if one doesn't exist.         |
    |`Region`            |The Azure region used for registration. See the [Supported regions](../concepts/system-requirements-23h2.md#azure-requirements) that can be used.          |
    |`AccountID`         |The user who will register and deploy the cluster.         |
    |`DeviceCode`        |The device code displayed in the console at `https://microsoft.com/devicelogin` and is used to sign in to the device.         |

    

   ```powershell
    #Define the subscription where you want to register your server as Arc device
    $Subscription = "YourSubscriptionID"
    
    #Define the resource group where you want to register your server as Arc device
    $RG = "YourResourceGroupName"

    #Define the region you will use to register your server as Arc device
    $Region = "eastus"
    
    #Define the tenant you will use to register your server as Arc device
    $Tenant = "YourTenantID"
    ```

    Here's a sample output of the parameters:

    ```output
    PS C:\Users\SetupUser> $Subscription = "<Subscription ID>"
    PS C:\Users\SetupUser> $RG = "myashcirg"
    PS C:\Users\SetupUser> $Tenant = "<Tenant ID>"
    PS C:\Users\SetupUser> $Region = "eastus"
    ```

1. Connect to your Azure account and set the subscription. You'll need to open browser on the client that you're using to connect to the server and open this page: `https://microsoft.com/devicelogin` and enter the provided code in the Azure CLI output to authenticate. Get the access token and account ID for the registration.  

    ```powershell
    #Connect to your Azure account and Subscription
    Connect-AzAccount -SubscriptionId $Subscription -TenantId $Tenant -DeviceCode

    #Get the Access Token for the registration
    $ARMtoken = (Get-AzAccessToken).Token

    #Get the Account ID for the registration
    $id = (Get-AzContext).Account.Id   
    ``` 

    Here's a sample output of setting the subscription and authentication:

    ```output
    PS C:\Users\SetupUser> Connect-AzAccount -SubscriptionId $Subscription -TenantId $Tenant -DeviceCode
    WARNING: To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code A44KHK5B5
    to authenticate.
    
    Account               SubscriptionName      TenantId                Environment
    -------               ----------------      --------                -----------
    guspinto@contoso.com AzureStackHCI_Content  <Tenant ID>             AzureCloud

    PS C:\Users\SetupUser> $ARMtoken = (Get-AzAccessToken).Token
    PS C:\Users\SetupUser> $id = (Get-AzContext).Account.Id
    ```
 
1. Finally run the Arc registration script. The script takes a few minutes to run.

    ```powershell
    #Invoke the registration script. Use a supported region.
    Invoke-AzStackHciArcInitialization -SubscriptionID $Subscription -ResourceGroup $RG -TenantID $Tenant -Region $Region -Cloud "AzureCloud" -ArmAccessToken $ARMtoken -AccountID $id  
    ```

    If you're accessing the internet via a proxy server, you need to pass the `-proxy` parameter and provide the proxy server as `http://<Proxy server FQDN or IP address>:Port` when running the script. 

    Here's a sample output of a successful registration of your servers:
    
    ```output
    PS C:\DeploymentPackage> Invoke-AzStackHciArcInitialization -SubscriptionID $Subscription -ResourceGroup $RG -TenantID $Tenant -Region $Region -Cloud "AzureCloud" -ArmAccessToken $ARMtoken -AccountID $id -Force
    Installing and Running Azure Stack HCI Environment Checker
    All the environment validation checks succeeded
    Installing Hyper-V Management Tools
    Starting AzStackHci ArcIntegration Initialization
    Installing Azure Connected Machine Agent
    Total Physical Memory:         588,419 MB
    PowerShell version: 5.1.25398.469
    .NET Framework version: 4.8.9032
    Downloading agent package from https://aka.ms/AzureConnectedMachineAgent to C:\Users\AzureConnectedMachineAgent.msi
    Installing agent package
    Installation of azcmagent completed successfully
    0
    Connecting to Azure using ARM Access Token
    Connected to Azure successfully   
    Microsoft.HybridCompute RP already registered, skipping registration 
    Microsoft.GuestConfiguration RP already registered, skipping registration
    Microsoft.HybridConnectivity RP already registered, skipping registration
    Microsoft.AzureStackHCI RP already registered, skipping registration
    INFO    Connecting machine to Azure... This might take a few minutes.
    INFO    Testing connectivity to endpoints that are needed to connect to Azure... This might take a few minutes.
      20% [==>            ]
      30% [===>           ]
      INFO    Creating resource in Azure...
    Correlation ID=<Correlation ID>=/subscriptions/<Subscription ID>/resourceGroups/myashci-rg/providers/Microsoft.HybridCompute/machines/ms309
      60% [========>      ]
      80% [===========>   ]
     100% [===============]
      INFO    Connected machine to Azure
    INFO    Machine overview page: https://portal.azure.com/
    Connected Azure ARC agent successfully
    Successfully got the content from IMDS endpoint
    Successfully got Object Id for Arc Installation <Object ID>
    $Checking if Azure Stack HCI Device Management Role is assigned already for SPN with Object ID: <Object ID>
    Assigning Azure Stack HCI Device Management Role to Object : <Object ID>
    $Successfully assigned Azure Stack HCI Device Management Role to Object Id <Object ID>
    Successfully assigned permission Azure Stack HCI Device Management Service Role to create or update Edge Devices on the resource group
    $Checking if Azure Connected Machine Resource Manager is assigned already for SPN with Object ID: <Object ID>
    Assigning Azure Connected Machine Resource Manager to Object : <Object ID>
    $Successfully assigned Azure Connected Machine Resource Manager to Object Id <Object ID>
    Successfully assigned the Azure Connected Machine Resource Manager role on the resource group
    $Checking if Reader is assigned already for SPN with Object ID: <Object ID>
    Assigning Reader to Object : <Object ID>
    $Successfully assigned Reader to Object Id <Object ID>
    Successfully assigned the reader Resource Manager role on the resource group
    Installing  TelemetryAndDiagnostics Extension
    Successfully triggered  TelemetryAndDiagnostics Extension installation
    Installing  DeviceManagement Extension
    Successfully triggered  DeviceManagementExtension installation
    Installing LcmController Extension
    Successfully triggered  LCMController Extension installation
    Please verify that the extensions are successfully installed before continuing...
    
    Log location: C:\Users\Administrator\.AzStackHci\AzStackHciEnvironmentChecker.log
    Report location: C:\Users\Administrator\.AzStackHci\AzStackHciEnvironmentReport.json
    Use -Passthru parameter to return results as a PSObject.   
    ```

1. After the script completes successfully on all the servers, verify that:


    1. Your servers are registered with Arc. Go to the Azure portal and then go to the resource group associated with the registration. The servers appear within the specified resource group as **Machine - Azure Arc** type resources.

        :::image type="content" source="media/deployment-arc-register-server-permissions/arc-servers-registered-1.png" alt-text="Screenshot of the Azure Stack HCI servers in the resource group after the successful registration." lightbox="./media/deployment-arc-register-server-permissions/arc-servers-registered-1.png":::

    1. The mandatory Azure Stack HCI extensions are installed on your servers. From the resource group, select the registered server. Go to the **Extensions**. The mandatory extensions show up in the right pane.

        :::image type="content" source="media/deployment-arc-register-server-permissions/mandatory-extensions-installed-registered-servers.png" alt-text="Screenshot of the Azure Stack HCI registered servers with mandatory extensions installed." lightbox="./media/deployment-arc-register-server-permissions/mandatory-extensions-installed-registered-servers.png":::

## Assign required permissions for deployment

This section describes how to assign Azure permissions for deployment from the Azure portal.

1. In the Azure portal, go to the subscription used to register the servers. In the left pane, select **Access control (IAM)**. In the right pane, select **+ Add** and from the dropdown list, select **Add role assignment**.

    :::image type="content" source="media/deployment-arc-register-server-permissions/add-role-assignment-a.png" alt-text="Screenshot of the Add role assignment in Access control in subscription for Azure Stack HCI deployment." lightbox="./media/deployment-arc-register-server-permissions/add-role-assignment-a.png":::

1. Go through the tabs and assign the following role permissions to the user who deploys the cluster:

    - **Azure Stack HCI Administrator**
    - **Reader**

1. In the Azure portal, go to the resource group used to register the servers on your subscription. In the left pane, select **Access control (IAM)**. In the right pane, select **+ Add** and from the dropdown list, select **Add role assignment**.

    :::image type="content" source="media/deployment-arc-register-server-permissions/add-role-assignment.png" alt-text="Screenshot of the Add role assignment in Access control in resource group for Azure Stack HCI deployment." lightbox="./media/deployment-arc-register-server-permissions/add-role-assignment.png":::

1. Go through the tabs and assign the following permissions to the user who deploys the cluster:

    - **Key Vault Administrator**
    - **Key Vault Contributor**
    - **Storage Account Contributor**
 
    <!--:::image type="content" source="media/deployment-arc-register-server-permissions/add-role-assignment-3.png" alt-text="Screenshot of the review + Create tab in Add role assignment for Azure Stack HCI deployment." lightbox="./media/deployment-arc-register-server-permissions/add-role-assignment-3.png":::-->

1. In the right pane, go to **Role assignments**. Verify that the deployment user has all the configured roles. 

    :::image type="content" source="media/deployment-arc-register-server-permissions/add-role-assignment-4.png" alt-text="Screenshot of the Current role assignment in Access control in resource group for Azure Stack HCI deployment." lightbox="./media/deployment-arc-register-server-permissions/add-role-assignment-4.png":::

## Next steps

After setting up the first server in your cluster, you're ready to deploy using Azure portal:

- [Deploy using Azure portal](./deploy-via-portal.md).
