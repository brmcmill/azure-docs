---
title:  "Expose applications to the internet using Application Gateway"
titleSuffix: Azure Spring Cloud
description: How to expose applications to internet using Application Gateway
author: karlerickson
ms.author: karler
ms.service: spring-cloud
ms.topic: how-to
ms.date: 11/09/2021
ms.custom: devx-track-java
---

# Expose applications to the internet using Application Gateway

This article explains how to expose applications to the internet using Application Gateway. When an Azure Spring Cloud service instance is deployed in your virtual network, applications on the service instance are only accessible in the private network. To make the applications accessible on the Internet, you need to integrate with Azure Application Gateway.

## Prerequisites

- [Azure CLI version 2.0.4 or later](/cli/azure/install-azure-cli).
- An Azure Spring Cloud service instance deployed in a virtual network with an application accessible over the private network using the default `.private.azuremicroservices.io` domain suffix. For more information, see [Deploy Azure Spring Cloud in a virtual network](./how-to-deploy-in-azure-virtual-network.md)
- A custom domain to be used to access the application.
- A certificate, stored in Key Vault, which matches the custom domain to be used to establish the HTTPS listener. For more information, see [Tutorial: Import a certificate in Azure Key Vault](../key-vault/certificates/tutorial-import-certificate.md).

## Configure Application Gateway for Azure Spring Cloud

We recommend that the domain name, as seen by the browser, is the same as the host name which Application Gateway uses to direct traffic to the Azure Spring Cloud back end. This recommendation provides the best experience when using Application Gateway to expose applications hosted in Azure Spring Cloud and residing in a virtual network. If the domain exposed by Application Gateway is different from the domain accepted by Azure Spring Cloud, cookies and generated redirect URLs (for example) can be broken.

To configure Application Gateway in front of Azure Spring Cloud, use the following steps.

1. Follow the instructions in [Deploy Azure Spring Cloud in a virtual network](./how-to-deploy-in-azure-virtual-network.md).
1. Acquire a certificate for your domain of choice and store that in Key Vault. For more information, see [Tutorial: Import a certificate in Azure Key Vault](../key-vault/certificates/tutorial-import-certificate.md).
1. Configure a custom domain and corresponding certificate from Key Vault on an app deployed onto Azure Spring Cloud. For more information, see [Tutorial: Map an existing custom domain to Azure Spring Cloud](./tutorial-custom-domain.md).
1. Deploy Application Gateway in a virtual network configured according to the following list:
   - Use Azure Spring Cloud in the backend pool, referenced by the domain suffixed with `private.azuremicroservices.io`.
   - Include an HTTPS listener using the same certificate from Key Vault.
   - Configure the virtual network with HTTP settings that use the custom domain name configured on Azure Spring Cloud instead of the domain suffixed with `private.azuremicroservices.io`.
1. Configure your public DNS to point to Application Gateway.

## Define variables

Next, use the following commands to define variables for the resource group and virtual network you created as directed in [Deploy Azure Spring Cloud in a virtual network](./how-to-deploy-in-azure-virtual-network.md). Customize the values based on your real environment. When you define `SPRING_APP_PRIVATE_FQDN`, remove `https://` from the URI.

```bash
SUBSCRIPTION='subscription-id'
RESOURCE_GROUP='my-resource-group'
LOCATION='eastus'
SPRING_CLOUD_NAME='name-of-spring-cloud-instance'
APPNAME='name-of-app-in-azure-spring-cloud'
SPRING_APP_PRIVATE_FQDN='$APPNAME.private.azuremicroservices.io'
VIRTUAL_NETWORK_NAME='azure-spring-cloud-vnet'
APPLICATION_GATEWAY_SUBNET_NAME='app-gw-subnet'
APPLICATION_GATEWAY_SUBNET_CIDR='10.1.2.0/24'
```

## Sign in to Azure

Use the following command to sign in to the Azure CLI and choose your active subscription.

```azurecli
az login
az account set --subscription $SUBSCRIPTION
```

## Acquire a certificate

### [Use a publicly signed certificate](#tab/public-cert)

For production deployments, you'll most likely use a publicly signed certificate. In this case, import the certificate in Azure Key Vault. For more information, see [Tutorial: Import a certificate in Azure Key Vault](../key-vault/certificates/tutorial-import-certificate.md). Make sure the certificate includes the entire certificate chain.

### [Use a self-signed certificate](#tab/self-signed-cert)

When you need a self-signed certificate for testing or development, you need to create it. You'll also need to ensure that the list of "Subject Alternative Names" in the certificate contains the domain name on which you'll expose the application. When creating a self-signed certificate through Azure Key Vault, you can do so through the Azure portal. Alternatively, when using the Azure CLI, you'll need a policy JSON file.

To request the default policy, use the following command:

```azurecli
az keyvault certificate get-default-policy
```

Next, adapt the policy JSON as shown in the following example, indicating the `subject` and `SubjectAlternativeNames`:

