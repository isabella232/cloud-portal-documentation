.. _api_endpoints:

API Endpoint documentation
^^^^^^^^^^^^^^^^^^^^^^^^^^

Create or Update Deployed Application Outputs
  A 'PUT' request to create or update application outputs/results.

Example request
::

  PUT /deployment/$PORTAL_DEPLOYMENT_REFERENCE/outputs HTTP/1.1
  Content-Type: application/json;charset=UTF-8
  Deployment-Secret : $PORTAL_CALLBACK_SECRET
  Host: $PORTAL_BASE_URL
  Body:
  [{"outputName":"internal ip","generatedValue":"192.168.3.14"},
  {"outputName":"startTime","generatedValue":"2019-01-11 13:45"}]

Example response
::

   HTTP/1.1 204 No content                                        |


Stop Deployed Application
  A 'PUT' request to stop/destroy deployed application.

Example request
::

   PUT /deployment/$PORTAL_DEPLOYMENT_REFERENCE/stopme HTTP/1.1
   Deployment-Secret : $PORTAL_CALLBACK_SECRET
   Host: $PORTAL_BASE_URL


Example response
::

   HTTP/1.1 200 OK


.. note::
  For more information about environmental variables
  $PORTAL_DEPLOYMENT_REFERENCE,$PORTAL_BASE_URL,
  $PORTAL_CALLBACK_SECRET see :ref:`Deployment variables <deployment-variables>`.
