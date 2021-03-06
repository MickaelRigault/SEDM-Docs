
SEDM Pipeline
=============

Pipeline Overview
-----------------

A new pipeline developed by Mickael Rigault has been installed for
automatic reduction of SEDM data.  The distribution is available on
github__.  It generates a geometry solution from the calibration images and
then automatically extracts target spectra based on the WCS solution of the
guider images.  The extracted spectrum is classified using SNID__ and if it
is a ZTF target, the ascii spectrum is uploaded to the growth marshal__.
The only interactive step is to generate a final report once all the
extractions have been verified.

__ https://github.com/MickaelRigault/pysedm
__ https://people.lam.fr/blondin.stephane/software/snid/
__ http://skipper.caltech.edu:8080/cgi-bin/growth/marshal.cgi

If there is a failure of the WCS solution, or if the target is particularly
difficult to model with a PSF, there are ways to re-extract the spectrum.
This will described in more detail below.

Python Requirements
^^^^^^^^^^^^^^^^^^^

The IFU pipeline is compatible with python v2.7 and v3.6 and currently runs
under the miniconda3__ distribution.  It requires the astroconda__ environment 
from STScI and expects the name to be 'astroconda':

``conda create -n astroconda stsci``

__ https://conda.io/miniconda.html
__ https://astroconda.readthedocs.io/en/latest/


Automated Pipeline Operations
-----------------------------

Now we describe the steps that the pipeline takes during the automated
operations.  We start with the pre-science processing that generates the
nightly geometry solution and then continue with the science target
processing.

Pre-Science Processing
^^^^^^^^^^^^^^^^^^^^^^

Pre-science processing occurs in the afternoon and takes roughly 30 minutes
to complete.  In the afternoon when the UT date changes, the following
steps are automatically performed:

#. The appropriate reduced directory is created using the UT date:
    * ``/scr2/sedmdrp/redux/20180823`` (e.g.)
#. The required raw calibration files are linked into the directory as they are taken.
#. Once all the bias files are acquired the master biases are generated.
#. All subsequent calibration files are linked into the directory and then bias-subtracted and cosmic ray cleaned.
#. Once all the calibration files are acquired a geometry solution is generated.
#. If there is a failure in the geometry solution, geometry files from previous runs are linked in.
#. The most recent fluxcal fits file is linked in.

Science Processing
^^^^^^^^^^^^^^^^^^

Now the SEDM is ready for science images.  Near the end of astronomical
twilight, science image acquisition begins.  The following steps are
automatically performed:

#. All new IFU images are linked in and bias-subtracted and cosmic ray cleaned.
#. The sky lines are used to solve for the flexure offsets for the observation.
#. The geometry solution is used to generate a flexure-corrected cube for the observation.
#. If the target is a standard star:
        a) no guider image is generated.
        b) the brightest spaxel is used to define the extraction region.
        c) a new fluxcal fits file is generated.
#. If the target is a science object:
        a) a guider image is generated from all the guider frames.
        b) the WCS is solved for the guider image
        c) an extraction region in the IFU is based on the guider WCS and the target coordinates.
        d) PSF-forced photometry is performed.
        e) the nearest fluxcal file is used to calibrate the science target.
        f) the telluric absorption is corrected.
        g) the resulting spectrum is classified using SNID
        h) the SNID results are put in the ascii spectrum header.
        i) the extraction is recorded in a file called ``report.txt``
#. If the target is a ZTF object:
        a) the spectrum is uploaded to the growth marshal.
        b) the marshal URL is recorded in the file ``report_ztf.txt``

The format of the ascii spectrum that is generated is universal enough to
be input to any classifier (Superfit, e.g.).


Interactive Processing
----------------------

All target extractions should be verified and adjusted if required.  Once
that is done a final report is generated that sends out a summary e-mail of
the night's results.  In order to do this, one has to connect to
`pharos.caltech.edu` via a VNC connection.  If the screen lock is active,
just enter the password to unlock it.  Below is is a figure showing the
layout of the main desktop screen connected through the VNC connection.

.. figure:: PharosSEDMdesktopNew.png

    Figure 1. Pharos sedmdrp desktop on screen 7 (5907).

The automatic pipeline script is running in the bottom right xterm window.  Some
status information can be gleaned from the output there.  The xterm set on
the left may be used by the observer to examine the files on pharos.  A web
browser will be set up on the secondary desktop to the right which can be
selected using the chooser on the lower right.  This is where you can
interact with the SEDM web site and the growth marshal and other web
services to look at finder charts.

