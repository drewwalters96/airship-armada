..
  Copyright 2018 AT&T Intellectual Property.
  All Rights Reserved.

  Licensed under the Apache License, Version 2.0 (the "License"); you may
  not use this file except in compliance with the License. You may obtain
  a copy of the License at

      http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
  WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
  License for the specific language governing permissions and limitations
  under the License.

.. _armada-documents:

Armada Documents
================

Armada consumes three types of documents to manage Kubernetes workloads:
:ref:`Charts`, :ref:`Chart Groups`, and :ref:`Manifests`.

.. _Charts:

Charts
------

Charts consist of the smallest building blocks in Armada. An Armada ``Chart``, while
comparable to a `Helm chart`_, is a document type that consists of the labels,
dependencies, source, and additional information needed to deploy a `Helm
chart`_ with Armada.

Basic Options
~~~~~~~~~~~~~~

A minimal Armada chart document has the following structure:

.. codeblock:: yaml

    ---
    schema: armada/Chart/v1
    metadata:
      schema: metadata/Document/v1
      name: blog-1
    data:
      chart_name: blog-1
      release: blog-1
      namespace: default
      wait:
        timeout: 100
        labels:
          component: blog
      install:
        no_hooks: false
      upgrade:
        no_hooks: false
      values: {}
      source:
        type: git
        location: https://github.com/namespace/repo
        subpath: .
        reference: master
      dependencies: []
    ...

+------------+----------+------------+----------------------------------------+
| Property   |          | Type       | Description                            |
+------------+----------+------------+----------------------------------------+
| chart_name | required | string     | An arbitrary name used to identify a   |
|            |          |            | chart in logging and error messages.   |
+------------+----------+------------+----------------------------------------+
| release    | required | string     | A `Helm release name`_.                |
+------------+----------+------------+----------------------------------------+
| namespace  | required | string     | A Kubernetes namespace to which Armada |
|            |          |            | will deploy a chart.                   |
+------------+----------+------------+----------------------------------------+
| wait       | optional | object     | Options Armada will use to wait on a   |
|            |          |            | resource or resources after            |
|            |          |            | installation or upgrade of a chart.    |
|            |          |            | See `wait`_.                           |
+------------+----------+------------+----------------------------------------+
| install    | required | object     | Installation options and hooks. See    |
|            |          |            | `install`_.                            |
+------------+----------+------------+----------------------------------------+
| upgrade    | required | object     | Upgrade options and hooks. See         |
|            |          |            | `upgrade`_.                            |
+------------+----------+------------+----------------------------------------+
| values     | required | dictionary | `Helm chart values`_.                  |
+------------+----------+------------+----------------------------------------+
| source     | required | object     | Information Armada will use to         |
|            |          |            | retrieve the source of a chart.        |
+------------+----------+------------+----------------------------------------+

.. _wait::

Armada uses wait options to wait on resources after the installation or
upgrade of a chart. Wait operations are critical to the successful deployment
of a chart; timeouts that occur as the result of a failed wait operation will
cause deployment of a chart to fail.

An example of a wait configuration:

.. codeblock:: yaml

    wait:
      timeout: 300
      resources:
        - type: pod
          labels:
            component: api
        - type: daemonset
          labels:
            component: logger
          min_ready: 1
      labels:
        application: example-app

.. NOTE::

    Chart deployment is successful when all resources defined in a wait
    configuration are ready.

+------------+----------+------------+----------------------------------------+
| Property   |          | Type       | Description                            |
+------------+----------+------------+----------------------------------------+
| timeout    | optional | int        | Time in which an Armada will wait for  |
|            |          |            | a chart to successfully deploy.        |
+------------+----------+------------+----------------------------------------+
| resources  | optional | array      | Resources which Armada will wait to    |
|            |          |            | enter a success state. See             |
|            |          |            | `resources`_.                          |
+------------+----------+------------+----------------------------------------+
| labels     | optional | dictionary | Labels Armada will use to identify     |
|            |          |            | pods and jobs during wait operations.  |
|            |          |            | Additionally, these labels will be     |
|            |          |            | used to identify all resources defined |
|            |          |            | in the resources array above.          |
+------------+----------+------------+----------------------------------------+
| native     | optional | boolean    | Use Helm's native wait feature with    |
|            |          |            | timeout above. All resources and       |
|            |          |            | labels are ignored.                    |
+------------+----------+------------+----------------------------------------+

