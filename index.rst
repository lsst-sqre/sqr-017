:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. Add content below. Do not include the document title.

.. note::

   **This technote is not yet published.**

   This technote describes the design of the Metric and Specification system that the new validation framework will use.

Frossie's design principles
===========================

* Metric should be defined independently of code.
  Client-side code should consume metric definitions., and so will server-side code.
* Implement as a thin shim over what will become a Butler put.
* UX for Metric developers: make it as easy as possible for a developer to define a new ``Metric`` and ``Measurements`` appear in SQUASH and be monitored.

The validate_base 2.0 data model
================================

The validation framework provides a rich set of objects for developers to use.
The core objects are metrics, specifications, measurements, and monitors.
Their relationships are summarized as:

| *Metrics* are *measured*.
| Measurements are tested by *specifications*.
| Specification tests trigger *monitors.*
|

The full list of validation framework objects is:

- MetricRepo
- :ref:`MetricSet <metricset>`
- :ref:`Metric <metric>`
- :ref:`MeasurementSet <measurementSet>`
- :ref:`Measurement <measurement>`
- :ref:`Blob <blob>`
- :ref:`Job <job>`
- :ref:`Provenance <provenance>`
- :ref:`Specification <specification>`
- Monitor
- :ref:`MeasurementView <measurementview>`


Afterburner
-----------

An ``afterburner`` is a piece of code that is executed on the output of a ``Task`` in order to calculate a ``Measurement`` of a ``Metric``.

.. _metricset:

MetricSet
---------

A ``MetricSet`` is a collection of ``Metrics`` and their associated metadata.

Attributes
^^^^^^^^^^

* name - ``jointcal_cfht_r``
* eups package name of data
* directory within eups data package for dataset
* butler dataId or list of dataIds (optional)
* Metrics (list?)

.. _metric:

Metric
------

A ``Metric`` is a quantity that is measureable.

Prior art
^^^^^^^^^

* ``lsst.validate.base.Metric``
* ``Metric.yaml format``

Attributes
^^^^^^^^^^

* ``name``
* ``description``
* ``unit``
* ``tags``
* ``reference.doc``
* ``reference.page``
* ``reference.url``

Questions & Notes
^^^^^^^^^^^^^^^^^

* ``Specifications`` are no longer contained by ``Metrics``.
* In the existing ``lsst.validate.base.Metric``, there is a parameters dictionary that defines constants for the Measurement code. For example, the annulus diameter from AMx Metrics. These parameters will be contained in the ``Specifications``.
* We talked about making the minimum ``Provenance`` required for a ``Measurement``/``Job`` being defined in the ``Metric``. Is this still a requirement?

.. _measurementSet:

MeasurementSet
--------------

A ``MeasurementSet`` is a collection of measurements, and their associated metadata.

Attributes
^^^^^^^^^^
_
* ``name`` — Name of the ``MetricSet`` that these ``Measurements`` are associated with (e.g. ``validate_drp``)
* ``job_id`` — Job identifier
* ``provenance`` — provenance of the executed job (*TODO: should this actually live in the job itself?*)
* ``measurements`` — dictionary of name: ``Measurement``.

.. _measurement:

Measurement
-----------

A ``Measurement`` is a realization of a ``Metric``: always a scalar value.

Prior art
^^^^^^^^^

* ``lsst.validate.base.MeasurementBase``

Attributes
^^^^^^^^^^

* ``name`` — Name of metric that this measured.
* ``value`` — scalar Measurement value, required to be persisted in units of ``Metric.unit`` as an ``astropy.Quantity``.

Questions & Notes
^^^^^^^^^^^^^^^^^

* ``lsst.validate.base.MeasurementBase`` originally had a parameters attribute that provided Provenance for how the Measurement was made (e.g., a S/N cut-off for star selection). These will now be part of the Task configuration, and available through the regular Provenance.
* ``lsst.validate.base.MeasurementBase`` also had an extras attribute where additional Measurement outputs could be persisted (JSON serializable). Do we still want this? Or do we always want such data to go into a Blob?

.. _blob:

Blob
----

A ``blob`` is a container for JSON-serializable data that may be associated with one or more Measurements that might be useful for rendering plots and doing SQUASH-side analysis.

Prior art
^^^^^^^^^

* ``lsst.validate.base.BlobBase``

.. _job:

Job
---

A ``Job`` is an execution of a pipeline, containing ``Measurements``, ``blobs``, and their ``Provenance``.