In the top-right Xterm window, the observer interacts with the pipeline as
described below.  Be sure to `cd` into the current directory, which is the
UT date formatted as YYYYMMDD (20180907, e.g., which would be found in
/scr2/sedmdrp/redux/20180907).

Verification
^^^^^^^^^^^^

The automated pipeline generates verification plots as each image is processed.
These are PNG image files that start with ``verify_``.  You can display all
of them using the ``display`` command from ImageMagick like this:

``display verify_*.png &``

Figures 2 - 4 show the three types of verification plots.  For all three types,
the acquisition finder chart is shown in the upper right and
the IFU spaxel plot is in the upper left.  The PSF extraction results are shown
in the lower left in three plots showing the Data, Model, and Residual.
Finally, in the lower right, is shown some form of the extracted spectrum.  For
a standard star, it will show the calibration check plot comparing the
reference spectrum to the observed spectrum (see Figure 2).

.. figure:: verify_forcepsf_auto_lstep1__crr_b_ifu20180907_03_03_14_STD-BD+33d2642.png

    Figure 2. Verification plot for standard star BD+33d2642

For a science target that has a successful classification from SNID, it will
show the SNID template match plot (see Figure 3).

.. figure:: verify_forcepsf_auto_lstep1__crr_b_ifu20180907_10_55_22_ZTF18abosrco.png

    Figure 3. Verification plot for successfully typed science target ZTF18abosrco

For a science target for which SNID fails to find a classification, it will
show only the extracted spectrum (see Figure 4).

.. figure:: verify_forcepsf_auto_lstep1__crr_b_ifu20180907_11_38_04_ZTF18absqitc.png

    Figure 4. Verification plot for unsuccessfuly typed science target ZTF18absqitc

The first step of verification is to compare the B&W finder (upper right) with
the IFU extraction region (upper left).  The red right-angle in the B&W finder
indicates the location of the target.  If the IFU extraction region indicated by
black dots contains the object and the centroid, indicated by either a red X or
a red circle is reasonably close to the target, then this is probably a good
extraction.  Next, examine the PSF fit and residual plots in the lower left.
If the model looks reasonably close to the data and the residuals look like the
model accounted for most of the target's flux, then the extraction was
successful.  This is also bolstered if the spectrum looks good and is either a
good match to a SNID template, or to a reference spectrum, or seems to have
good signal-to-noise.

If you want further verification of the target, you will need to move to the
desktop to the right (using the chooser in the lower right, or by moving the
mouse the the right edge of the desktop).  There you can open a web browser, if
needed, and log into the ZTF marshal, the TNS website, or any other web-based
source of finder charts for the target.


Adjustment
^^^^^^^^^^

There are three types of adjustment that can be made.  The first two will
completely replace the original spectrum, but will still need to be
re-classified, re-reported on the slack channel (`pysedm-report`), and
re-uploaded to the growth marshal (if the target is a ZTF object).  As of now,
we are only documenting the first two in full.  The third adjustment creates
new files and requires more bookkeeping and is therefore, not recommended
unless specifically required.


Fix Centroid
~~~~~~~~~~~~

This is the simplest adjustment to make.  It will arise in some cases if the WCS
solution of the guider images failed.  This is indicated in the IFU spaxel plot
by a red circle instead of a red X.  When the WCS solution fails, the
extraction is defined by the brightest pixel.  This is fine for standard stars,
but does not always work for science targets.  Sometimes even successful WCS
solutions will define the centroid in the wrong place.  Let the finder chart in
the verification plot and any other finders from the web be your guide.

It is also possible that a target that is strongly influenced by a neighbor
(host galaxy, nearby star) can be fixed by just moving the centroid, and hence
moving the extraction region, off of the offending neighbor.

To make this adjustment, you simply need to pass the fixed centroid to the
`extract_star.py` program.  Use the IFU spaxel plot to determine the new
centroid for the target.  Then enter the command:

``extract_star.py <UTdate> --auto <timestr> --autobins 6 --centroid <X Y>``,

where <UTdate> is the current directory date string (formatted YYYYMMDD) and
<timestr> is the UT time stamp for the specific observation (formatted
HH_MM_SS, and shown in the title of the verification plot), and <X Y> are
replaced by the correct centroid values as determined from the IFU spaxel plot.
Integer values are usually accurate enough for the new centroid.  Here is an
example:

``extract_star.py 20180907 --auto 10_11_12 --autobins 6 --centroid -3 7``.