```json
{
  // ...
    "subject": "C=US, ST=WA, L=Redmond, O=Contoso, OU=Contoso HR, CN=myapp.mydomain.com",
    "subjectAlternativeNames": {
        "dnsNames": [
            "myapp.mydomain.com",
            "*.myapp.mydomain.com"
        ],
        "emails": [
            "hello@contoso.com"
        ],
          "upns": []
      }
  // ...
}
```

After you've finished updating the policy JSON (see [Update Certificate Policy](/rest/api/keyvault/certificates/update-certificate-policy/update-certificate-policy)), you can create a self-signed certificate in Key Vault by using the following commands:

```azurecli
KV_NAME='name-of-key-vault'
CERT_NAME_IN_KV='name-of-certificate-in-key-vault'

az keyvault certificate create \
    --vault-name $KV_NAME \
    --name $CERT_NAME_IN_KV \
    --policy "$KV_CERT_POLICY"
```

---

## Configure the public domain name on Azure Spring Cloud

Traffic will enter the application deployed on Azure Spring Cloud using the public domain name. To configure your application to listen to this host name and do so over HTTPS, use the following commands to add a custom domain to your app:

```azurecli
KV_NAME='name-of-key-vault'
KV_RG='resource-group-name-of-key-vault'
CERT_NAME_IN_ASC='name-of-certificate-in-Azure-Spring-Cloud'
CERT_NAME_IN_KV='name-of-certificate-with-intermediaries-in-key-vault'
DOMAIN_NAME=myapp.mydomain.com

# provide permissions to ASC to read the certificate from Key Vault:
VAULTURI=$(az keyvault show -n $KV_NAME -g $KV_RG --query properties.vaultUri -o tsv)

# get the object id for the Azure Spring Cloud Domain-Management Service Principal:
ASCDM_OID=$(az ad sp show --id 03b39d0f-4213-4864-a245-b1476ec03169 --query objectId --output tsv)

# allow this Service Principal to read and list certificates and secrets from Key Vault:
az keyvault set-policy -g $KV_RG -n $KV_NAME  --object-id $ASCDM_OID --certificate-permissions get list --secret-permissions get list

# add custom domain name and configure TLS using the certificate:
az spring-cloud certificate add \
    --resource-group $RESOURCE_GROUP \
    --service $SPRING_CLOUD_NAME \
    --name $CERT_NAME_IN_ASC \
    --vault-certificate-name $CERT_NAME_IN_KV \
    --vault-uri $VAULTURI
az spring-cloud app custom-domain bind \
    --resource-group $RESOURCE_GROUP \
    --service $SPRING_CLOUD_NAME \
    --domain-name $DOMAIN_NAME \
    --certificate $CERT_NAME_IN_ASC \
    --app $APPNAME
```

> [!NOTE]
> For development and test purposes when you want to do TLS termination at Application Gateway, there's no need to configure a certificate on Azure Spring Cloud or to provide permissions for Azure Spring Cloud to read from Key Vault. You just need to bind the domain name, which you can do with the following command:
>
> ```azurecli
> az spring-cloud app custom-domain bind \
>     --resource-group $RESOURCE_GROUP \
>     --service $SPRING_CLOUD_NAME \
>     --domain-name $DOMAIN_NAME \
>     --app $APPNAME
> ```

## Create network resources

The Azure Application Gateway to be created will join the same virtual network as--or peered virtual network to--the Azure Spring Cloud service instance. First create a new subnet for the Application Gateway in the virtual network using `az network vnet subnet create`, and also create a Public IP address as the Frontend of the Application Gateway using `az network public-ip create`.

```azurecli
APPLICATION_GATEWAY_PUBLIC_IP_NAME='app-gw-public-ip'
az network vnet subnet create \
    --name $APPLICATION_GATEWAY_SUBNET_NAME \
    --resource-group $RESOURCE_GROUP \
    --vnet-name $VIRTUAL_NETWORK_NAME \
    --address-prefix $APPLICATION_GATEWAY_SUBNET_CIDR
az network public-ip create \
    --resource-group $RESOURCE_GROUP \
    --location $LOCATION \
    --name $APPLICATION_GATEWAY_PUBLIC_IP_NAME \
    --allocation-method Static \
    --sku Standard
```

## Create a Managed Identity for Application Gateway

Application Gateway will need to be able to access Key Vault to read the certificate. To do so, it will use a User-assigned [Managed Identity](../active-directory/managed-identities-azure-resources/overview.md). Create the Managed Identity by using the following command:

```azurecli
APPGW_IDENTITY_NAME='name-for-appgw-managed-identity'
az identity create \
    --resource-group $RESOURCE_GROUP \
    --name $APPGW_IDENTITY_NAME
```

Then fetch the objectId for the Managed Identity as it will be used later on to give rights to access the certificate in Key Vault:

```azurecli
APPGW_IDENTITY_CLIENTID=$(az identity show --resource-group $RESOURCE_GROUP --name $APPGW_IDENTITY_NAME --query clientId --output tsv)
APPGW_IDENTITY_OID=$(az ad sp show --id $APPGW_IDENTITY_CLIENTID --query objectId --output tsv)
```

## Set policy on Key Vault

Configure Key Vault using the following command so that the Managed Identity for Application Gateway is allowed to access the certificate stored in Key Vault:

