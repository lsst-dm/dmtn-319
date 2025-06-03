#######################################################
DM-49817: Plans for fake injection for AP commissioning
#######################################################

.. abstract::

   Describe and discuss possible plans for fake source injection campaign in the Alert Production for commissioning


The AP performance estimate with Fake injection
===============================================

The Vera Rubin LSST Sruvey Alert Production requirements demand that the used pipleines deliver high completeness of transient source detection, as well as low contamination of false-positives.

In order to improve the processing pipelines, and achieve the desired performance it is also necessary to measure the performance periodically, and if possible, continuously. One of our main tools for addressing pipeline evolution, data quality and scientific requirement goals is the injection of synthetic sources (or Fakes) into the image pixels, with controlled parameters. The properties of these sources as measured in the images by our software can reveal qualitatively and cuantitatively what is our performance level and more critically, where are our shortcomings.

Desigining a sufficient fake injection campaign is therefore critical, to understand our software, and guarantee a standard in scientific performance. However, this is a challenging task, with computational resources being a premium resource that is highly prioritised, and mostly dedicated to essential operations tasks.


Overall description of fake injection tool
==========================================

Software tool: ``source_injection`` package
-------------------------------------------

We inject sources using the ``source_injection`` package from the lsst stack, and leverage its framework to modify the nominal pipelines (``ApPipe.yaml``) so they include the injection as well as the automatic analysis needed tasks.

The pipeline is usually referred as ``ApPipeWithFakes.yaml`` in the pipeline repository (`ap_pipe`_). We inject into ``visit_image`` dataset types at the moment, and further down we will explain injection on coadded images as well.

.. _ap_pipe: https://github.com/lsst/ap_pipe

Types of fake sources
---------------------

We can list as primary tool for AP the fake point sources as our main type of source to measure transient detection performance. It has been discussed the possibility of simulating and rendering extended sources like comets, and in fact this is technically possible. No streak sources or SSOs are being injected at the moment as well.

Simulating variability of sources
---------------------------------

To simulate variable sources we need to inject in the template coadded images, with potentially as well another counterpart injection in the visit images, with a different magnitude value.

There is as well the challenge of making this realistic: injecting into visit images used to coadd, with correct variability brightness and therefore obtain a realistic coadd "average" flux.

Injection catalogs
------------------

We currently construct a catalog of injected sources by joining two different samples of point sources, a set of hosted sources to emulate transients in galaxies and second set of hostless.
The hosts are selected from the pipeline source catalog that is produced upstream by imposing a cut in their extendedness measurement, and selecting :math:`N_{src}=min(100, N\times0.05)` of the available sources per detector.

For each host we pick a random position angle and radius using its light profile shape, and also a random value of brightness for the injected source, with magnitudes higher than the host source.

The hostless sources instead have random positions in the ccd focal plane, and with magnitudes chosen from a random uniform distribution with :math:`20 \geq m \geq m_{lim} + 1`  with :math:`m_{lim}` the limiting magnitude of the image.



Operational procedure and implementation examples
=================================================

To inject the sources it is first needed an injection catalog, that can has to be already ingested into the corresponding butler repository.

This requires that we have a clear selected dataset to perform the injections, given computational constraints. The most important piece of information is related to the position of the fakes (if they are hosted, also the host ID and metadata), and also, related to the visit ID (that is a proxy for MJD or time).
This way, we can have fakes that are different from visit to visit in position, abut also in brightness, and build light-curves.

In any case, no fake catalog can be produced on demand, that is, while running the fake injection pipeline, as it is a pre-requisite Input for the injection tasks. This catalog creation, therefore, must be run after the single-frame processing of the night is done, or at least the visits to ingest are already with minimum-standard calibration in astrometry and photometry.

For this we need a catalog creation tool that can be run in a separate process, and that can be used to generate the fake catalog for a given set of visits, and then ingest it into the butler repository.


Operation strategies for injection campaigns
--------------------------------------------

Pre-selected regions of the sky where to inject
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

It is interesting the idea of performing the injection into small selected fields, distributed in the sky so there is at least some observations per night of these fields. Effectively, this would be done with the skymap tesselation already available, using a choice of tract-patch-filter.

The advantage of this approach is that we can have a catalog of fakes sources with fixed positions in the sky and already chosen specific variability models, and every night we use the overlap of the recently taken images with the fields to calculate the corresponding magnitude or flux to inject. An extra advantage of this approach is that we can have pre-injected templates already stored in a specific collection. This can save a lot of planning and computational cost.