This will completely replace the spectrum file for the object and re-generate
the plots.  You will want to display the new plots.  Find the appropriate
psf profile plot file (starts with ``psfprofile_`` and ends with ``.png``).
Use the display command to check if your improved centroid had the effect you
wanted.  You can also check the extracted spectrum in the same way.  Find the
spectrum plot file (starts with ``spec_forcepsf_`` and ends with ``.png``) and
display it.  As a final check, you can display the new IFU spaxel plot (starts
with ``ifu_spaxels_`` and ends with ``.png``).  This plot will now have a black
X where your adjusted centroid falls on the spaxels.

It is fine to tweak the centroid and re-extract the spectrum more than once.
It's important to get a good extraction and this sometimes takes more than
one adjustment to the centroid.

Fix Extraction Region
~~~~~~~~~~~~~~~~~~~~~

This is also a fairly easy adjustment to make.  If the extraction region
includes a neighbor that strongly influences the psf model, and just moving
the centroid doesn't fix it, you can use the `--display` parameter of the
`extract_star.py` program to re-draw the region.  To do this enter the
command:

``extract_star.py <UTdate> --auto <timestr> --autobins 6 --display``,

which will bring up a display window showing the IFU spaxel plot with the
current centroid indicated.  The left panel is the spectrum in the extraction
region and the right is the spaxel map where you can re-draw the region.

Just hold down the shift key and draw a region (by left clicking and dragging
the mouse) around your target that does not include the offending neighbor.
Once you release the left mouse button, the selected region will be shown on
the plot.  If you want to try again, hit the <ESC> key, which will reset the
region, and try again.  Once you are happy with the region, close the plot.
This is done by using the menu at the upper left corner of the window and
selecting `Close`.  The extraction will proceed once the window is closed.

Here is an example for this command:

``extract_star.py 20180907 --auto 10_11_12 --autobins 6 --display``.

As with fixing the centroid, the spectrum file and all the plots will be
replaced.  Use the same method described above to verify that your new
region achieved what you wanted.

Fix Extraction Method
~~~~~~~~~~~~~~~~~~~~~

This is a more challenging adjustment to make.  As of now, the two previous
adjustments seem to be able to fix nearly every situation.  If you need to
perform an aperture extraction, please contact the SEDM team and we can
instruct you how to do this.


Re-Classify
^^^^^^^^^^^

If you have re-extracted an object that was previously classified by SNID,
it's a good idea to remove the old template match plot.  If you don't, this
plot may be taken as the correct classification of the object.  To find the
old template plot, look for a file that starts with ``spec_`` and includes
your target name and ends with ``.png``.  The template match file will have
a classification type in the filename.  Look for Ia, Ib, QSO, e.g., in the file
name just after the target name and delete that plot file.

Once that is done, you can re-classify the spectrum.  This is done by entering:

``make classify``

in the terminal.

Re-Report
^^^^^^^^^

After re-classification, you should send a new report to the SEDM slack channel
`pysedm-report` with the updated extraction.  To do this you enter the command:

``pysedm_report.py <UTdate> --contains <timestr> --slack``,

where <UTdate> and <timestr> have the same meaning as before.  Here is an
example of this command:

``pysedm_report.py 20180907 --contains 10_11_12 --slack``.

This pushes a new report onto the slack channel.  If you have access to the
channel, it is good to make a short comment there that indicates why you have
re-extracted.

Re-Upload
^^^^^^^^^

If the target you are working on is a ZTF target, then you will want to
push your new results to the growth marshal.  If the old spectrum has been
replaced, then you will need to delete the corresponding ``*.upl`` file.  These
files keep track of what has already been uploaded to the marshal.  Therefore,
any new version will not upload unless that file is deleted.  This file will
have the same root as the new text spectrum file, but will end with ``.upl``.
Once this has been deleted, just enter the command:

``make ztfupload``

and this will re-upload the text spectrum to the marshal.

If you have an account on the marshal and if the original spectrum was from a
bad extraction, then you should log onto the marshal, navigate to the target
that was re-extracted and delete the old spectrum.


Final Report
^^^^^^^^^^^^
The last step is to generate the final report which sends an e-mail report out
the to the SEDM team.  To initiate this final step, please enter:

``make finalreport``

It is a good idea to check this e-mail (if you are on the list) and make sure
all of the links work and that the correct extractions are displayed.

Congratulations!  You are done, for now...

Last updated on |version|