Prior art
^^^^^^^^^

* ``lsst.validate.base.Job``

Attributes
^^^^^^^^^^

* ``Measurements`` — list of Measurement objects (``TODO: or a MeasurementSet?``).
* ``blobs`` — list of Blob objects. Each blob should be reference by at least one * Measurement.
* ``Provenance`` — data structure that fully specifies the Provenance of the pipeline run.

Questions & Notes
^^^^^^^^^^^^^^^^^^^

* What is the schema of ``Provenance``? At minimum, it includes the input dataIds (input dataset) and task configurations.
* Not all ``Provenance`` is currently known within the pipeline. We use post-qa to hydrate Job Provenance with package versions and Jenkins environment variables. However, working towards a state where post-qa is no longer used as a shim, it's not unreasonable to move this into validate_base.

.. _provenance:

Provenance
----------

All metadata associated with this ``Job`` run, including Config parameters, Butler dataRefs, cluster configuration, etc.

Questions & Notes
^^^^^^^^^^^^^^^^^

* How is provenance defined?
* How do we define queries on provenance in a ``Specification``?
* How do we map between this provenance and the one that DAX will maintain?

.. _specification:

Specification
-------------

A ``Specification`` is a binary (pass/fail) evaluation of a ``Measurement`` of a ``Metric``. There can be an arbitrary number of ``Specifications`` associated with a ``Metric``.

Attributes
^^^^^^^^^^

* ``name`` — Identifier of the ``Metric`` that this Specification is attached to.
* ``provenance_query`` — only ``Measurements`` that have matching ``Provenance`` parameters are tested by this ``Specification``.
* ``parameters`` - A dict of key:value pairs that must be matched by the ``Job``'s ``Provenance`` regarding particular values used in a calculation (e.g. diameter used for aperture photometry).
* ``alert_listeners`` - Slack IDs of people who are alerted if a ``Measurement`` fails the ``Specification``.
* ``alert_channels`` - Slack Channel IDs that recieve messages when a ``Measurement`` fails a ``Specification``.
* ``threshold`` and comparison_operator — ``Measurement`` passes ``Specification`` if ``Measurement`` is on the side of the threshold indicated by the comparison operator.
* ``range`` — ``Measurement`` passes ``Specification`` if Measurement is within this range (new).

Questions & Notes
^^^^^^^^^^^^^^^^^

* Either threshold or range can be set. Possibly there should be different classes of Specification (i.e., a ThresholdSpecification or a RangeSpecification).
* Note that we're jettisoning some of the earlier ``Specification`` class baggage, like dependencies. This means that the definitions of Metrics are no longer driven by definitions of Specifications, as they currently are for AFx/ADx, for example. Instead, this flexibility is handled by additional Metrics.
* Should the ``Parameters`` just be part of the ``Provenance``, or should they be a separate section for maintanence convinience and get ingested into the ``Provenance``?

.. _measurementview:

MeasurementView
---------------

A MeasurementView is a collection of Measurements for a Metric, possibly filtered by Provenance. A MeasurementView can be used to populate a Measurement timeseries (regression plot), as seen in SQUASH. A MeasurementView is essentially a DB query, but provides a more concrete API for us to think about how we can do data science against Measurements.

Attributes
^^^^^^^^^^

* ``Metric_name``
* ``Provenance_query``

.. _validate-metrics:

validate_metrics: A package for metric and specification definitions
====================================================================

All packages that make metric measurements define those metrics as YAML files in the ``validate_metrics`` package.
Likewise, all specifications for these metrics are also centrally defined in YAML files committed to ``validate_metrics``.
This design is appealing because SQUASH infrastructure can watch the ``validate_metrics`` repository and populate its DB from ``validate_metrics`` as a single source of truth.
``validate_metrics`` effectively becomes a user interface for package developers and test engineers to configure the testing system.

``validate_metrics`` is designed to be a data-only package (though it still provides a version in Python, ``lsst.validate.metrics.__version__``)
``validate_base`` provides Python access to metrics and specifications.

Within ``validate_metrics``, developers work in two directories:

- ``/metrics`` hosts *metric definition* YAML files.
  For each Stack package that generates metric measurements there is a metric definition file named after that package.
  For example:

  .. code-block:: text

     metrics/
       validate_drp.yaml
       jointcal.yaml

  The format of these YAML files is :ref:`described below <metric-yaml>`.
  We expect these metric definitions to be slow moving, and only change when a new metric measurement is coded into a Stack package.