Additional advantages of this case is the straightforward obtention of light-curves of variable or transient sources, with realistic history. Therefore, this is a clear science verification test, that can deliver metrics on detection, but as well, measurement of properties of transients and variables.

The field selection and the amount of sources would be chosen so we have enough statistical sampling of our detection/selection function, and we don't bias towards specific configuration scenarios (stellar density for example).

A problem with this strategy, is that we would be probing only selected template images, one per area (or group of patches), and so we would have information from the fake recovery metrics that is correlated in the sky.
A mitigation strategy would be to perform the choice of fields in cycles, and rotate them frequently enough.

Another problem that needs solving for this approach is the chance of having visits that marginally overlap with the chosen regions of the sky for injection, triggering full processing of certain images, but with less return of value in terms of calculated metrics. Maybe we can mitigate such scenario with a threshold on the amount of overlap between the observed visits and the injection fields, but this also raises the issue of having a (potentially low) fraction of cases with incomplete light curve recovery.


Random full-visit injections
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We can perform a random choice of the night observation visit list, and do full-focal plane injection of fake sources. For this to work, we would have to run this at day-time processing, with the results of astrometric calibration finished to the whole set of visits, in order to probe observations spanning diverse areas of the sky. This also ensures no correlation of the results, by probing templates of different tract-patches of the sky.

The random full-visit injection has advantages: it probes all detectors in the focal plane at once, thus providing a good health check on the detection efficiency and contamination for all CCDs in a periodic way. This can aid in detecting subtle problems quickly, and rule out detector problems.
This also would probe templates coming from contiguous tract-patches, and show problems at the edge of the overlaps, check for race conditions on any source IDs or diaSrc/diaObj database. This could even be tested with random *consecutive* full-visit injections, probing the prompt-processing machinery for association as well.

A key challenge to overcome for this, is the template fake and visit fake catalog generation, which has to be done prior to starting the injection pipelines, on the fly, after selecting the full-visit images to use. Even further, the calculation of the template catalog has to correspond to the full-visit sky coverage, and include some buffer, so they have to be generated simultaneously.


Random detector injections
^^^^^^^^^^^^^^^^^^^^^^^^^^

This approach is very similar to the random full-visit case, but scaling down the area of test to just one detector (or a group of them). Advantages of this strategy include the savings of computational cost, and possibly, the injection on many more visits per night, covering as well the observing plan more uniformly across the night, testing for telescope and condition changes like airmass, cirrus, vibration, thermal camera noise, etc.

Again, here the challenge to overcaome is the choice of detectors, and the calculation of template fake source to inject.


Note on "Random" choices
------------------------

We will often here refer to "random" visit, or "randomly chosen" field, etc. This randomization of chosen data or parameters to explore, is usually for computational power saving, and the quantification of metrics across the wide parameter space. Therefore, there are certain preferences when chosing random data: we usually want a comprehensive coverage of the most tipycal scenario of observations for Rubin LSST survey, but also we want to find corner cases where performance might be degraded (for example in high airmass or moon illumination).

A random choice therefore, will not be uniformly distributed random, but more like an optimized random sampling of the operations instrumental configuration and observational conditions.


Note on image subtraction
=========================

We raise attention to the issue that the image subtraction technique with the injected fake sources should not obtain a different matching kernel result than the nominal result without injections. This can be a secondary null test or verification, that would serve the purpose of testing stability of our subtraction, and possible assign an error to the kernel determination.

Another approach to ensure this is satisfied, is to not re-derive the kernel, but instead use the already calculated kernel and re-apply the exact convolution calculations to get the image difference with the fake sources.
This would be providing performance metric information on the subtractions already processed and characterize them with the highest possible fidelity.


Resources needed for the campaign
=================================

The fake injection campaign would need three stages:

- Catalog generation and ingestion

- Data processing

  - visit image injection
  - template injection
  - diaSource association and
  - fake catalog crossmatching
  - catalog consolidation

- Metric estimation and figure generation

The most intensive stages are related to the image data processing. The processing is very similar to nominal prompt processing pipeline, but it does not requires a PPDB (or APDB) to obtain metrics, and it has the extra tasks needed for pixel injection.






Add content here
================

See the `Documenteer documentation <https://documenteer.lsst.io/technotes/index.html>`_ for tips on how to write and configure your new technote.


