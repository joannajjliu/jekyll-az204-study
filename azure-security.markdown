---
layout: page
title: Implement Azure Security
permalink: /azure-security/
---

## (15 - 20%)

### Implement User Authentication and Authorization

- Implement OAuth2 authentication
  - OAuth2 Implicit grant flow: Use for SPAs/ client-based apps. Allows apps to get tokens without a backend server credential exchange.
  - Authorization code grant flow: Perform authentication and authorization for server-based web applications. Requires the application to provide a client secret, or certificate, to securely provide access tokens.
  - Multi-tenant and personal account type: Allows users to log in with accounts in any Azure AD directory, as well as a personal Microsoft account.
  - Single-tenant: apps are only avaialble in the tenant (directory) they were registered in, also known as their home tenant.
  - Multi-tenant: apps are available to users in both their home tenant, as well as other tenant.
- Create and implement shared access signatures
- Register apps and use Azure Active Directory to authenticate users:

  - _API Management authentication policies_:

    - **Basic**: Authenticate with a backend service using Basic authentication. Effectively sets the HTTP Authorization header to the value corresponding to the credentials provided in the policy. Does not involve Azure AD.

    ```xml
    <authentication-basic username="username" password="password" />
    ```

    - **Client Certificate**: authenticate with a backend service using client certiciate. The certificate needs to be installed into API Management first, and is identified by its thumprint.

    ```xml
    <!--Policy statement-->
    <authentication-certificate thumbprint="thumbprint" certificate-id="resource name" />

    <!--Examples-->
    <!--Client certificate identified by thumbprint-->
    <authentication-certificate thumbprint="CA06F56B258B7A0D4F2B05470939478" />
    <!--Client certificate identified by resource name-->
    <authentication-certificate certificate-id="544fe9ddf3b8f30fb4" />
    ```

    - **Managed identity**: Authenticate with a backend service using the managed identity. Uses the managed identity to obtain an access token from Azure AD, for accessing the specified resource. Both system-assigned identity and multiple user-assigned identity can be used to request a token. If client-id is not provided, system-assigned identity is assumed.

    ```xml
    <!--Policy statement-->
    <authentication-managed-identity resource="resource" client-id="clientid of user-assigned identity" output-token-variable-name="token-variable" ignore-error="true|false" />
    <!--Authenticate with a backend service-->
    <authentication-managed-identity resource="https://management.azure.com/" /> <!--Azure Resource Manager-->
    ```

  - _Access restriction policies_:
    - **Check HTTP header**: Enforce existence and/or value of a HTTP Header.
    - **Limit call rate by subscription**: Prevents API usage spikes by limiting call rate, on a per subscription basis.
    - **Limit call rate by key**: Prevent API usage spikes by limiting call rate, on a per key basis
    - **Restrict caller IPs**: Filter (allow/deny) calls from specific IP addresses and/or address ranges.
    - **Set usage quota by subscription**: Enforce a renewable or lifetime call volume and/or bandwidth quota, on a per subscription basis.
    - **Set usage quota by key**: Enforce a renewable or lifetime call volume and/or bandwith quota, on a per key basis.
    - **Validate JWT (JSON Web Token)**: Enforces existence and validity of a JWT extracted from either a specified HTTP Header, or a specified query parameter.

- Control access to resources using role-based access controls (RBAC):
  - RBAC scope can be assigned at various levels: **management group** > **subscription** > **resource group** > **resource**
  - **Contributor**: Create and manage all types of Azure resources, without the ability to grant resource access to other users.
  - **Owner**: Has full access to Azure, including granting resource access to other users.
  - **Reader**: Only allowed to view resources.
  - **User Access Administrator**: Grant resource access to other users.

### Implement Secure Cloud Solutions

- Secure app configuration data by using the App Configuration and KeyVault API

  - _Azure App Configuration_: a service to centrally manage application settings and feature flags.
  - Azure App Configuration works WITH Azure Key Vault. Data is encrypted in Azure App Configuration, and Azure Key Vault uses a more secure storage environment (hardware-level encryption, granular access policies, and management operations)
  - Separate configuration stores can be configured to support different environments, including development, test, and production.
  - Configure disk encryption for an Azure Linux VM, using an encryption key from a Key Vault, created for the purpose. Use a KEK (key encryption key) to protect the encryption secret:

  ```powershell
  # create Key Vault, and enable it to support disk encryption
  az keyvault create --name "myKeyVault" --resource-group "myRG" --location "eastus"
  az keyvault update --name "myKeyVault" --resource-group "myRG" --enabled-for-disk-encryption "true"

  <#  create a KEK as an additional layer of security for encryption keys,
  and add to Key Vault.
  For Azure disk encryption, an RSA key type must be specified;
  other key type options are not supported #>
  az keyvault key create --name "myKEK" --vault-name "myKeyVault" --kty RSA-HSM

  az vm encryption enable -g "myRG" --name "myLinuxVM" --disk-encryption-keyvault "myKeyVault" --key-encryption-key "myKEK"
  ```

  - **az appconfig kv export**: Copy key-value pairs from the specified App Configuration file to a local file (ex. JSON file), or to a different App Configuration store. Ex:

  ```powershell
  az appconfig kv export --name myDevAppConfigStore --file ~.DevFix.json
  ```

  - **az appconfig kv import**: Copy from one or more sources (ex. other App Configuration files, JSON, YAML, or properties files) into an App Configuration file.

- Manage keys, secrets, and certificates by using the KeyVault API:
  - **az keyvault key import**: Import private key into a Key Vault.
  - **az keyvault key backup**: Back up a private key that is downloaded to the client running the command.
  - **az keyvault secret**: Manage Key Vault secrets (including backing up, restoring, and recovering secrets)
- Implement Managed Identities for Azure resources

  - Enable system-assigned managed identity during Azure VM creation (use **AssignIdentity** param):

  ```powershell
  $vmConfig = New-AzVMConfig -VMName myVM -AssignIdentity:$SystemAssigned ...
  ```

  - Enable system-assigned managed identity on an existing Azure VM, using PowerShell (use **AssignIdentity** param):

  ```powershell
  Connect-AzAccount

  # retrieve VM properties:
  $vm = Get-AzVM -ResourceGroupName myResourceGroup -Name myVM

  # enable system-assigned managed identity
  Update-AzVM -ResourceGroupName myResourceGroup -VM $vm -AssignIdentity:$SystemAssigned
  ```

  - Add a user-assigned managed identity to an existing Azure VM:

  ```powershell
  # returns Id
  New-AzUserAssignedIdentity -ResourceGroupName <RESOURCEGROUP> -Name <USER ASSIGNED IDENTITY NAME>

  # Returned Id from previous step usined in IdentityID parameter
  $vm = Get-AzVM -ResourceGroupName <RESOURCE GROUP> -Name <VM NAME>
  Update-AzVM -ResourceGroupNAme <RESOURCE GROUP> -VM $vm -IdentityType UserAssigned -IdentityID "/subscriptions/<SUBSCRIPTION ID>/resourcegroups/<RESOURCE GROUP>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/<USER ASSIGNED IDENTITY NAME>"
  ```

  - Change identity on an existing Azure VM, using Powershell:
    - **IdentityType** values: None, SystemAssigned, UserAssigned, SystemAssignedUserAssigned
    - Specify an **IdentityID**, to remove all other identities on an Azure VM

  ```powershell
  $vm = Get-AzVm -ResourceGroupName myResourceGroup -Name myVM

  Update-AzVM -ResourceGroupName myResourceGroup -VirtualMachine $vm -IdentityType UserAssigned -IdentityID <USER ASSIGNED IDENTITY NAME>
  ```
