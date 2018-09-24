.. _using-the-portal:

Using the |project_name|
========================

The |portal_link| is offered as a service to |elixir| users as well as users coming from other research communities and institutions.

How to access the |portal_link|
----------------------------------------------------------------------

The |project_name| is available at the address https://cloud-portal.ebi.ac.uk.

At the moment, users needs to sign-up for an |elixir| account to access the |portal_link|. You can easily obtain your |elixir| identity for free at the
`ELIXIR sign-up page <https://www.elixir-europe.org/register>`_, and go to the |project_name| login page at |portal_link|/signon.

.. _`Cloud Profile`:

Setting up up your Cloud Profile
-----------------------------------------------------------------------

Your Cloud Profile contains all the information required to deploy your applications to a given cloud provider. It's subdivided in three sections: `Configurations`_ , `Cloud Credentials`_, and
`Deployment Parameters`_. `Cloud Credentials`_ are combined with a set of `Deployment Parameters`_ and a SSH public key to give the `Configuration`_ required to deploy a certain
application in the cloud provider of choice.

You can access your current `Cloud Profile`_ `here <|portal_link|/profile>`__ (You need to `log-in <|portal_link|/welcome/login>`_ first!)

.. _cloud-credentials:

Cloud Credentials
~~~~~~~~~~~~~~~~~
Interacting with clouds, being them private or public, usually requires authentication. Many of the tools used to interact with cloud APIs rely on environment
variables to source authentication tokens, with the clear advantage of not requiring credentials to lie in some file with your filesystem. |terraform|, the open-source
tools that the |portal_link| uses to provision the required virtual infrastructure, is not an exception to this.

Each cloud provider requires a set of *cloud credentials*. While it's possible somebody has *shared* some credentials with you already via a **Team**, you can find the instructions
to insert you personal `Cloud Credentials`_ below.

Amazon Web Service (AWS)
^^^^^^^^^^^^^^^^^^^^^^^^

Add your `Cloud Credentials`_ like this:

+--------------------------------------+---------------------+
| Key                                  | Value               |
+======================================+=====================+
| ``AWS_ACCESS_KEY_ID``                | ``YOUR_ACCESS_KEY`` |
+--------------------------------------+---------------------+
| ``AWS_SECRET_ACCESS_KEY``            | ``YOUR_SECRED_KEY`` |
+--------------------------------------+---------------------+
| ``AWS_DEFAULT_REGION`` *(Optional)*  | ``YOUR_AWS_REGION`` |
+--------------------------------------+---------------------+

You can find this information in you `AWS <https://aws.amazon.com>`_ user page
under the section ``IAM`` —> ``USERS`` (or via the official `docs <https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html>`_). ``AWS_DEFAULT_REGION`` is *optional* and can be omitted if the
region is selected in another way (i.e. via a |terraform| variable specificied in the `Deployment Parameters`_). A list of the
AWS regions can be found `here <https://docs.aws.amazon.com/general/latest/gr/rande.html>`__.

Azure
^^^^^

Add your ``Cloud Credentials`` like this:

+-------------------------+--------------------------+
| Key                     | Value                    |
+=========================+==========================+
| ``ARM_SUBSCRIPTION_ID`` | ``YOUR_SUBSCRIPTION_ID`` |
+-------------------------+--------------------------+
| ``ARM_CLIENT_ID``       | ``YOUR_CLIENT_ID``       |
+-------------------------+--------------------------+
| ``ARM_CLIENT_SECRET``   | ``YOUR_CLIENT_SECRET``   |
+-------------------------+--------------------------+
| ``ARM_TENANT_ID``       | ``YOUR_TENANT_ID``       |
+-------------------------+--------------------------+
| ``ARM_ENVIRONMENT``     | ``public``               |
+-------------------------+--------------------------+

In order to use Terraform with Azure it is required to create a ``Service Principal`` via the Azure portal.
The Terraform documentation provide an extensive explanation on how to obtain this in its `official documentation <https://www.terraform.io/docs/providers/azurerm/authenticating_via_service_principal.html#creating-a-service-principal-in-the-azure-portal>`_.

OpenStack
^^^^^^^^^

Add your ``Cloud Credentials`` like this:

+-----------------------------+-------------------------------------------+
| Key                         | Value                                     |
+=============================+===========================================+
| ``OS_AUTH_URL``             | ``YOUR_AUTH_URL``                         |
+-----------------------------+-------------------------------------------+
| ``OS_USERNAME``             | ``YOUR_USERNAME``                         |
+-----------------------------+-------------------------------------------+
| ``OS_PASSWORD``             | ``YOUR_PASSWORD``                         |
+-----------------------------+-------------------------------------------+
| ``OS_TENANT_ID``            | ``YOUR_TENANT_ID``                        |
+-----------------------------+-------------------------------------------+
| ``OS_TENANT_NAME``          | ``YOUR_TENANT_NAME``                      |
+-----------------------------+-------------------------------------------+
| ``OS_REGION_NAME``          | ``YOUR_REGION_NAME``                      |
+-----------------------------+-------------------------------------------+


The specific set of values for your OpenStack provider might be slightly different and is contained in the
``OpenStack RC file``, which is project-specific and contains the credentials used by all the OpenStack services.