- ``/specs`` hosts *specification definition*.
  These YAML files are organized into sub-directories named after the metric YAML file, but otherwise the names of specification YAML files has no programmatic meaning.
  For example:

  .. code-block:: text

     specs/
       validate_drp/
         LPM-17.yaml
         cfht_gri.yaml

  In this example, official specifications defined in :lpm:`17`, the Science Requirements Document, are coded in ``LPM-17.yaml``.
  This specification file would remain static, while developers would typically add custom, ad-hoc, specifications in other files, like ``cfht_gri.yaml``.
  The format of specification YAML files is :ref:`described below <spec-yaml>`.

.. _metric-yaml:

Metric YAML format
------------------

This is an example of a PA1 metric encoded in ``validate_metrics/metrics/validate_drp.yaml``:

.. code-block:: yaml

   PA1:
     description: >
       The maximum rms of the unresolved source magnitude distribution around the mean value.
     unit: mmag
     reference:
       doc: LPM-17
       url: http://ls.st/lpm-17
       page: 21

The root level of a metric YAML file is an *associative array* (equivalent to a Python dict) where *keys* are metric names.
In the above example, only ``PA`` is shown, but it might be followed by other metrics like ``PF1`` and ``PA2``.

A metric definition itself is minimal, consisting of only three fields:

- ``description``: a sentence, or even multiple paragraphs, that describe the metric.
  This description is consumed by the Science Pipelines documentation, and also shown by SQUASH.

- ``unit``: the string representation of the :py:obj:`astropy.units.Unit` that measurements of a metric are made in.
  Unitless metrics (a count, for example), have units written as ``---``.
  Percentages can be written as ``%``.
  Fractions are not supported by `astropy.unit` so fractional metrics must be rephrased as percentages.

- ``reference``: this field points to further documentation where a metric is formally defined.
  Provide ``doc``, ``url``, and ``page`` fields as appropriate.

Fully qualified metric name
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Metrics can be referenced universally by their *fully qualified name*:

.. code-block:: text

   { package name }.{ metric name }

For example, the fully qualified name for the example metric is ``validate_drp.PA1``.

When working inside a package, where context is clear, the validate API can permit metrics to be addressed by name alone, ``PA1``.

.. _spec-yaml:

Specification YAML format
-------------------------

A complete specification looks like:

.. code-block:: yaml

   ---
   metric: 'PA1'
   name: 'design_gri'
   threshold:
     value: 5.0
     unit: '%'
     operator: '<='
   provenance_query:
     filter: ['g', 'r', 'I']
   ...

Notice that each specification is encapsulated within a corresponding YAML document (which are divided by ``---`` tokens).
There is always one specification per YAML document.
This architecture allows us to spread specifications across many YAML files in the ``validate_metrics`` repository, and permit specifications to reference each other (see :ref:`partials <spec-partials-yaml>` and :ref:`inheritance <spec-inheritance-yaml>`, below).

The fields of a specification are:

- ``metric``: the name of the metric this specification applies to.
  Since specifications are encapsulated by package, there is no need to use the fully-qualified metric name.

- ``name``: the name of this metric.

  Specifications extend the naming system of metrics.
  The *fully qualified name* of this specification is ``validate_drp.PA1.design_gri`` (assuming the specification is defined in ``/specs/validate_drp/``).

- ``threshold``: this is a test against a measurement.
  A measurement *passes* a specification test if this statement evaluates to true:

  .. code-block:: text

     { measurement value } { operator } { threshold value }

  Other test formats are available for specifications.
  See :ref:`below <spec-test-yaml>`.

- ``provenance_query`` field is an associative array (dictionary) of query terms for measurement provenance that this specification can be applied to.
  The query language is currently undefined, so the example is a pseudocode query where the ``filter`` must be *one of* ``g``, ``r`` or ``i``.

.. _spec-test-yaml:

Specification tests
^^^^^^^^^^^^^^^^^^^

The binary comparison test is quite common, but its not the *only* imaginable test structure.
Other types of tests that may be supported by the validation framework are:

- ``tolerance``: consisting of a target value, and a symmetric tolerance window.

- ``window``: test if a measurement deviates from the sample of previous measurements in a given window, by a given amount.

- ``function``: specifies an importable Python function that computes a binary True (pass) or False (fails) result.

.. _spec-partials-yaml:

Specification partials
^^^^^^^^^^^^^^^^^^^^^^

