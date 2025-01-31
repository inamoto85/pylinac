
============================
CatPhan module documentation
============================

Overview
--------

.. automodule:: pylinac.ct
    :no-members:

Running the Demo
----------------

To run one of the CatPhan demos, create a script or start an interpreter and input:

.. code-block:: python

    from pylinac import CatPhan504
    CatPhan504.run_demo() # the demo is a Varian high quality head scan

Results will be printed to the console and a figure showing the slices analyzed will pop up::

     - CatPhan 504 QA Test -
    HU Linearity ROIs: {'Poly': -45.0, 'PMP': -200.0, 'Acrylic': 115.0, 'Teflon': 997.0, 'Delrin': 340.0, 'Air': -998.0, 'LDPE': -102.0}
    HU Passed?: True
    Uniformity ROIs: {'Center': 14.0, 'Top': 6.0, 'Left': 10.0, 'Bottom': 5.0, 'Right': 0.0}
    Uniformity index: -1.3806706114398422
    Integral non-uniformity: 0.006951340615690168
    Uniformity Passed?: True
    MTF 50% (lp/mm): 0.95
    Low contrast ROIs "seen": 3
    Low contrast visibility: 3.4654015681608437
    Geometric Line Average (mm): 49.93054775087732
    Geometry Passed?: True
    Slice Thickness (mm): 2.5007568359375
    Slice Thickness Passed? True

.. plot::

    from pylinac import CatPhan504
    cbct = CatPhan504.from_demo_images()
    cbct.analyze()
    cbct.plot_analyzed_image()

As well, you can plot and save individual pieces of the analysis:

.. code-block:: python

    cbct = CatPhan504.from_demo_images()
    cbct.analyze()
    cbct.plot_analyzed_subimage('linearity')
    cbct.save_analyzed_subimage('linearity.png', subimage='linearity')

.. raw:: html
    :file: images/cbct_hu_lin.html

Or:

.. code-block:: python

    cbct.plot_analyzed_subimage('rmtf')

.. raw:: html
    :file: images/cbct_rmtf.html

Or generate a PDF report:

.. code-block:: python

    cbct.publish_pdf('mycbct.pdf')



Typical Use
-----------

CatPhan analysis as done by this module closely follows what is specified in the CatPhan manuals, replacing the need for manual measurements.
There are 4 CatPhan models that pylinac can analyze: :class:`~pylinac.ct.CatPhan504`, :class:`~pylinac.ct.CatPhan503`, & :class:`~pylinac.ct.CatPhan600`, &
:class:`~pylinac.ct.CatPhan604`, each with their own class in
pylinac. Let's assume you have the CatPhan504 for this example. Using the other models/classes is exactly
the same except the class name.

.. code-block:: python

    from pylinac import CatPhan504  # or import the CatPhan503 or CatPhan600

The minimum needed to get going is to:

* **Load images** -- Loading the DICOM images into your CatPhan object is done by passing the images in during construction.
  The most direct way is to pass in the directory where the images are:

  .. code-block:: python

    cbct_folder = r"C:/QA Folder/CBCT/June monthly"
    mycbct = CatPhan504(cbct_folder)

  or load a zip file of the images:

  .. code-block:: python

    zip_file = r"C:/QA Folder/CBCT/June monthly.zip"
    mycbct = CatPhan504.from_zip(zip_file)

  You can also use the demo images provided:

  .. code-block:: python

    mycbct = CatPhan504.from_demo_images()