You can download the ``OpenStack RC file`` file from the OpenStack
dashboard as an administrative user or any other user.

1. Log in to the OpenStack dashboard.
2. Choose the ``Project`` for which you want to download the OpenStack
   RC file
3. Click ``Compute`` —> ``API Access``. (In older version of OpenStack
   click ``Compute`` —>``Access & Security``)
4. Click ``Download OpenStack RC File`` and save the file.

Google Cloud Platform (GCP)
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Authenticating with Google Cloud services requires the set up of the ``GOOGLE_CREDENTIALS`` variable, as described
in the official `GCP documentation <https://developers.google.com/identity/protocols/application-default-credentials#howtheywork>`_.

While locally ``GOOGLE_CREDENTIALS`` can simply point to a JSON file containing your access keys, in the |project_name| you're required
to upload the JSON string directly.

The JSON file can be downloaded directly from the `Google Developers Console <https://console.developers.google.com/>`_ following these steps:

1. Log into the `Google Developers Console <https://console.developers.google.com/>`__ and select a
   ``project``.
2. The API Manager view should be selected, click on ``Credentials`` on
   the left, then ``Create credentials``, and finally “Service account
   key”.
3. Select ``Compute Engine default service account`` in the
   ``Service account`` drop-down, and select ``JSON`` as the key type.
4. Clicking ``Create`` will download your ``credentials``.

Once you have your credentials you can add them to ``Cloud Credentials``
in this form:

+-----------------------------------+--------------------------------------+
| Key                               | Value                                |
+===================================+======================================+
| ``GOOGLE_CREDENTIALS``            | ``{ "type":"service_account", [..]}``|
+-----------------------------------+--------------------------------------+

.. _`Deployment Parameters`:

Deployment parameters
~~~~~~~~~~~~~~~~~~~~~

``Deployment parameters`` represent a set of inputs specific that are related to
the cloud provider and eventually to a specific application. In general, they
provide information about the shared instances that you can have in place in
your cloud provider or just information that you prefer to set up just the first
time and avoid to repeat every time you deploy the instance.

The deployment parameters required by an appliance are expressed in the
documentation page of the git repository of the same appliance.

For your convenience you can use a single ``deployment parameter``
configuration for different appliances: it will make use only of the
share inputs ignoring the ones that are not relevant.
A deployment parameter can also be used to overwrite any of the
variables defined in the ``terraform.tfvars`` file even when it is not
reported as input in the ``manifest`` file.


.. _`Configuration`:

Configurations
~~~~~~~~~~~~~~

Configurations represent a way to link a set of `Cloud Credentials`_ with a set
of `Deployment Parameters`_ and an SSH public key. The use of a configuration
simplifies the deployment of the applications, allowing to store and reuse as
much configuration as possible.

Specify a new configuration is very easy:

-  click on the ``+`` button;
-  assign a name of your choice;
-  choose one of the ``Cloud Provider`` that you have previously
   defined;
-  choose one of the ``Deployment parameters`` that you have previously
   defined;
-  (optionally) add a public SSH key.


.. _inputs:

Inputs
------

``Inputs`` parameters represent a set of parameters that are likely going to
change per deployment and thus cannot determined in advance. The best example
of this is the number of nodes a compute cluster you're about to deploy will
need to have.

Inputs can also refer to variables defined in `Deployment Parameters`_, and thus
allow to override them only when required.

Managing the Registry
--------------------------------

How to add an Application to the Registry
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Adding a new Application is very simple: you just need to know the
URL of the git repository where the Applications is stored. As a test, you can
add one of the applications maintained by the TSI team:
https://github.com/EMBL-EBI-TSI/cpa-instance

Starting from the |project_name| Home:
- Click `Application Repository <|portal_link|/repository>`_ in
the menu on the left-hand side;
- Click on the ``+`` button;
- Enter the ``URL`` of the git repository;
- Click ``Add``.

Your new application is now included in your Repository!

Applications compliance
~~~~~~~~~~~~~~~~~~~~~~~

The ``EMBL-EBI Cloud Portal`` requires the presence of a well-formed
Manifest file in the root directory of each git repository containing
an Application. This file is is a simple ``JSON`` file

The ``manifest.json`` file contains a simple dictinary specifying, for example,
the Application name and mainainer along with the supported Cloud Providers.
Trying to add an Application repository that does not contain - or contains a
malformed manifest file, will result in an error.

Programmatic access (ecp-cli)
--------------------------------
To facilitate automated deployment and application tested, a command line tool is available to interact with the portal: [ecp-cli](https://github.com/EMBL-EBI-TSI/ecp-cli).

To use, clone the repo; ecp.py is the main executable. First login by running

``ecp.py login``

This will take you through the Elixir login process that will authenticate you through the portal. Your token will be stored automatically. Next, you can e.g. list your applications by running:

``ecp.py get deployments``

The general syntax is `ecp.py *verb* *resource* [*name*]`, where verb is one of ``get`` ``create`` ``delete`` or ``stop`` (deployments only) and resource is one of ``cred`` ``param`` ``config`` ``app`` ``deployment`` ``logs`` ``destroylogs`` or ``status``.

For more examples see the repo README.
