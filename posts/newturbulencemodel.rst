.. title: How to add a turbulence model in OpenFOAM-3.0.0
.. slug: newturbulencemodel
.. date: 2016-06-25 18:42:41 UTC+01:00
.. tags: OpenFOAM, C++, turbulenceModels, template
.. category:
.. link:
.. description:
.. type: text

In OpenFOAM-3.0.0 [#]_, the turbulence models library had major facelift. Actually, it had major operation not just a facelift. Now, all models are based on one base template class including RANS, LES, incompressible and compressible. Therefore, if you followed most of the available tutorials on how to modify/implement turbulence model, it will not work. You will face many compiler errors from the first step. This post shows how to overcome this problem step-by-step. Moreover, it discusses briefly the reasons causing these errors.

.. TEASER_END: click to read the rest of the article

What is the problem?
--------------------

Back in January 2015, a new commit [#]_ was pushed to OpenFOAM-dev repository on gitHub [#]_. This update was relased in November 2015 with OpenFOAM-3.0.0 [#]_. The new implementation is based on multilayer ``template`` class [#]_.
If you normally consider a ``class`` as a blueprint of an object, then you can consider ``template`` class as a blueprint of a blueprint of an object. An object of class is usually initialised in the main program and the compiler does not have to know anything about it during the compilation process of the class. Unlike a ``template`` which is written by the compiler.

First let's compare how add a turbulence model in the old versions and what will happen if we followed the same process in OpenFOAM-3.0.0.

Previous versions
~~~~~~~~~~~~~~~~~~~~~

As you may read in many available tutorials online about how to create a new turbulence model in OpenFOAM, you just have to follow the these steps to create incompressible turbulence model based on ``realizableKE``

.. code:: console

    # copy to run directory
    $ cp -r $FOAM_SRC/turbulenceModels/incompressible/RAS/realizableKE/ .
    # rename all files
    $ mv realizableKE myrealizableKE
    $ mv myrealizableKE/realizableKE.H myrealizableKE/myrealizableKE.H
    $ mv myrealizableKE/realizableKE.C myrealizableKE/myrealizableKE.C
    # rename all instances of realizableKE to myrealizableKE
    # you can use replace all in your text editor
    $ find myrealizableKE -type f -exec sed -i 's/realizableKE/myrealizableKE/g' {} \;

Then a Make directory is needed to compile the new mode,

``Make/files``

.. code:: console

    myrealizableKE.C

    LIB = $(FOAM_USER_LIBBIN)/libmyrealizableKE

``Make/options`` (you can copy it from ``$FOAM_SRC/turbulenceModels/incompressible/RAS/Make/options`` and link it to ``RASModels`` library)

.. code:: console

    EXE_INC = \
        -I$(LIB_SRC)/turbulenceModels/incompressible/RAS/lnInclude \
        -I$(LIB_SRC)/turbulenceModels \
        -I$(LIB_SRC)/transportModels \
        -I$(LIB_SRC)/finiteVolume/lnInclude \
        -I$(LIB_SRC)/meshTools/lnInclude

    LIB_LIBS = \
        -lincompressibleRASModels \
        -lincompressibleTurbulenceModel \
        -lfiniteVolume \
        -lmeshTools

For compressible flow, it is the same process with minor modifications to ``Make``. That is all you need, just add the new library to ``controlDict`` libs and you are ready to go.

New version
~~~~~~~~~~~
let's try to apply the same steps using any OpenFOAM version starting from ``v-3.0.0``

.. code:: console

    # note: Diffrent path.
    $ cp -r $FOAM_SRC/TurbulenceModels/turbulenceModels/RAS/realizableKE .

Then follow the same steps as `Previous versions`_. If you still not sure how to update your case files to ``v-3.0.0``, check this post [#]_

First error
^^^^^^^^^^^

If you followed these steps you will get a very long error but actually it is just one repeated error like this one,

.. code:: console

    myrealizableKE.C:###:##: error: redefinition of 'function name'
    myrealizableKE.C:###:##: note: 'function name' previously declared here

Probably you are familiar with such error. It basically means that the member functions of class ``myrealizableKE`` are defined more than once. Indeed the compiler never lies unless it has a bug. The first idea could be tried to over come this error is removing the following three lines in the ``myrealizableKE.H`` (`comment them out we will need them later`). You maybe do not understand these lines but ``.C`` file included in ``.H`` file in ``C++`` `looks weird!!`

.. code:: c++

    // #ifdef NoRepository
    // #   include "myrealizableKE.C"
    // #endif

Now recompile the code, `surprisingly no errors!!`

Second error
^^^^^^^^^^^^
However this code was compiled without any errors, it will not work. That could be confirmed by linking the library to any case and select the new turbulence model ``myrealizableKE``. Simply the solver will not recognise the new model. Now, uncomment the three lines and save all files and let's solve this problem.

Solution
---------
Definitely copy and rename strategy does not work here. There are two reasons causing this problem and you may face if you tried to modify any of the core ``template`` libraries in OpenFOAM. The first reason is the fact that ``myrealizableKE`` is  a ``template`` class. The second reason, it is not included in the run time table [#]_.

RANS incompressible models
~~~~~~~~~~~~~~~~~~~~~~~~~~

Follow the same steps as described in `New version`_ section but an extra file is needed which you can copy from OpenFOAM source

.. code:: console

    $ cp $FOAM_SRC/TurbulenceModels/incompressible/turbulentTransportModels/turbulentTransportModels.C makeTurModel.C

In simple words, this file has two tasks. The first task this file will do is inislize the template class [#]_ [#]_. The second task is adding the new model to the run time selection table.

Edit ``makeTurModel.C`` (Please note this file is based on `turbulentTransportModels.C`_ and `makeTurbulenceModel.H`_)

.. _turbulentTransportModels.C: https://github.com/OpenFOAM/OpenFOAM-dev/blob/master/src/TurbulenceModels/incompressible/turbulentTransportModels/turbulentTransportModels.C

.. _makeTurbulenceModel.H: https://github.com/OpenFOAM/OpenFOAM-dev/blob/master/src/TurbulenceModels/turbulenceModels/makeTurbulenceModel.H

.. code:: c++

    #include "IncompressibleTurbulenceModel.H"
    #include "transportModel.H"
    #include "addToRunTimeSelectionTable.H"
    #include "makeTurbulenceModel.H"

    #include "laminar.H"
    #include "RASModel.H"
    #include "LESModel.H"

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //
    #define createBaseTurbulenceModel(Alpha, Rho, baseModel, BaseModel, Transport) \
                                                                                   \
        namespace Foam                                                             \
        {                                                                          \
            typedef BaseModel<Transport> Transport##BaseModel;                     \
            typedef RASModel<Transport##BaseModel> RAS##Transport##BaseModel;      \
            typedef LESModel<Transport##BaseModel> LES##Transport##BaseModel;      \
        }

    createBaseTurbulenceModel
    (
        geometricOneField,
        geometricOneField,
        incompressibleTurbulenceModel,
        IncompressibleTurbulenceModel,
        transportModel
    );

    #define makeRASModel(Type)                                                     \
        makeTemplatedTurbulenceModel                                               \
        (transportModelIncompressibleTurbulenceModel, RAS, Type)

    #define makeLESModel(Type)                                                     \
        makeTemplatedTurbulenceModel                                               \
        (transportModelIncompressibleTurbulenceModel, LES, Type)

   #include "myrealizableKE.H"
   makeRASModel(myrealizableKE);

Also modify ``files`` to compile ``makeTurModel.C``

.. code:: console

    makeTurModel.C

    LIB = $(FOAM_USER_LIBBIN)/libmyrealizableKE

Finally ``options``

.. code:: console

    EXE_INC = \
        -I$(LIB_SRC)/TurbulenceModels/turbulenceModels/lnInclude \
        -I$(LIB_SRC)/TurbulenceModels/incompressible/lnInclude \
        -I$(LIB_SRC)/transportModels/incompressible/lnInclude \
        -I$(LIB_SRC)/finiteVolume/lnInclude \
        -I$(LIB_SRC)/meshTools/lnInclude

    LIB_LIBS = \
        -lturbulenceModels \
        -lincompressibleTurbulenceModels \
        -lincompressibleTransportModels \
        -lfiniteVolume \
        -lmeshTools

RANS compressible models
~~~~~~~~~~~~~~~~~~~~~~~~

Here you can see a glimpse of the beauty of ``template`` classes in ``C++`` `(You do not have to agree with me on this!)`. The same source files ``myrealizableKE.H`` and ``myrealizableKE.C`` can be used to compile the same model but for compressible flow. Only ``makeTurModel.C`` and ``Make`` need to be modified as follows, (Please note this file is based on `turbulentFluidThermoModels.C`_ and `compressible/makeTurbulenceModel.H`_)

.. _turbulentFluidThermoModels.C: https://github.com/OpenFOAM/OpenFOAM-dev/blob/master/src/TurbulenceModels/compressible/turbulentFluidThermoModels/turbulentFluidThermoModels.C

.. _compressible/makeTurbulenceModel.H: https://github.com/OpenFOAM/OpenFOAM-dev/blob/master/src/TurbulenceModels/compressible/turbulentFluidThermoModels/makeTurbulenceModel.H


.. code:: c++

    #include "CompressibleTurbulenceModel.H"
    #include "compressibleTransportModel.H"
    #include "fluidThermo.H"
    #include "addToRunTimeSelectionTable.H"
    #include "makeTurbulenceModel.H"

    #include "ThermalDiffusivity.H"
    #include "EddyDiffusivity.H"

    #include "laminar.H"
    #include "RASModel.H"
    #include "LESModel.H"

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //
    #define createBaseTurbulenceModel(                                             \
        Alpha, Rho, baseModel, BaseModel, TDModel, Transport)                      \
                                                                                   \
        namespace Foam                                                             \
        {                                                                          \
            typedef TDModel<BaseModel<Transport>>                                  \
                Transport##BaseModel;                                              \
            typedef RASModel<EddyDiffusivity<Transport##BaseModel>>                \
                RAS##Transport##BaseModel;                                         \
            typedef LESModel<EddyDiffusivity<Transport##BaseModel>>                \
                LES##Transport##BaseModel;                                         \
        }

    createBaseTurbulenceModel
    (
        geometricOneField,
        volScalarField,
        compressibleTurbulenceModel,
        CompressibleTurbulenceModel,
        ThermalDiffusivity,
        fluidThermo
    );

    #define makeRASModel(Type)                                                     \
        makeTemplatedTurbulenceModel                                               \
        (fluidThermoCompressibleTurbulenceModel, RAS, Type)

    #define makeLESModel(Type)                                                     \
        makeTemplatedTurbulenceModel                                               \
        (fluidThermoCompressibleTurbulenceModel, LES, Type)

    #include "myrealizableKE.H"
    makeRASModel(myrealizableKE);

In ``files`` just change the name of the compiled dynamic library and change ``options`` to

.. code:: console

    EXE_INC = \
        -I$(LIB_SRC)/TurbulenceModels/compressible/lnInclude \
        -I$(LIB_SRC)/TurbulenceModels/turbulenceModels/lnInclude \
        -I$(LIB_SRC)/transportModels/compressible/lnInclude \
        -I$(LIB_SRC)/thermophysicalModels/basic/lnInclude \
        -I$(LIB_SRC)/thermophysicalModels/specie/lnInclude \
        -I$(LIB_SRC)/thermophysicalModels/solidThermo/lnInclude \
        -I$(LIB_SRC)/thermophysicalModels/solidSpecie/lnInclude \
        -I$(LIB_SRC)/finiteVolume/lnInclude \
        -I$(LIB_SRC)/meshTools/lnInclude \

    LIB_LIBS = \
        -lcompressibleTurbulenceModels \
        -lcompressibleTransportModels \
        -lfluidThermophysicalModels \
        -lsolidThermo \
        -lsolidSpecie \
        -lturbulenceModels \
        -lspecie \
        -lfiniteVolume \
        -lmeshTools

The order of first two links in ``EXE_INC`` is `essential`.

Final Remarks
-------------
The introduced ``createBaseTurbulenceModel`` macro is based on ``makeBaseTurbulenceModel`` which is included in OpenFOAM source code. It can be used with LES models as well, but with using ``makeLESModel`` instead of ``makeRASModel``. Hopefully in the near future, I will explain this solution in more details. If you interested in adding new phaseCompressible turbulence model, please check my post on CFD-online [#]_.

.. class:: alert alert-info

    Please feel free to comment below. Your feedback will be highly appreciated.

.. [#]  OpenFOAM® and OpenCFD® are registered trademarks of OpenCFD Limited,
        the producer OpenFOAM software. All registered trademarks are property
        of their respective owners. This offering is not approved or endorsed
        by OpenCFD Limited, the producer of the OpenFOAM software and owner
        of the OPENFOAM® and OpenCFD® trade marks.
        Hassan Kassem is not associated to OpenCFD.

.. [#] `commit 93732c8af4a545c617399600ee810081fdb42b07`_
.. _commit 93732c8af4a545c617399600ee810081fdb42b07: https://github.com/OpenFOAM/OpenFOAM-dev/commit/93732c8af4a545c617399600ee810081fdb42b07

.. [#] `OpenFOAM-dev`_
.. _OpenFOAM-dev: https://github.com/OpenFOAM/OpenFOAM-dev

.. [#] `OpenFOAM 3.0.0 relase notes 2015`_
.. _OpenFOAM 3.0.0 relase notes 2015: http://openfoam.org/release/3-0-0/

.. [#] `Templates and Template Classes in C++`_
.. _Templates and Template Classes in C++:   http://www.cprogramming.com/tutorial/templates.html

.. [#] `Updating OpenFOAM case files for 3.0.x`_
.. _Updating OpenFOAM case files for 3.0.x: http://petebachant.me/updating-openfoam-case-files-for-30x/

.. [#] `Run-Time Type Selection Series`_, sourceflux blog
.. _Run-Time type selection series: http://www.sourceflux.de/blog/series/rts-2/

.. [#] `Compiling Template Classes`_
.. _Compiling Template Classes : https://www.cs.umd.edu/class/fall2002/cmsc214/Projects/P2/proj2.temp.html

.. [#] `How To Organize Template Source Code`_
.. _How To Organize Template Source Code: http://www.codeproject.com/Articles/3515/How-To-Organize-Template-Source-Code

.. [#] `Adding New phaseCompressible Turbulence Model`_
.. _Adding New phaseCompressible Turbulence Model: http://www.cfd-online.com/Forums/openfoam-programming-development/170283-adding-new-phasecompressible-turbulence-model.html#post606552

.. raw:: html

    <div data-social-share-privacy='true'></div>
