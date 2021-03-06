

.. _sphx_glr_auto_examples_01_DataOperations_plot_design_matrix.py:


Design Matrix Creation
======================

This tutorial illustrates how to use the Design_Matrix class to flexibly create design matrices that can then be used with the Brain_Data class to perform univariate regression.

Design Matrices can be thought of as "enhanced" pandas dataframes; they can do everything a pandas dataframe is capable of, with some added features. Design Matrices follow a data organization format common in many machine learning applications such as the sci-kit learn API: 2d tables organized as observations by features. In the context of neuro-imaging this often translates to TRs by conditions of interest + nuisance covariates (1st level analysis), or participants by conditions/groups (2nd level analysis).



Design Matrix Basics
--------------------

Lets just create a basic toy design matrix by hand corresponding to a single participant's data from an experiment with 12 TRs, collected at a temporal resolution of 1.5s. For this example we'll have 4 unique "stimulus conditions" that each occur for 2 TRs (3s) with 1 TR (1.5s) of rest between events.



.. code-block:: python


    from nltools.data import Design_Matrix
    import numpy as np

    dm = Design_Matrix(np.array([
                                [1,0,0,0],
                                [1,0,0,0],
                                [0,0,0,0],
                                [0,1,0,0],
                                [0,1,0,0],
                                [0,0,0,0],
                                [0,0,1,0],
                                [0,0,1,0],
                                [0,0,0,0],
                                [0,0,0,1],
                                [0,0,0,1]
                                ]),
                                sampling_rate = 1.5,
                                columns=['stim_A','stim_B','stim_C','stim_D']
                                )






Notice how this look exactly like a pandas dataframe. That's because design matrices are *subclasses* of dataframes with some extra attributes and methods.



.. code-block:: python


    print(dm)





.. rst-class:: sphx-glr-script-out

 Out::

    stim_A  stim_B  stim_C  stim_D
    0        1       0       0       0
    1        1       0       0       0
    2        0       0       0       0
    3        0       1       0       0
    4        0       1       0       0
    5        0       0       0       0
    6        0       0       1       0
    7        0       0       1       0
    8        0       0       0       0
    9        0       0       0       1
    10       0       0       0       1


Let's take a look at some of that meta-data. We can see that no columns have been convolved as of yet and this design matrix has no polynomial terms (e.g. such as an intercept or linear trend).



.. code-block:: python


    print(dm.details())





.. rst-class:: sphx-glr-script-out

 Out::

    nltools.data.design_matrix.Design_Matrix(sampling_rate=1.5, shape=(11, 4), convolved=[], polynomials=[])


We can also easily visualize the design matrix using an SPM/AFNI/FSL style heatmap



.. code-block:: python


    dm.heatmap()




.. image:: /auto_examples/01_DataOperations/images/sphx_glr_plot_design_matrix_001.png
    :align: center




A common operation might include adding an intercept and polynomial trend terms (e.g. linear and quadtratic) as nuisance regressors. This is easy to do. Note that polynomial terms are normalized to unit variance (i.e. mean = 0, std = 1) before inclusion to keep values on approximately the same scale.



.. code-block:: python


    # with include_lower = True (default), 1 here means: 0-intercept, 1-linear-trend, 2-quadtratic-trend
    dm_with_nuissance = dm.add_poly(2,include_lower=True)
    dm_with_nuissance.heatmap()




.. image:: /auto_examples/01_DataOperations/images/sphx_glr_plot_design_matrix_002.png
    :align: center




We can see that 3 new columns were added to the design matrix. We can also inspect the change to the meta-data. Notice that the Design Matrix is aware of the existence of three polynomial terms now.



.. code-block:: python


    print(dm_with_nuissance.details())





.. rst-class:: sphx-glr-script-out

 Out::

    nltools.data.design_matrix.Design_Matrix(sampling_rate=1.5, shape=(11, 7), convolved=[], polynomials=['intercept', 'poly_1', 'poly_2'])


Polynomial variables are not the only type of nuisance covariates that can be generate for you. Design Matrix also supports the creation of discrete-cosine basis functions ala SPM. This will create a series of filters added as new columns based on a specified duration, defaulting to 180s.



.. code-block:: python


    # Short filter duration for our simple example
    dm_with_cosine = dm.add_dct_basis(duration=5)
    print(dm_with_cosine.details())





.. rst-class:: sphx-glr-script-out

 Out::

    nltools.data.design_matrix.Design_Matrix(sampling_rate=1.5, shape=(11, 10), convolved=[], polynomials=['cosine_1', 'cosine_2', 'cosine_3', 'cosine_4', 'cosine_5', 'cosine_6'])


Load and Manipulate an Onsets File
-----------------------------------

Nltools provides basic file-reading support for 2 or 3 column formatted onset files. Users can look at the onsets_to_dm function as a template to build more complex file readers if desired or to see additional features. Nltools includes an example onsets file where each event lasted exactly 1 TR. Lets use that to create a design matrix with an intercept and linear trend



.. code-block:: python


    from nltools.utils import get_resource_path
    from nltools.file_reader import onsets_to_dm
    from nltools.data import Design_Matrix
    import os

    onsetsFile = os.path.join(get_resource_path(),'onsets_example.txt')
    dm = onsets_to_dm(onsetsFile, TR=2.0, runLength=160, sort=True,add_poly=1)
    dm.heatmap()




.. image:: /auto_examples/01_DataOperations/images/sphx_glr_plot_design_matrix_003.png
    :align: center




Design Matrix makes it easy to perform convolution and will auto-ignore all columns that are consider polynomials. By default it will use the one-parameter glover_hrf kernel (see nipy for details). However, any kernel can be passed as an argument, including a list of different kernels for highly flexible convolution across many types of data (e.g. SCR).



