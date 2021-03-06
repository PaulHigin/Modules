# PowerShell Secrets Management module

PowerShell Secrets Management module provides a convenient way for a user to store secrets from a local machine.
It can store the secret locally using the built-in local vault.
In addition other secret storage solutions, or extension vaults, can be registered that provide either local or remote secret storage.  

The module supports the following secret types:

- byte[]
- string
- SecureString
- PSCredential
- Hashtable

The module exposes cmdlets for for accessing and manipulating secrets, and also cmdlets for registering and manipulating vault extensions.  

Registering extension vaults:

- Register-SecretsVault
- Get-SecretsVault
- Unregister-SecretsVault

Accessing secrets:

- Add-Secret
- Get-Secret
- Get-SecretInfo
- Remove-Secret

## Vault extension registration

Vault extensions are registered to the current user context.
They are registered as PowerShell modules that expose required methods used by Secrets Management to manipulate secrets.
The required methods can be exposed in two different module types: a binary module or a script module.
There are four methods that Secrets Management requires an extension vault to provide:

- Get-Secret  
Retrieves a single secret object from the extension vault (required).
- Get-SecretInfo  
Enumerates all secrets from the extension vault (required).
Only information about the secret is returned: secret name, secret type.
The actual secret is not returned.
- SetSecret  
Stores a single secret object to the extension vault (optional).
- Remove-Secret  
Removes a single secret object from the extension vault (optional).

### Binary module vault extension

This is a PowerShell module with a manifest file (.psd1) that specifies a managed assembly that implements the SecretsManagementExtension abstract class.
An example of such a module has been provided in the `ExtensionModules\AzKeyVault\` directory.
This binary module uses the Azure `Az.Accounts` and `Az.KeyVault` PowerShell modules to add/remove/retrieve SecureString secrets from an Azure KeyVault resource.
See the `ExtensionModules\AzKeyVault\src\AKVaultExtension.cs` source file for details.
This module requires extra information in order to connect to an Azure KeyVault resource, and uses the Secrets Management optional `-VaultParameters` optional dictionary parameter to provide that information to the implementation module.
Since the Secrets Management can't know if the `-VaultParameters` contain secrets, the dictionary is always securely stored as a hashtable in the built-in local vault.
An example registration of a working vault extension using this module is:

```powershell
Register-SecretsVault -Name AzKeyVault -ModuleName AzKeyVaultModule -VaultParameters @{ AZKVaultName = 'MyAzKeyVault'; SubscriptionId = 'f3bc301d-40b7-4bcb-8e66-b1b238200f02' }
```

This module basically implements the four required methods as a C# class implementing the abstract SecretsManagementExtension class (again see the `AKVaultExtension.cs` source file for details).

```C#
public override object GetSecret(
    string name,
    IReadOnlyDictionary<string, object> parameters,
    out Exception error)
{ }

public override KeyValuePair<string, string>[] GetSecretInfo(
    string filter,
    IReadOnlyDictionary<string, object> parameters,
    out Exception error)
{ }

public override bool SetSecret(
    string name,
    object secret,
    IReadOnlyDictionary<string, object> parameters,
    out Exception error)
{ }

public override bool RemoveSecret(
    string name,
    IReadOnlyDictionary<string, object> parameters,
    out Exception error)
{ }
```

### Script module vault extension

This is a PowerShell module that implements the required four methods as PowerShell script functions.
The actual module containing the script implementation is in a subdirectory of the vault module and named `ImplementingModule`.
This is done to "hide" the implementing module from PowerShell command discovery, which prevents the implementing module script functions from polluting the user commands seen at the command line.
An example of this kind of vault extension module is at `ExtensionModules\AzKeyVaultScript\`.

This module provides the same Azure KeyVault extension as the binary module above, but does so with PowerShell script functions rather than a managed binary implementing type.
It consists of an ImplementingModule module that implements the script functions in the `ImplementingModule.psm1` file, and exports the four required functions in the `ImplementingModule.psd1` manifest file.  

ImplementingModule.psd1

```powershell
@{
    ModuleVersion = '1.0'
    RootModule = '.\ImplementingModule.psm1'
    FunctionsToExport = @('Set-Secret','Get-Secret','Remove-Secret','Get-SecretInfo')
}
```

ImplementingModule.psm1

```powershell
function Get-Secret
{
    param (
        [string] $Name,
        [hashtable] $AdditionalParameters
    )

}

function Get-SecretInfo
{
    param (
        [string] $Filter,
        [hashtable] $AdditionalParameters
    )

}

function Set-Secret
{
    param (
        [string] $Name,
        [object] $Secret,
        [hashtable] $AdditionalParameters
    )

}

function Remove-Secret
{
    param (
        [string] $Name,
        [hashtable] $AdditionalParameters
    )

}
```

A vault extension doesn't need to provide full implementation of all required methods.
For example, a vault extension does not need to provide a way to add or remove a secret through the Secrets Management cmdlets.
However, it should always implement secret retrieval (GetSecret, GetSecretInfo).
If a vault extension doesn't support some functionality (such as add/remove a secret, or a particular secret type), then it should throw an exception with a meaningful error message.  

Both of the binary and script modules implement the four functions in similar ways.
The method/functions take the same parameters, including optional additional parameters, and return the same object(s).

- Get-Secret  
Input Parameters: Name, AdditionalParameters  
Output: Secret object
- Get-SecretInfo  
Input Parameters: Filter, AdditionalParameters  
Output: PSObject with two properties: Name, TypeName
- Set-Secret  
Input Parameters: Name, Secret, AdditionalParameters  
Output: True on success, False otherwise
- Remove-Secret  
Input Parameters: Name, AdditionalParameters  
Output: True on success, False otherwise

You have to be careful with PowerShell script functions, because there are many ways for objects to be added to the output pipeline and the Secrets Management module expects very specific output objects from the functions.
Make sure your script implementation does not inadvertently insert spurious objects to the pipeline, which will confuse the Secrets Management module.
