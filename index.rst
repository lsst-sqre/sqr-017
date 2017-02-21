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

   This technote describes the design of the metric and specification system that the new validation framework will use.

Frossie's design principles
===========================

* Metric something something of code. (FIXME)
* Implement as a thin shim over what will become a Butler put.
* UX for metric developers: make it as easy as possible for a developer to define a new metric and measurements appear in SQUASH and be monitored.

The validate_base 2.0 data model
================================

Metric
------

Abstractly, a metric is something that's measureable. Concretely, a metric maps to code that makes a measurement.

Prior art
^^^^^^^^^

* ``lsst.validate.base.Metric``
* ``metric.yaml format``

* ``Attributes``
* ``name``
* ``description``
* ``unit``
* ``reference.doc``
* ``reference.page``
* ``reference.url``

Questions & Notes
^^^^^^^^^^^^^^^^^

* Specifications are not longer contained by Metrics.
* In the existing lsst.validate.base.Metric, there is a parameters dictionary that defines constants for the measurement code. For example, the annulus diameter from AMx metrics. Do we want to keep this? Or should all measurement code configuration happen in task configuration?
* We talked about making the minimum provenance required for a measurement/job being defined in the Metric. Is this still a thing?

Measurement
-----------

A measurement is a realization of a metric: always a scalar value.

Prior art
^^^^^^^^^

* ``lsst.validate.base.MeasurementBase``

Attributes
^^^^^^^^^^

* ``job_id`` — Job identifier.
* ``metric_name`` — Metric identifier.
* ``value`` — scalar measurement value, required to be persisted in units of Metric.unit.

Questions & Notes
^^^^^^^^^^^^^^^^^

* ``lsst.validate.base.MeasurementBase`` originally had a parameters attribute that provided provenance for how the measurement was made (e.g., a S/N cut-off for star selection). These will now be part of the Task configuration, and available through the regular provenance.
* ``lsst.validate.base.MeasurementBase`` also had an extras attribute where additional measurement outputs could be persisted (JSON serializable). Do we still want this? Or do we always want such data to go into a Blob?

Blob
----

A blob is a container for JSON-serializable data that may be associated with one or more measurements that might be useful for rendering plots and doing SQUASH-side analysis.

Prior art
^^^^^^^^^

* ``lsst.validate.base.BlobBase``

Job
---

A job is an execution of a pipeline, containing measurements, blobs, and their provenance.

Prior art
^^^^^^^^^

* ``lsst.validate.base.Job``

Attributes
^^^^^^^^^^

* ``measurements`` — list of Measurement objects.
* ``blobs`` — list of Blob objects. Each blob should be reference by at least one * measurement.
* ``provenance`` — data structure that fully specifies the provenance of the pipeline run.

Questions & Notes
^^^^^^^^^^^^^^^^^^^

* What is the schema of provenance? It includes both the input dataIds (input dataset) and task configurations?
* Not all provenance is currently known within the pipeline. We use post-qa to hydrate Job provenance with package versions and Jenkins environment variables. However, working towards a state where post-qa is no longer used as a shim, it's not unreasonable to move this into validate_base.

Specification
-------------

A specification is a binary (pass/fail) evaluation of a measurement. There can be an arbitrary number of specifications associated with a metric.

Attributes
^^^^^^^^^^

* ``metric_name`` — Identifier of the metric that this specification is attached to.
* ``provenance_query`` — only measurements that have matching provenance parameters are tested by this specification.
* ``alert_listeners`` - Slack IDs of people who are alerted if a measurement fails the specification.
* ``alert_channels`` - Slack Channel IDs that recieve messages when a measurement fails a specification.
* ``threshold`` and comparison_operator — measurement passes specification if measurement is on the side of the threshold indicated by the comparison operator.
* ``range`` — measurement passes specification if measurement is within this range (new).

Questions & Notes
^^^^^^^^^^^^^^^^^^^

* Either threshold or range can be set. Possibly there should be different classes of specification (i.e., a ThresholdSpecification or a RangeSpecification).
* Note that we're jettisoning some of the earlier Specification class baggage, like parameters, and dependencies. This means that the definitions of metrics are no longer driven by definitions of specifications, as they currently are for AFx/ADx, for example. Instead, this flexibility is handled by additional metrics.

MeasurementView
---------------

A MeasurementView is a collection of measurements for a metric, possibly filtered by provenance. A MeasurementView can be used to populate a measurement timeseries (regression plot), as seen in SQUASH. A MeasurementView is essentially a DB query, but provides a more concrete API for us to think about how we can do data science against measurements.

Attributes
^^^^^^^^^^

* metric_name
* provenance_query

How packages define new metrics
===============================

* Every package that measures metrics defines these metrics in an etc/metrics.yaml file.
* How is this metric added to the SQUASH DB?
* Package developer implements measurement code and puts data into measurement/blob/job objects.

How measurements are submitted to SQUASH
========================================

Design Principles
-----------------
* Think about Airplane Mode.
* Think about how this will eventually be a Butler.put().

Proposal
--------

Packages construct a Job that contains measurements, blobs and provenance. This Job, serialized to JSON, is sent over the logger. A special metric logger is used that saves this log statement to a separate file. A next-generation post-qa sends this job to SQUASH's REST API.

* Bonus: Packages could provide Jupyter Notebooks that locally consume the log data to show plots and pass/fail specification status.
* Bonus: make validate_base capable of generating the Jupyter Notebook!
* Bonus: share Bokeh plots between notebooks and SQUASH.

How specifications are registered
=================================

Design principles
-----------------

* Specifications are a mechanism for LSST staff to monitor a MeasurementView and be alerted whenever a new measurement exceeds a threshold or range.
* It needs to be easy for any LSST staff member to register a new specification; there shouldn't.
* Specifications should be available offline, but be synced to SQUASH.

Proposal
--------
There is a common EUPS package that contains Specifications in a YAML format. These specifications are available, through a Python API, to packages so that they can show real-time pass/fail status of measurements. The specifications are also synchronized with the SQUASH database. If someone wants to be alerted by a specification, they sign themselves up as an owner of the specification.