.. code-block:: python


    dm = dm.convolve()
    print(dm.details())
    dm.heatmap()




.. image:: /auto_examples/01_DataOperations/images/sphx_glr_plot_design_matrix_004.png
    :align: center


.. rst-class:: sphx-glr-script-out

 Out::

    nltools.data.design_matrix.Design_Matrix(sampling_rate=2.0, shape=(160, 15), convolved=['BillyRiggins', 'BuddyGarrity', 'CoachTaylor', 'GrandmaSaracen', 'JasonStreet', 'JulieTaylor', 'LandryClarke', 'LylaGarrity', 'MattSaracen', 'SmashWilliams', 'TamiTaylor', 'TimRiggins', 'TyraCollette'], polynomials=['intercept', 'poly_1'])


Load and Z-score a Covariates File
----------------------------------

Now we're going to handle a covariates file that's been generated by a preprocessing routine. First we'll read in the text file using pandas and convert it to a design matrix.



.. code-block:: python


    import pandas as pd

    covariatesFile = os.path.join(get_resource_path(),'covariates_example.csv')
    cov = pd.read_csv(covariatesFile)
    cov = Design_Matrix(cov, sampling_rate = 2.0)
    # Design matrix uses seaborn's heatmap for plotting so excepts all keyword arguments
    # We're just adjusting colors here to visualize things a bit more nicely
    cov.heatmap(vmin=-1,vmax=1)




.. image:: /auto_examples/01_DataOperations/images/sphx_glr_plot_design_matrix_005.png
    :align: center




Similar to adding polynomial terms, Design Matrix has multiple methods for data processing and transformation such as downsampling, upsampling, and z-scoring. Let's use the z-score method to normalize the covariates we just loaded.



.. code-block:: python


    # Use pandas built-in fillna to fill NaNs in the covariates files introduced during the pre-processing pipeline, before z-scoring
    # Z-score takes an optional argument of which columns to z-score. Since we don't want to z-score any spikes, so let's select everything except that column
    cov = cov.fillna(0).zscore(cov.columns[:-1])
    cov.heatmap(vmin=-1,vmax=1)




.. image:: /auto_examples/01_DataOperations/images/sphx_glr_plot_design_matrix_006.png
    :align: center




Concatenate Multiple Design Matrices
------------------------------------

A really nice feature of Design Matrix is simplified, but intelligent matrix concatentation. Here it's trivial to horizontally concatenate our convolved onsets and covariates, while keeping our column names and order.



.. code-block:: python


    full = dm.append(cov,axis=1)
    full.heatmap(vmin=-1,vmax=1)




.. image:: /auto_examples/01_DataOperations/images/sphx_glr_plot_design_matrix_007.png
    :align: center




But we can also intelligently vertically concatenate design matrices to handle say, different experimental runs, or participants. The method enables the user to indicate which columns to keep separated (if any) during concatenation or which to treat as extensions along the first dimension. By default the class will keep all polylnomial terms separated. This is extremely useful when building 1 large design matrix composed of several runs or participants with separate means.



.. code-block:: python


    dm2 = dm.append(dm, axis=0)
    dm2.heatmap(vmin=-1,vmax=1)




.. image:: /auto_examples/01_DataOperations/images/sphx_glr_plot_design_matrix_008.png
    :align: center




Specific columns of interest can also be kept separate during concatenation (e.g. keeping run-wise spikes separate). As an example, we treat our first experimental regressor as different across our two design matrices. Notice that the class also preserves (as best as possible) column ordering.



.. code-block:: python


    dm2 = dm.append(dm, axis=0, unique_cols=['BillyRiggins'])
    dm2.heatmap(vmin=-1,vmax=1)




.. image:: /auto_examples/01_DataOperations/images/sphx_glr_plot_design_matrix_009.png
    :align: center




Design Matrix can also create polynomial terms and intelligently keep them separate during concatenation. For example lets concatenate 4 design matrices and create separate 2nd order polynomials for all of them



.. code-block:: python


    # Notice that append can take a list of Design Matrices in addition to just a single one
    dm_all = dm.append([dm,dm,dm], axis=0, add_poly=2)
    dm_all.heatmap(vmin=-1,vmax=1)




.. image:: /auto_examples/01_DataOperations/images/sphx_glr_plot_design_matrix_010.png
    :align: center


.. rst-class:: sphx-glr-script-out

 Out::

    Design Matrix already has intercept...skipping
    Design Matrix already has 1th order polynomial...skipping
    Design Matrix already has intercept...skipping
    Design Matrix already has 1th order polynomial...skipping
    Design Matrix already has intercept...skipping
    Design Matrix already has 1th order polynomial...skipping
    Design Matrix already has intercept...skipping
    Design Matrix already has 1th order polynomial...skipping


Data Diagnostics
----------------

Design Matrix also provides a few tools for cleaning up perfectly correlated columns (resulting in failure if trying to perform regression), replacing data, and computing collinearity.



.. code-block:: python


    # We have a good design here so no problems
    dm_all.clean(verbose=False)
    dm_all.vif()




.. rst-class:: sphx-glr-script-out

 Out::

    Dropping columns not needed...skipping


**Total running time of the script:** ( 0 minutes  1.402 seconds)



.. only :: html

 .. container:: sphx-glr-footer


  .. container:: sphx-glr-download

     :download:`Download Python source code: plot_design_matrix.py <plot_design_matrix.py>`



  .. container:: sphx-glr-download

     :download:`Download Jupyter notebook: plot_design_matrix.ipynb <plot_design_matrix.ipynb>`


.. only:: html

 .. rst-class:: sphx-glr-signature

    `Gallery generated by Sphinx-Gallery <https://sphinx-gallery.readthedocs.io>`_