* **Analyze the images** -- Once the folder/images are loaded, tell pylinac to start analyzing the images. See the
  Algorithm section for details and :meth:`~pylinac.cbct.CatPhan504.analyze`` for analysis options:

  .. code-block:: python

    mycbct.analyze()

* **View the results** -- The CatPhan module can print out the summary of results to the console as well as draw a matplotlib image to show where the
  samples were taken and their values:

  .. code-block:: python

      # print results to the console
      print(mycbct.return_results())
      # view analyzed images
      mycbct.plot_analyzed_image()
      # save the image
      mycbct.save_analyzed_image('mycatphan504.png')
      # generate PDF
      mycbct.publish_pdf('mycatphan.pdf', open_file=True)  # open the PDF after saving as well.

Algorithm
---------

The CatPhan module is based on the tests and values given in the respective CatPhan manual. The algorithm works like such:

**Allowances**

* The images can be any size.
* The phantom can have significant translation in all 3 directions.
* The phantom can have significant roll and moderate yaw and pitch.

**Restrictions**

    .. warning:: Analysis can fail or give unreliable results if any Restriction is violated.

* The phantom used must be an unmodified CatPhan 504, 503, or 600.
* All of the relevant modules must be within the scan extent; i.e. one can't scan only part of the phantom.


**Pre-Analysis**

* **Determine image properties** -- Upon load, the image set is analyzed for its DICOM properties to determine mm/pixel
  spacing, rescale intercept and slope, manufacturer, etc.
* **Convert to HU** -- The entire image set is converted from its raw values to HU by applying the rescale intercept
  and slope which is contained in the DICOM properties.
* **Find the phantom z-location** -- Upon loading, all the images are scanned to determine where the HU linearity
  module (CTP404) is located. This is accomplished by examining each image slice and looking for 2 things:

    * *If the CatPhan is in the image.* At the edges of the scan this may not be true.
    * *If a circular profile has characteristics like the CTP404 module*. If the CatPhan is in the image, a circular profile is taken
      at the location where the HU linearity regions of interest are located. If the profile contains low, high, and lots of medium
      values then it is very likely the HU linearity module. All such slices are found and the median slice is set as the
      HU linearity module location. All other modules are located relative to this position.

**Analysis**

* **Determine phantom roll** -- Precise knowledge of the ROIs to analyze is important, and small changes in rotation
  could invalidate automatic results. The roll of the phantom is determined by examining the HU module and converting to
  a binary image. The air holes are then located and the angle of the two holes determines the phantom roll.

  .. note::
        For each step below, the "module" analyzed is actually the mean, median, or maximum of 3 slices (+/-1 slice around and
        including the nominal slice) to ensure robust measurements. Also, for each step/phantom module, the phantom center is
        determined, which corrects for the phantom pitch and yaw.

        Additionally, values tend to be lazy (computed only when asked for), thus the calculations listed may sometimes
        be performed only when asked for.

* **Determine HU linearity** -- The HU module (CTP404) contains several materials with different HU values. Using
  hardcoded angles (corrected for roll) and radius from the center of the phantom, circular ROIs are sampled which
  correspond to the HU material regions. The mean pixel value of the ROI is the stated HU value.
* **Determine HU uniformity** -- HU uniformity (CTP486) is calculated in a similar manner to HU linearity, but
  within the CTP486 module/slice.
* **Calculate Geometry/Scaling** -- The HU module (CTP404), besides HU materials, also contains several "nodes" which
  have an accurate spacing (50 mm apart). Again, using hardcoded but corrected angles, the area around the 4 nodes are
  sampled and then a threshold is applied which identifies the node within the ROI sample. The center of mass of the node is
  determined and then the space between nodes is calculated.
* **Calculate Spatial Resolution/MTF** -- The Spatial Resolution module (CTP528) contains 21 pairs of aluminum bars
  having varying thickness, which also corresponds to the thickness between the bars. One unique advantage of these
  bars is that they are all focused on and equally distant to the phantom center. This is taken advantage of by extracting
  a :class:`~pylinac.core.profile.CollapsedCircleProfile` about the line pairs. The peaks and valleys of the profile are located;
  peaks and valleys of each line pair are used to calculated the MTF. The relative MTF (i.e. normalized to the first line pair) is then
  calculated from these values.
* **Calculate Low Contrast Resolution** -- Low contrast is inherently difficult to determine since detectability of humans
  is not simply contrast based. Pylinac's analysis uses both the contrast value of the ROI as well as the ROI size to compute
  a "detectability" score. ROIs above the score are said to be "seen", while those below are not seen. Only the 1.0% supra-slice ROIs
  are examined. Two background ROIs are sampled on either side of the ROI contrast set. The score for a given ROI is
  calculated like so :math:`\frac{ROI_{pixel} - background}{ROI_{stdev}} * ROI_{diameter}`, where :math:`ROI_{pixel}` is the
  mean pixel value of the ROI, :math:`background` is the mean pixel value of the two background ROIs, and :math:`ROI_{diameter}`
  is the diamter of the ROI in mm. The default detectability score is 10.
* **Calculate Slice Thickness** -- Slice thickness is measured by determining the FWHM of the wire ramps in the CTP404 module.
  A profile of the area around each wire ramp is taken, and the FWHM is determined from the profile. Based on testing, the FWHM
  is not always perfectly detected and may not "catch" the profile, giving an undervalued representation. Thus, the
  two longest profiles are averaged and the value is converted from pixels to mm and multiplied by 0.42.


**Post-Analysis**

* **Test if values are within tolerance** -- For each module, the determined values are compared with the nominal values.
  If the difference between the two is below the specified tolerance then the module passes.

Troubleshooting
---------------

First, check the general :ref:`general_troubleshooting` section.
Most problems in this module revolve around getting the data loaded.

* If you're having trouble getting your dataset in, make sure you're loading the whole dataset.
  Also make sure you've scanned the whole phantom.
* Make sure there are no external markers on the CatPhan (e.g. BBs), otherwise the localization
  algorithm will not be able to properly locate the phantom within the image.
* Ensure that the FOV is large enough to encompass the entire phantom. If the scan is cutting off the phantom
  in any way it will not identify it.
* The phantom should never touch the edge of an image, see above point.
* Make sure you're loading the right CatPhan class. I.e. using a CatPhan600 class on a CatPhan504
  scan may result in errors or erroneous results.

API Documentation
-----------------

The CatPhan classes uses several other classes. There are several Slices of Interest (SOI), most of which contain Regions of Interest (ROI).

.. autoclass:: pylinac.ct.CatPhan504

.. autoclass:: pylinac.ct.CatPhan503

.. autoclass:: pylinac.ct.CatPhan600

.. autoclass:: pylinac.ct.CatPhan604

.. autoclass:: pylinac.ct.CatPhanBase

Module classes (CTP404, etc)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. autoclass:: pylinac.ct.Slice

.. autoclass:: pylinac.ct.CatPhanModule

.. autoclass:: pylinac.ct.CTP404

.. autoclass:: pylinac.ct.CTP528

.. autoclass:: pylinac.ct.CTP515

.. autoclass:: pylinac.ct.CTP486


ROI Objects
^^^^^^^^^^^

.. autoclass:: pylinac.ct.ROIManagerMixin

.. autoclass:: pylinac.ct.HUDiskROI

.. autoclass:: pylinac.ct.RectangleROI

.. autoclass:: pylinac.ct.ThicknessROI

.. autoclass:: pylinac.ct.GeometricLine

Helper Functions
^^^^^^^^^^^^^^^^

.. autofunction:: combine_surrounding_slices