.. NOTE::

    Helm's native wait feature waits for all Pods, PVCs, Services, before
    marking a deployed release as successful. Failed release deployments using
    the native Helm wait feature will result in a GRPC error from Tiller.

.. _resources:

+------------+----------+------------+----------------------------------------+
| Property   |          | Type       | Description                            |
+------------+----------+------------+----------------------------------------+
| type       | required | string     | A Kubernetes resource type. (e.g. pod, |
|            |          |            | job, daemonset, statefulset).          |
+------------+----------+------------+----------------------------------------+
| labels     | optional | dictionary | Labels Armada will use to wait on a    |
|            |          |            | defined resource.                      |
+------------+----------+------------+----------------------------------------+
| min_ready  | optional | int        | A minimum number of pods that must be  |
|            |          |            | ready for a successful wait operation  |
|            |          |            | involving a controller resource. Only  |
|            |          |            | supported by controller types (i.e..   |
|            |          |            | deployment, daemonset, statefulset).   |
+------------+----------+------------+----------------------------------------+


.. _Chart Groups:

Chart Groups
------------

A ``Chart Group`` document is a grouping of related charts that assigns the
order of their deployment.

A ``Chart Group`` has the following structure:

.. codeblock:: yaml

    ---
    schema: armada/ChartGroup/v1
    metadata:
      schema: metadata/Document/v1
      name: chart-group-name
    data:
      description: Deploys Simple Service
      sequenced: False
      chart_group:
        - chart-name
        - chart-name

+-------------+----------+------------+---------------------------------------+
| Property    |          | Type       | Description                           |
+-------------+----------+------------+---------------------------------------+
| name        | required | string     | Name of the chart group.              |
+-------------+----------+------------+---------------------------------------+
| description | optional | string     | Description of the chart group.       |
+-------------+----------+------------+---------------------------------------+
| sequenced   | optional | boolean    | Option to deploy the charts belonging |
|             |          |            | to a chart group in parallel (i.e. at |
|             |          |            | the same time) or in sequence (i.e.   |
|             |          |            | one at a time). Default is False.     |
+-------------+----------+------------+---------------------------------------+
| chart_group | required | array      | Array of chart document names that    |
|             |          |            | belong to the chart group.            |
+-------------+----------+------------+---------------------------------------+

.. _Manifests:

Manifests
---------

A ``Manifest`` document is the largest building block in Armada. ``Manifest``
documents group chart groups for a single deployment.

An Armada ``Manifest`` has the following structure:

.. codeblock:: yaml

   ---
   schema: armada/Manifest/v1
   metadata:
     schema: metadata/Document/v1
     name: manifest-name
   data:
     release_prefix: armada
     chart_groups:
       - chart_group

+----------------+----------+------------+------------------------------------+
| Property       |          | Type       | Description                        |
+----------------+----------+------------+------------------------------------+
| release_prefix | required | string     | A prefix that is appended to the   |
|                |          |            | release name of every chart.       |
+----------------+----------+------------+------------------------------------+
| chart_groups   | required | array      | Array of chart_groups deployed     |
|                |          |            | with a manifest.                   |
+----------------+----------+------------+------------------------------------+

Document Validation
-------------------

Introduction
^^^^^^^^^^^^

All documents consumed by Armada are validated against `Deckhand DataSchemas`_,
JSON schemas that contains additional metadata for use with Deckhand's
`layering`_ and `substitution`.

.. _Deckhand DataSchemas: https://airship-deckhand.readthedocs.io/en/latest/document-types.html?highlight=dataschema#dataschema
.. _Helm charts: https://docs.helm.sh/developing_charts/
.. _layering: https://airship-deckhand.readthedocs.io/en/latest/layering.html
.. _substitution: https://airship-deckhand.readthedocs.io/en/latest/substitution.html

Schemas
^^^^^^^

* ``Chart`` schema.

  JSON schema against which all documents with ``armada/Chart/v1``
  ``metadata.name`` are validated.

  .. literalinclude::
    ../../../armada/schemas/armada-chart-schema.yaml
    :language: yaml
    :lines: 15-
    :caption: Schema for ``armada/Chart/v1`` documents.

  This schema is used to sanity-check all ``Chart`` documents that are passed
  to Armada.

