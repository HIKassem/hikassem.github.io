.. title: How to add a new turbulence model, revisited 
.. slug: newturbulencemodel6
.. date: 2018-10-28 11:55:47 UTC
.. tags: OpenFOAM, C++, turbulenceModels, template
.. category: 
.. link: 
.. description: 
.. type: text
.. previewimage: /images/template6.png

.. figure:: /images/template6.png
    :target: /images/template6.png

|

In this post, :doc:`my old tutorial <newturbulencemodel>` on how to implement a new turbulence model in OpenFOAM-3.0 is revisited.
The old post is over two years old and since then a lot of changes in OpenFOAM 
have been introduced by OpenFOAM foundation.
Fortunately, implementing a new model in OpenFOAM-5.0 upwards is much easier now.
Therefore, this will be a very short post since all the basics has been covered 
in the previous post.

.. TEASER_END: click to read the rest of the article

``komegaSST`` as an example
----------------------------

Unlike the old post, ``kOmegaSST`` is used as an example. It seems that many 
OpenFOAMers are using ``kOmegaSST`` as a base model for their development which is
quite understandable due to the popularity of the model in many applications. 
Also, that has been reflected in OpenFOAM implementation of the model since ``version-4.0``.
The model has been reimplemented as base model for all ``kOmegaSST`` based models 
and called  ``kOmegaSSTbase``. The model governing equations are implemend in the 
base class ``kOmeagaSSTbase`` [#]_.

As usual, we need to copy few files from OpenFOAM source to a local directory of 
your choice, typically OpenFOAM user run is a reasonable option. Open a new 
terminal and source OpenFOAM-5.0 or OpenFOAM-6.0. Then change directory to any folder
where you want to save your code. The follow the commend below to copy and rename the files.

.. code:: console

    cp -r $FOAM_SRC/TurbulenceModels/turbulenceModels/RAS/kOmegaSST newkOmegaSST
    cd newkOmegaSST
    mv kOmegaSST.H newkOmegaSST.H
    mv kOmegaSST.C newkOmegaSST.C
    cp -r $FOAM_SRC/TurbulenceModels/incompressible/Make/ .
    cp -r $FOAM_SRC/TurbulenceModels/incompressible/turbulentTransportModels/turbulentTransportModels.C maketurbulentTransportModels.C

It is the time to modify the source code, first edit ``maketurbulentTransportModels.C``.
Just keep the include statement in the beginning and remove everything else and add these two lines,

.. code:: c++

    #include "newkOmegaSST.H"
    makeRASModel(newkOmegaSST);


The ``Make/files`` should point to ``maketurbulentTransportModels.C`` as follows,

.. code:: console

    maketurbulentTransportModels.C

    LIB = $(FOAM_USER_LIBBIN)/libmyincompressibleTurbulenceModels

The ``Make/options`` should links the following libraries as,

.. code:: console

    EXE_INC = \
        -I$(LIB_SRC)/TurbulenceModels/turbulenceModels/lnInclude \
        -I$(LIB_SRC)/TurbulenceModels/incompressible/lnInclude \
        -I$(LIB_SRC)/transportModels \
        -I$(LIB_SRC)/finiteVolume/lnInclude \
        -I$(LIB_SRC)/meshTools/lnInclude \

    LIB_LIBS = \
        -lincompressibleTransportModels \
        -lturbulenceModels \
        -lfiniteVolume \
        -lmeshTools

.. class:: alert alert-info

    Now replace every instance of ``kOmegaSST`` to ``newkOmegaSST`` in both ``newkOmegaSST.H`` and ``newkOmegaSST.H``.
    Please be careful, do not use ``sed`` nor replace all functionally in your editior 
    without reviewing the changes. As discussed before, all ``kOmegaSST`` based models 
    are inherited from a base class called ``kOmegaSSTbase``. 
    Therefore, do not change these instance where ``kOmegaSSTbase`` appears ( line[41] [#]_) nor the 
    base type in class declaration ( line[59] [#]_) and definition ( line[50] [#]_). 

Just add the new library to ``controlDict`` libs and you are ready to go!!

.. [#] `kOmeagaSSTbase`_
.. _kOmeagaSSTbase: https://github.com/OpenFOAM/OpenFOAM-5.x/tree/master/src/TurbulenceModels/turbulenceModels/Base/kOmegaSST

.. [#] `Line 41 in kOmegaSST.H`_
.. _Line 41 in kOmegaSST.H: https://github.com/OpenFOAM/OpenFOAM-5.x/blob/7f7d351b741bf6406366a043cac98de56d2d44dd/src/TurbulenceModels/turbulenceModels/RAS/kOmegaSST/kOmegaSST.H#L41

.. [#] `Line 59 in kOmegaSST.H`_
.. _Line 59 in kOmegaSST.H: https://github.com/OpenFOAM/OpenFOAM-5.x/blob/7f7d351b741bf6406366a043cac98de56d2d44dd/src/TurbulenceModels/turbulenceModels/RAS/kOmegaSST/kOmegaSST.H#L59

.. [#] `Line 50 in kOmegaSST.C`_
.. _Line 50 in kOmegaSST.C: https://github.com/OpenFOAM/OpenFOAM-5.x/blob/7f7d351b741bf6406366a043cac98de56d2d44dd/src/TurbulenceModels/turbulenceModels/RAS/kOmegaSST/kOmegaSST.C#L50