```azurecli
az keyvault set-policy \
    --name $KV_NAME \
    --resource-group $KV_RG \
    --object-id $APPGW_IDENTITY_OID \
    --secret-permissions get list \
    --certificate-permissions get list
```

## Create Application Gateway

Create an application gateway using `az network application-gateway create` and specify your application's private fully qualified domain name (FQDN) as servers in the backend pool. Make sure to use the user-assigned Managed Identity and to point to the certificate in Key Vault using the certificate's Secret ID. Then update the HTTP setting using `az network application-gateway http-settings update` to use the public host name.

```azurecli
APPGW_NAME='name-for-application-gateway'

KEYVAULT_SECRET_ID_FOR_CERT=$(az keyvault certificate show --name $CERT_NAME_IN_KV --vault-name $KV_NAME --query sid --output tsv)

az network application-gateway create \
    --name $APPGW_NAME \
    --resource-group $RESOURCE_GROUP \
    --location $LOCATION \
    --capacity 2 \
    --sku Standard_v2 \
    --frontend-port 443 \
    --http-settings-cookie-based-affinity Disabled \
    --http-settings-port 443 \
    --http-settings-protocol Https \
    --public-ip-address $APPLICATION_GATEWAY_PUBLIC_IP_NAME \
    --vnet-name $VIRTUAL_NETWORK_NAME \
    --subnet $APPLICATION_GATEWAY_SUBNET_NAME \
    --servers $SPRING_APP_PRIVATE_FQDN \
    --key-vault-secret-id $KEYVAULT_SECRET_ID_FOR_CERT \
    --identity $APPGW_IDENTITY_NAME
```

It can take up to 30 minutes for Azure to create the application gateway.

>[!NOTE]
> For development and test purposes when it desired to do TLS termination at Application Gateway, change the `http-settings-port` to `80` and `http-settings-protocol` to `Https` and continue to follow the instructions below for "Use a publicly signed certificate".

### Update HTTP Settings to use the domain name towards the backend

#### [Use a publicly signed certificate](#tab/public-cert-2)

Update the HTTP settings to use the public domain name as the hostname instead of the domain suffixed with ".private.azuremicroservices.io" to send traffic to Azure Spring Cloud with.

```azurecli
az network application-gateway http-settings update \
    --resource-group $RESOURCE_GROUP \
    --gateway-name $APPGW_NAME \
    --host-name-from-backend-pool false \
    --host-name $DOMAIN_NAME \
    --name appGatewayBackendHttpSettings
```

#### [Use a self-signed certificate](#tab/self-signed-cert-2)

Update the HTTP settings to use the public domain name as the hostname instead of the domain suffixed with ".private.azuremicroservices.io" to send traffic to Azure Spring Cloud with. Given that a self-signed certificate is used, it will need to be allow-listed on the HTTP Settings of Application Gateway.

To allowlist the certificate, first fetch the public portion of it from Key Vault by using the following command:

```azurecli
az keyvault certificate download \
    --vault-name $KV_NAME \
    --name $CERT_NAME_IN_KV \
    --file ./selfsignedcert.crt \
    --encoding DER
```

Then upload it to Application Gateway using this command:

```azurecli
az network application-gateway root-cert create \
    --resource-group $RG \
    --cert-file ./selfsignedcert.crt \
    --gateway-name $APPGW_NAME \
    --name MySelfSignedTrustedRootCert
```

Now you can update the HTTP Settings to trust the new (self-signed) root certificate by using this command:

```azurecli
az network application-gateway http-settings update \
    --resource-group $RG \
    --gateway-name $APPGW_NAME \
    --host-name-from-backend-pool false \
    --host-name $DOMAIN_NAME \
    --name appGatewayBackendHttpSettings \
    --root-certs MySelfSignedTrustedRootCert
```

---

### Check the deployment of Application Gateway

After it's created, check the backend health by using the following command. The output of this command enables you to determine whether the application gateway reaches your application through its private FQDN.

```azurecli
az network application-gateway show-backend-health \
    --name $APPGW_NAME \
    --resource-group $RESOURCE_GROUP
```

The output indicates the healthy status of backend pool, as shown in the following example:

```output
{
  "backendAddressPools": [
    {
      "backendHttpSettingsCollection": [
        {
          "servers": [
            {
              "address": "my-azure-spring-cloud-hello-vnet.private.azuremicroservices.io",
              "health": "Healthy",
              "healthProbeLog": "Success. Received 200 status code",
              "ipConfiguration": null
            }
          ]
        }
      ]
    }
  ]
}
```

## Configure DNS and access the application

Now configure the public DNS to point to Application Gateway using a CNAME or A-record. You can find the public address for Application Gateway by using the following command:

```azurecli
az network public-ip show \
    --resource-group $RESOURCE_GROUP \
    --name $APPLICATION_GATEWAY_PUBLIC_IP_NAME \
    --query [ipAddress] \
    --output tsv
```

You can now access the application using the public domain name.

## See also

- [Troubleshooting Azure Spring Cloud in VNET](./troubleshooting-vnet.md)
- [Customer Responsibilities for Running Azure Spring Cloud in VNET](./vnet-customer-responsibilities.md)