* ``Chart Group`` schema.

  JSON schema against which all documents with ``armada/Chart/v1``
  ``metadata.name`` are validated.

  .. literalinclude::
    ../../../armada/schemas/armada-chartgroup-schema.yaml
    :language: yaml
    :lines: 15-
    :caption: Schema for ``armada/ChartGroup/v1`` documents.

  This schema is used to sanity-check all ``Chart Group`` documents that are
  passed to Armada.

* ``Manifest`` schema.

  JSON schema against which all documents with ``armada/Manifest/v1``
  ``metadata.name`` are validated.

  .. literalinclude::
    ../../../armada/schemas/armada-manifest-schema.yaml
    :language: yaml
    :lines: 15-
    :caption: Schema for ``armada/Manifest/v1`` documents.

  This schema is used to sanity-check all ``Manifest`` documents that are passed
  to Armada.

.. _authoring-guidelines:

Authoring Guidelines
--------------------

Introduction
^^^^^^^^^^^^

Authoring Armada documents requires careful consideration. This guide seeks to
provide as many considerations as possible for authoring production-grade
deployment documents.

Structure
^^^^^^^^^

Armada consumes YAML files containing ``Manifest`, ``Chart Group``, and
``Chart`` documents. We recommend authoring your Armada documents in the
following order:

1. Charts
2. Chart Groups
3. Manifests

When authoring each document, we recommend copying the entirety structure of
the document outlined in this section and tuning each document to your
deployment requirements. Please make note of deprecations, as they **will** be
removed in future releases.

Authoring Charts
^^^^^^^^^^^^^^^^

Copy the following chart structure:

.. codeblock:: yaml

---
schema: armada/Chart/v1
metadata:
  schema: metadata/Document/v1
  name: example-chart
data:
  release: example-release
  chart_name: example-chart
  namespace: default
  values: {}
  test:
    enabled: True
    options:
      cleanup: False
  wait:
    timeout: 300
    resources: []
    labels: {}
    native: False
  source:
    type: git
    location: https://git.openstack.org/namespace/repository.git
    subpath: chart-dir
    reference: 307f1318c4e83f247f1e3838478957e7555d6ce0
  upgrade:
    no_hooks: False
      pre: []
      options:
        force: False
        recreate_pods: False
          type: object
...

Metadata
~~~~~~~~

Add a name for the Armada chart document. This name is used by Armada to
identify the chart in logging/error messages as well as ``Chart Group``
documents.

Data
~~~~

Several data points are meant to be straightforward and can be populated (e.g.
``release``, ``namespace``, ``chart_name``). 

Values
~~~~~~

Armada ``Chart`` documents support a ``dictionary`` of Helm chart values.
Adding chart values to this section is the equivalent of providing a values
file to Helm and will override values present in a Helm chart.

.. codeblock:: yaml

    values:
      ports:
        http: 80
        https: 81

Protected Charts
~~~~~~~~~~~~~~~~

Declaring a chart as ``protected`` will prohibit Armada from purging its
release if it enters a ``FAILED`` state during deployment. To mark a chart as
``protected`` include the following dictionary in your chart:

.. codeblock:: yaml

    protected:
      continue_processing: True

By default, if Armada encounters a protected chart in a ``FAILED`` state, it
will bypass the chart and continue with the deployment process. You can prevent
this behavior by disabling the ``continue_processing`` option.

Testing Charts
~~~~~~~~~~~~~~

By default, Armada will run Helm tests on deployed resources and not delete
test pods. Helm test behavior can be customized by adding the following object
to your chart:

.. codeblock:: yaml

    test:
      enabled: True
      options:
        cleanup: False

Waiting for resources
~~~~~~~~~~~~~~~~~~~~~

Armada will wait for any jobs and pods with labels matching the ``wait.labels``
dictionary by default.

.. DANGER::

    Failing to specify labels can cause unintended consequences during
    deployment, as Armada will wait for **all** resources to enter a ``READY``
    state before declaring a deployent successful.

An example of a simple wait configuration:

.. codeblock:: yaml

    wait:
      timeout: 300
      labels:
        application: blog
        component: database

Armada also offers fine-grained control over the wait process by specifically
declaring the resources you intend to wait for.

For more information regarding resource declaration in your wait configuration,
see `wait`_.

Source
~~~~~~

Armada supports deployment of charts using several types of resources. For
more information regarding alternative source types, see `source`_.

.. _Helm chart: https://docs.helm.sh/developing_charts/
