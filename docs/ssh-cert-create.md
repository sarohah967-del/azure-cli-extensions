# az provisionedmachine ssh-cert-create
Create a short-lived SSH certificate for authenticating to a provisioned machine.

Requests a role-specific SSH authorization from HCI RP (a JSON Web Signature signed with the HCI RP SSH CA key), embeds it in an OpenSSH certificate and countersigns that certificate with the device's CA key stored in the customer Azure Key Vault. The certificate includes the caller's identity and role, and is valid for the active access window (at most 8 hours).

Requires an active eligible Provisioned Machine Admin or Contributor role on the target device resource. The caller's identity and role are resolved automatically.

## Flow

1. Verify the caller's active PIM activation and resolve the role (`provisionedmachineadmin` or `provisionedmachinecontributor`). Reader cannot SSH.
2. Generate an ephemeral RSA-4096 key pair.
3. `POST {resourceId}/issueSshCertificateForAdmin` or `.../issueSshCertificateForContributor` (`api-version=2026-03-15-preview`) with `username`, `userPublicKey`, `durationInHours`, `validFrom` and `validTo`. The window is clamped to the RP limit of 8 hours (minus a 5-minute clock-skew margin). HCI RP returns `username`, `role`, `validFrom`, `validTo` and `userSSHJSONWebSignature` (compact JWS strings).
4. Build an OpenSSH user certificate with the RP-authorized username, role and validity. The JWS strings are stored comma-separated in the non-critical `hci-rp-jws@microsoft.com` extension. The certificate is signed (countersigned) with the `{deviceId}-ssh-ca` key in the customer Key Vault.
5. The device verifies the Key Vault (customer device CA) signature, then the HCI RP JWS in the extension against the HCI RP CA trust anchor.

HCI RP returns HTTP 501 while issuance is disabled; the command reports this as an error.

## Parameters

Parameter
Parameter Type
Comments/Description
Resource ID
(resource-id, r)

Required
Fully qualified ARM resource ID of the target device
Vault Name
(vault-name, v)

Required
Name of the Azure Key Vault containing the SSH CA signing key
Cert Path
(cert-path)

Optional
Custom output path for the certificate file. Defaults to a temporary directory
Private Key Path
(private-key-path)

Optional
Custom output path for the private key file. Defaults to a temporary directory
Subscription
(subscription)

Optional
Subscription id

##  Example

```
az provisionedmachine ssh-cert-create \  --vault-name "myKeyVault" \  --resource-id "/subscriptions/.../providers/Microsoft.AzureStackHCI/edgeMachines/myDevice" 
```

```
az provisionedmachine ssh-cert-create \  --vault-name "myKeyVault" \  --resource-id "/subscriptions/.../providers/Microsoft.AzureStackHCI/edgeMachines/myDevice" \  --private-key-path "~/.ssh/device_key" \  --cert-path "~/.ssh/device_cert.pub" 
```

## Generated Command

```
Command
    az provisionedmachine ssh-cert-create : Create a short-lived SSH certificate for authenticating
    to a provisioned machine.
        Generates an SSH certificate signed by the device's CA key stored in
        Azure Key Vault. The certificate includes the caller's identity and
        role, and is valid for the duration of the active access window.

        Requires an active eligible role on the target device resource.
        The caller's identity and role are resolved automatically.

Arguments
    --resource-id -r [Required] : Fully qualified ARM resource ID of the target device.
    --vault-name -v  [Required] : Name of the Azure Key Vault containing the SSH CA signing key.
    --cert-path                 : Custom output path for the certificate file. Defaults to a
                                  temporary directory.
    --private-key-path          : Custom output path for the private key file. Defaults to a
                                  temporary directory.

Global Arguments
    --debug                     : Increase logging verbosity to show all debug logs.
    --help -h                   : Show this help message and exit.
    --only-show-errors          : Only show errors, suppressing warnings.
    --output -o                 : Output format.  Allowed values: json, jsonc, none, table, tsv,
                                  yaml, yamlc.  Default: json.
    --query                     : JMESPath query string. See http://jmespath.org/ for more
                                  information and examples.
    --subscription              : Name or ID of subscription. You can configure the default
                                  subscription using `az account set -s NAME_OR_ID`.
    --verbose                   : Increase logging verbosity. Use --debug for full debug logs.

Examples
    Create a certificate using default output paths
        az provisionedmachine ssh-cert-create --vault-name myKeyVault --resource-id
        /subscriptions/.../providers/Microsoft.AzureStackHCI/edgeMachines/myDevice

    Create a certificate with custom output paths
        az provisionedmachine ssh-cert-create --vault-name myKeyVault --resource-id
        /subscriptions/.../providers/Microsoft.AzureStackHCI/edgeMachines/myDevice --private-key-
        path ~/.ssh/device_key --cert-path ~/.ssh/device_cert.pub
```
