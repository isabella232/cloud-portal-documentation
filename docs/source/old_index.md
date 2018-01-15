# Index

## Access to the [`EMBL-EBI Cloud Portal`](https://portal.tsi.ebi.ac.uk)

The EMBL-EBI Cloud Portal is offered as a service to ELIXIR and other communities.  
Users will need a freely available [ELIXIR identity](https://www.elixir-europe.org/services/compute/aai) to access the portal.  
Obtain your ELIXIR identity [here](https://www.elixir-europe.org/register). Then you can [login](https://portal.tsi.ebi.ac.uk/welcome/login).


## Set up the cloud [`Profile`](https://portal.tsi.ebi.ac.uk/profile)

From the global menu click [`Profile`](https://portal.tsi.ebi.ac.uk/profile).  

A minimal Profile set up include this steps:

- Create a new entry in the `Profile - Cloud Credentials` section.
- Create a new entry in the `Profile - Deployment Parameters` section.
- Create a new entry in the `Profile - Configurations` section.


### Cloud Credentials

Each cloud provider requires a set of **cloud credentials**.  
**Personal credentials** can be set up following the instructions for the specific cloud provider.  
It also possible to use **shared credentials**, in this case, the values will not be visible or editable.

#### Amazon Web Service (AWS)
Add your `Cloud Credentials` in this form:
```eval_rst
+-------------------+-----------------+
| Key               | Value           |
+===================+=================+
|`TF_VAR_access_key`|`your_access-key`|
+-------------------+-----------------+
|`TF_VAR_secret_key`|`your_secret_key`|
+-------------------+-----------------+
```

|Key|Value|
|---|---|
|`TF_VAR_access_key`|`your_access-key`|
|`TF_VAR_secret_key`|`your_secret_key`|

You can find this information (or create a new one) in you AWS user page under the section:

`IAM` ---> `USERS`

#### Azure
Add your `Cloud Credentials` in this form:

|Key|Value|
|---|---|
|`ARM_SUBSCRIPTION_ID `|`YOUR_SUBSCRIPTION_ID`|
|`ARM_CLIENT_ID `|`YOUR_CLIENT_ID`|
|`ARM_CLIENT_SECRET `|`YOUR_CLIENT_SECRET`|
|`ARM_TENANT_ID `|`YOUR_TENANT_ID`|
|`ARM_ENVIRONMENT `|`public`|

In order to use Terraform with Azure is required to create a `Service Principal` via the Azure portal.  
Terraform documentation provide an extensive explanation that you can find [here](https://www.terraform.io/docs/providers/azurerm/authenticating_via_service_principal.html#creating-a-service-principal-in-the-azure-portal).

#### OpenStack
Add your `Cloud Credentials` in this form:

|Key|Value|
|---|---|
|`OS_AUTH_URL`|`https://extcloud06.ebi.ac.uk:13000/v3`|
|`OS_USERNAME`|`your_username`|
|`OS_PASSWORD`|`your_password`|
|`OS_PROJECT_ID`|`your_project_id `|
|`OS_PROJECT_NAME`|`EBI-TSI`|
|`OS_USER_DOMAIN_NAME`|`Default`|
|`OS_INTERFACE`|`public`|
|`OS_IDENTITY_API_VERSION`|`3`|

The specific values for your OpenStack provider are contained in the `OpenStack RC file`, which is project-specific and contains the credentials used by OpenStack Compute, Image, and Identity services.

You can download the `OpenStack RC file` file from the OpenStack dashboard as an administrative user or any other user.

1. Log in to the OpenStack dashboard.
2. Choose the `project` for which you want to download the OpenStack RC file
3. Click `Compute` ---> `API Access`.
	(In older version of OpenStack click `Compute` --->`Access & Security`)
4. Click` Download OpenStack RC File` and save the file.

#### Google Cloud Platform (GCP)

Authenticating with Google Cloud services requires the set up of The [`GOOGLE_CREDENTIALS `](https://developers.google.com/identity/protocols/application-default-credentials#howtheywork)
variable, which can also contain the path of a file to obtain credentials from.

The `GOOGLE_CREDENTIALS` is available in a JSON file.

This file is downloaded directly from the [Google Developers Console](https://console.developers.google.com/).  
To make the process more straightforward, it is documented here:

1. Log into the [Google Developers Console](https://console.developers.google.com/) and select a `project`.
2. The API Manager view should be selected, click on `Credentials` on the left, then `Create credentials`, and finally "Service account key".
3. Select `Compute Engine default service account` in the `Service account` drop-down, and select `JSON` as the key type.
4. Clicking `Create` will download your `credentials`.

Once you have your credentials you can add them to `Cloud Credentials` in this form:

| Key|Value|
|---|---|
|`GOOGLE_CREDENTIALS`| `{ "type":"service_account", "project_id":"", "private_key_id":"", "private_key":"-----BEGIN PRIVATE KEY-----", "client_email": "", "client_id":"", "auth_uri":"", "token_uri":"", "auth_provider_x509_cert_url":"", "client_x509_cert_url":""}`|


### Deployment parameters

`Deployment parameters` represent a set of inputs specific that are related to the cloud provider and eventually to a specific application.  
In general, they provide information about the shared instances that you can have in place in your cloud provider or just information that you prefer to set up just the first time and avoid to repeat every time you deploy the instance.

The deployment parameters required by an appliance are expressed in the documentation page of the git repository of the same appliance.

For your convenience you can use a single `deployment parameter` configuration for different appliances:  it will make use only of the share inputs ignoring the ones that are not relevant.  
A deployment parameter can also be used to overwrite any of the variables defined in the `terraform.tfvars` file even when it is not reported as input in the `manifest` file.  

* Note: The current version of the portal is requiring to include the parameters of all the cloud providers, in the deployment configurations. This behaviour will change soon, for the moment please enter a random value for the parameters that are not necessary for your cloud provider.

### Configuration

Configurations represent a way to link a Cloud Credential with a Deployment Parameter and an SSH public key.  
The use of a configuration simplifies the deployment of the application, allowing to store and reuse the specified parameters.  
Specify a new configuration is very easy:

 - click on the `+`  button
 - assign a name of your choice,
 - choose one of the `Cloud Provider` that you have previously defined,
 - choose one of the `Deployment parameters` that you have previously defined,
 - (optionally) add a public ssh key

## Inputs

`Inputs` parameters represent a set of parameters that are meant to more volatile than the `Deployment parameters` or just cannot be determined in advance.  
They are related to the specific application, and to the cloud provider and eventually.  
They are expressed on the documentation page of the git repository of the same appliance.

You can also avoid typing the inputs and it will use the defaults (but avoid to click with the mouse on them, otherwise it will see as a void instead of the default).

## Add a new Application Repository

Adding a new Application is extremely simple you just need to know the URL of the git repository, for example, you can add this address: `https://github.com/EMBL-EBI-TSI/cpa-instance`

- From the global menu click [`Application Repository`](https://portal.tsi.ebi.ac.uk/repository)
- click on the `+`  button
- enter the `URL` of the git repository

### Repository compliance

The `EMBL-EBI Cloud Portal` requires the presence of well-formed `manifest.json` file, in the `root directory` of each git repository, to be considered compliant and therefore be accepted as a new application.  
The `manifest.json` file contains simple data structures, that expose the application details to the `EMBL-EBI Cloud Portal`.
