..
  Technote content.

  See https://developer.lsst.io/docs/rst_styleguide.html
  for a guide to reStructuredText writing.

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label
     :target: http://target.link/url

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1
.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. Add content below. Do not include the document title.

.. note::

   **This technote is not yet published.**

   This technote describes the design of the Metric and Specification system that the new validation framework will use.

Frossie's design principles
===========================

* Metric something something of code. (FIXME)
* Implement as a thin shim over what will become a Butler put.
* UX for Metric developers: make it as easy as possible for a developer to define a new ``Metric`` and ``Measurements`` appear in SQUASH and be monitored.

The validate_base 2.0 data model
================================

Afterburner
-----------

An ``afterburner`` is a piece of code that is executed on the output of a ``Task`` in order to calculate a ``Measurement`` of a ``Metric``.

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
* ``reference.doc``
* ``reference.page``
* ``reference.url``

Questions & Notes
^^^^^^^^^^^^^^^^^

* ``Specifications`` are no longer contained by ``Metrics``.
* In the existing ``lsst.validate.base.Metric``, there is a parameters dictionary that defines constants for the Measurement code. For example, the annulus diameter from AMx Metrics. These parameters will be contained in the ``Specifications``.
* We talked about making the minimum ``Provenance`` required for a ``Measurement``/``Job`` being defined in the ``Metric``. Is this still a requirement?

Measurement
-----------

A ``Measurement`` is a realization of a ``Metric``: always a scalar value.

Prior art
^^^^^^^^^

* ``lsst.validate.base.MeasurementBase``

Attributes
^^^^^^^^^^

* ``Job_id`` — Job identifier.
* ``Metric_name`` — Metric identifier.
* ``value`` — scalar Measurement value, required to be persisted in units of ``Metric.unit``.

Questions & Notes
^^^^^^^^^^^^^^^^^

* ``lsst.validate.base.MeasurementBase`` originally had a parameters attribute that provided Provenance for how the Measurement was made (e.g., a S/N cut-off for star selection). These will now be part of the Task configuration, and available through the regular Provenance.
* ``lsst.validate.base.MeasurementBase`` also had an extras attribute where additional Measurement outputs could be persisted (JSON serializable). Do we still want this? Or do we always want such data to go into a Blob?

Blob
----

A ``blob`` is a container for JSON-serializable data that may be associated with one or more Measurements that might be useful for rendering plots and doing SQUASH-side analysis.

Prior art
^^^^^^^^^

* ``lsst.validate.base.BlobBase``

Job
---

A ``Job`` is an execution of a pipeline, containing ``Measurements``, ``blobs``, and their ``Provenance``.

Prior art
^^^^^^^^^

* ``lsst.validate.base.Job``

Attributes
^^^^^^^^^^

* ``Measurements`` — list of Measurement objects.
* ``blobs`` — list of Blob objects. Each blob should be reference by at least one * Measurement.
* ``Provenance`` — data structure that fully specifies the Provenance of the pipeline run.

Questions & Notes
^^^^^^^^^^^^^^^^^^^

* What is the schema of ``Provenance``? At minimum, it includes the input dataIds (input dataset) and task configurations.
* Not all ``Provenance`` is currently known within the pipeline. We use post-qa to hydrate Job Provenance with package versions and Jenkins environment variables. However, working towards a state where post-qa is no longer used as a shim, it's not unreasonable to move this into validate_base.

Provenance
----------

All metadata associated with this ``Job`` run, including Config parameters, Butler dataRefs, cluster configuration, etc.

Questions & Notes
^^^^^^^^^^^^^^^^^

* How is provenance defined?
* How do we define queries on provenance in a ``Specification``?
* How do we map between this provenance and the one that DAX will maintain?

Specification
-------------

A ``Specification`` is a binary (pass/fail) evaluation of a ``Measurement`` of a ``Metric``. There can be an arbitrary number of ``Specifications`` associated with a ``Metric``.

Attributes
^^^^^^^^^^

* ``Metric_name`` — Identifier of the ``Metric`` that this Specification is attached to.
* ``Provenance_query`` — only ``Measurements`` that have matching ``Provenance`` parameters are tested by this ``Specification``.
* ``Parameters`` - A dict of key:value pairs that must be matched by the ``Job``'s ``Provenance`` regarding particular values used in a calculation (e.g. diameter used for aperture photometry).
* ``alert_listeners`` - Slack IDs of people who are alerted if a ``Measurement`` fails the ``Specification``.
* ``alert_channels`` - Slack Channel IDs that recieve messages when a ``Measurement`` fails a ``Specification``.
* ``threshold`` and comparison_operator — ``Measurement`` passes ``Specification`` if ``Measurement`` is on the side of the threshold indicated by the comparison operator.
* ``range`` — ``Measurement`` passes ``Specification`` if Measurement is within this range (new).

Questions & Notes
^^^^^^^^^^^^^^^^^

* Either threshold or range can be set. Possibly there should be different classes of Specification (i.e., a ThresholdSpecification or a RangeSpecification).
* Note that we're jettisoning some of the earlier ``Specification`` class baggage, like dependencies. This means that the definitions of Metrics are no longer driven by definitions of Specifications, as they currently are for AFx/ADx, for example. Instead, this flexibility is handled by additional Metrics.
* Should the ``Parameters`` just be part of the ``Provenance``, or should they be a separate section for maintanence convinience and get ingested into the ``Provenance``?

MeasurementView
---------------

A MeasurementView is a collection of Measurements for a Metric, possibly filtered by Provenance. A MeasurementView can be used to populate a Measurement timeseries (regression plot), as seen in SQUASH. A MeasurementView is essentially a DB query, but provides a more concrete API for us to think about how we can do data science against Measurements.

Attributes
^^^^^^^^^^

* ``Metric_name``
* ``Provenance_query``

How packages define new Metrics
===============================

* Every ``Metric`` and ``Specification`` is defined in yaml files in ``validation_Metrics``, in per-package directories.
* How is this Metric added to the SQUASH DB?
* Package developer implements Measurement code and puts data into Measurement/blob/Job objects.

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