Specifications might repeat information.
For example, a ``provenance_query`` for a certain test dataset.
We apply DRY design principles though *partials*.
A partial has an ``id`` field, and can't be a specification on its own.
For example:

.. code-block:: yaml

   ---
   # specification partial
   id: 'base'
   metric: 'PA1'
   threshold:
     unit: ''
     operator: '<='
   provenance_query:
     filter: ['g', 'r', 'I']

   ---
   # design specification instance that mixes in the base partial
   # validate_drp.PA1.design
   name: 'design'
   base: '#base'
   threshold:
     value: 5.0

   ---
   # stretch specification instance that mixes in the base partial
   # validate_drp.PA1.stretch
   name: 'stretch'
   base: '#base'
   threshold:
     value: 3.0
   ...

A partial can be referenced from the ``base`` field by prefixing the ``id`` with ``#``.
Partials can also be referenced from across files (but within the same package's ``specs`` directory) by providing a filename:

.. code-block:: yaml

   base: "cfht_gri#base"

.. _spec-inheritance-yaml:

Specification inheritance
^^^^^^^^^^^^^^^^^^^^^^^^^

Specifications can also inherit from specifications; generally to add partials.
Specifications are referenced through their fully qualified name ``validate_drp.PA1.design_gri``, or the package-relative fully qualified name, ``PA1.design_gri``.
For example:

.. code-block:: yaml

   ---
   # Specification partial
   id: 'PA1-base'
   metric: 'PA1'
   threshold:
     unit: 'mmag'
     operator: "<="

   ---
   # validate_drp.PA1.minimum_gri
   name: "minimum_gri"
   base: "#PA1-base"
   threshold:
     value: 8.0

   ---
   # Partial that queries a cfht_gri dataset
   id: 'cfht-base'
   provenance_query:
     dataset_repo_url: 'https://github.com/lsst/validation_data_cfht.git'
     filters: ['g', 'r', 'i']
     visits: [849375, 850587]
     ccd: [12, 13, 14, 21, 22, 23]

   ---
   # validate_drp.PA1.cfht_minimum_gri
   name: 'cfht_minimum_gri'
   base: ['PA1.minimum_gri', '#cfht-base']
   ...

The fully-hydrated ``validate_drp.PA1.minimum_gri`` specification is:

.. code-block:: yaml

   ---
   name: 'minimum_gri'
   metric: 'PA1'
   threshold:
     value: 8.0
     unit: 'mmag'
     operator: "<="

And the fully-hydrated ``validate_drp.PA1.cfht_minimum_gri`` specification is:

.. code-block:: yaml

   name: 'cfht_minimum_gri'
   metric: 'PA1'
   provenance_query:
     dataset_repo_url: 'https://github.com/lsst/validation_data_cfht.git'
     filters: ['g', 'r', 'i']
     visits: [849375, 850587]
     ccd: [12, 13, 14, 21, 22, 23]
   threshold:
     value: 8.0
     unit: 'mmag'
     operator: "<="

How Measurements are submitted to SQUASH
========================================

Design Principles
-----------------
* Think about Airplane Mode.
* Think about how this will eventually be a Butler.put().

Proposal
--------

Packages construct a ``Job`` that contains ``Measurements``, ``blobs`` and ``Provenance``. This ``Job``, serialized to JSON, is sent over the logger. A special Metric logger is used that saves this log statement to a separate file. A next-generation post-qa sends this Job to SQUASH's REST API.

* Bonus: Packages could provide Jupyter Notebooks that locally consume the log data to show plots and pass/fail Specification status.
* Bonus: make validate_base capable of generating the Jupyter Notebook!
* Bonus: share Bokeh plots between notebooks and SQUASH.

How Specifications are registered
=================================

Design principles
-----------------

* ``Specifications`` are a mechanism for LSST staff to monitor a ``MeasurementView`` and be alerted whenever a new ``Measurement`` exceeds a threshold or range.
* It needs to be easy for any LSST staff member to register a new ``Specification``; they should not be required to contact SQuaRE to register or change a ``Specification``.
* Specifications should be available offline, but be synced to SQUASH.

Proposal
--------
There is a common EUPS package that contains ``Specifications`` in a YAML format. These Specifications are available, through a Python API, to packages so that they can show real-time pass/fail status of Measurements. The Specifications are also synchronized with the SQUASH database. If someone wants to be alerted by a Specification, they sign themselves up as an owner of the Specification.
