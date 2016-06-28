.. title: How to solve ODE in OpenFOAM
.. slug: ode
.. date: 2016-06-19 10:21:19 UTC+01:00
.. tags: mathjax, OpenFOAM, gnuplot, ODE, C++
.. category:
.. link:
.. description:
.. type: text
.. previewimage: /images/results.png


.. How to solve ODE in OpenFOAM
.. ----------------------------

OpenFOAM [#]_ has a library designed to solve ordinary differential equations (ODEs).
It is widely used for solving chemistry in OpenFOAM. This post focuses on
how to solve any ODE in OpenFOAM. it is not intended to be a complete review or
explaining every ODE solver, however, this post shows step-by-step how to define
an ODE and how to write a small program to solve it. Finally, the results will be
compared and plotted aginst the extract solution using Gnuplot.



.. TEASER_END: click to read the rest of the article


Mathematical background
-----------------------

.. class:: alert alert-info pull-right

.. contents::

In order to solve any high order ODE numerically, it should be reduced first to
a system of first order ODEs. This process of order reduction produces a number
of ODEs equal to the order of the original ODE. Here, the reduction process as
described by Ferziger [#]_ for general high order ODE,

    .. math::

        y^n = f(x,y,y',\cdots y^{n-1})

where :math:`n` is the order. Then it can be redefined as (transformation),


    .. math::

        y_1 &= y \\
        y_j &= y^{j-1} \quad j=1,2,\cdots n-1

Finally, the system could be represented as (differentiation),

    .. math::

        f_j = y'_j &= y_{j+1} \quad j=1,2,\cdots n-1\\
        f_n = y'_n &= f(x,y_1,y_2,\cdots y_n)

As described above, the process could be summarised into two steps; a
transformation of variables and differentiation. In OpenFOAM, there is an extra
step required for stiff system solvers only. These solvers require the jacobian
matrix and the time partial derivatives.

    .. math::

        J =
        \begin{bmatrix}
        \frac{\partial f_1}{\partial y_1} & \frac{\partial f_1}{\partial y_2} &
        \cdots &\frac{\partial f_1}{\partial y_n}\\
        \frac{\partial f_2}{\partial y_1} & \frac{\partial f_2}{\partial y_2} &
        \cdots &\frac{\partial f_2}{\partial y_n}\\
        \vdots & \vdots & \ddots & \vdots \\
        \frac{\partial f_n}{\partial y_1} & \cdots & \cdots
        &\frac{\partial f_n}{\partial y_n}
        \end{bmatrix}

Where :math:`f_n` is the equation :math:`n` in the reduced system. Also the partial derivative with respect to :math:`x` (independent variable)

    .. math::

        \frac{\partial f_1}{\partial x} \cdots \frac{\partial f_n}{\partial x}

Equation of motion
-------------------

The equation of motion of single degree of freedom system (simple mass-spring
system) , which is a second order ODE, will be used here as an example.

    .. math::

        m\ddot{q} + kq &= 0 \\
        \ddot{q} &= -\frac{k}{m}q

where :math:`m` is the mass, :math:`k` is the stiffness and :math:`q` is the
displacement.

This equation has an exact solution represents the harmonic oscillation of the
system as (assuming zero initial velocity);

    .. math::

        q = q_0 \cos(\omega t)

where :math:`q_0` is the initial displacement and :math:`\omega` is the natural
frequency of the system :math:`\omega=\sqrt{k/m}`.


It is the time to apply the methodology described in the previous section (`Mathematical background`_) on the
equation of motion.

- `Step A`_: Transformation from :math:`q` to :math:`y`

    .. _Step A:

    .. math::

        y_1 &= q \\
        y_2 &= \dot{q}

- `Step B`_: Differentiation (substitute first)

    .. _Step B:

    .. math::

        \ddot{q} = -\frac{k}{m}y_1

    then,

    .. math::

        f_1 = \dot{y_1} &= \dot{q} \\
                        &= y_2\\
        f_2 = \dot{y_2} &= \ddot{q} \\
                        &= -\frac{k}{m}y_1

- `Step C`_: Jacobian (optional)

    .. _Step C:

    The Jacobian matrix is required for stiff solvers only. For this case,
    it is just :math:`(2\times2)` matrix,

    .. math::

        J =
        \begin{bmatrix}
        0.0 & 1.0 \\
        -k/m & 0
        \end{bmatrix}

    and time derivatives (independent variable in this case),

    .. math::

        \frac{\partial f_1}{\partial t} &= 0 \\
        \frac{\partial f_2}{\partial t} &= 0


Where to start?
---------------

It is typical in OpenFOAM to search for a similar implementation as a starting
point. Fortunately, there is a small test program included in OpenFOAM source
files. It is described in details in [#]_. So copy it to your
run directory then compile and run it. (``RKCK45`` is one of OpenFOAM ODE
solvers).

.. code:: console

    $ cp -r $FOAM_APP/test/ODE .
    $ wmake
    $ Test-ODE RKCK45


You should get a lot of numbers on the screen and finally you should see;

.. code:: console

    Analytical: y(2.0) = 4(0.223891 0.576725 0.352834 0.128943)
    Numerical:  y(2.0) = 4(0.223891 0.576725 0.352835 0.128943), dxEst = 0.402302


.. class:: alert alert-info

    Please note that since OpenFOAM-2.3 version, the ordinary differential equation
    solvers have been updated [#]_. Therefore Zongyuan [3]_ report has some
    outdated parts.


Implementation
--------------

The reduction step is essential to solve the equation and as it is described
above. This step is important in OpenFOAM and in any other code like python or
Octave/Matlab. The implementation in OpenFOAM is divided into two main classes;
``ODEsystem`` and ``ODEsolver``. From its name you can guess the function of
each class. ``ODEsystem`` is an abstract class defines the system of first order
ODEs as explained above. ``ODEsolver`` is the base class for all the ODE solvers
in OpenFOAM.

Back to ``Test-ODE.C``, open it in any text editor (I prefer atom or qt-creator for
big projects) and start examining the code. You will notice that the code is
divided into two parts, the first part is the definitionn of class called
``testODE`` and the second part is the ``main`` function.

``myODE``
~~~~~~~~~

testODE is basically the ODE system definition which is inherited from the base
abstract class ``ODEsystem``. Let's modify it to represent our equation. Before we
start just rename it myODE. Also, modify the class constructor to allow passing
:math:`m` and :math:`k` values from the main application.

.. code:: c++

    class myODE
    :
    public ODESystem
    {
        //- Mass of the system
        const scalar m_;
        //- Stiffness of the system
        const scalar k_;

        public:

        myODE(const scalar& mass, const scalar& stiffness)
        :
        ODESystem(),
        m_(mass),
        k_(stiffness)
    {}


This class has only three functions, namely, ``nEqns``,
``derivatives`` and ``jacobian``. ``nEqns`` represents the number of equation of this
system which is essentially the order of the original equation (for this case
is :math:`2`).

.. code:: c++

    label nEqns() const
    {
        return 2;
    }


The second function is ``derivatives`` which is the definition of the system of
the first order ODEs (`Step B`_);

.. code:: c++

    void derivatives
    (
        const scalar x,
        const scalarField& y,
        scalarField& dydx
    ) const
    {
        dydx[0] = y[1];
        dydx[1] = -(k_/m_+VSMALL)*y[0]
    }

The third and final function in this class is ``jacobian`` and it represents the
jacobian matrix (`Step C`_) and the time derivatives of the system.

.. code:: c++

    void jacobian
    (
        const scalar x,
        const scalarField& y,
        scalarField& dfdx,
        scalarSquareMatrix& dfdy
    ) const
    {
        dfdx[0] = 0.0;
        dfdx[1] = 0.0;

        dfdy[0][0] = 0.0;
        dfdy[0][1] = 1.0;

        dfdy[1][0] = -(k_/m_+VSMALL);
        dfdy[1][1] = 0;
    }


``main`` program
~~~~~~~~~~~~~~~~~~

Now we have ``myODE`` and it is time to solve the system. The first line inside the
main function is basically to allow the program to read the ``ODEsolver`` name
from the command line. We will keep it and add extra two parameters for :math:`m`
and :math:`k` (lines 7 & 8).

Please read the comments inside the code carefully.  One point could cause
confusion here is ``yStart`` (line 31). It is ``scalarField`` has the same size as the system
(order). ``yStart`` has two functions here, the fist one is applying the initial
conditions. In our case and according to (`Step A`_), these initial conditions are
the system initial displacement and velocity. The second function of ``yStart``
is storing the output results of the ODEsolver.

.. code:: c++
    :linenos:

    int main(int argc, char *argv[])
    {
        argList::validArgs.append("ODESolver");
        argList args(argc, argv);

        // system properties (mass and stiffness)
        const scalar mass = 2.0; // Kg
        const scalar stiff = 2048.0; // N/m
        // Initial displacement
        const scalar y0 = 0.2; // m
        const label n = 200;
        const scalar endTime = 1.0; // s

        // Create the ODE system as object of type myODE
        myODE ode(mass,stiff);

        // Create dictionary and add the odeSolver name
        dictionary dict;
        dict.add("solver", args[1]);

        // Create the selected ODE system solver
        autoPtr<ODESolver> odeSolver = ODESolver::New(ode, dict);

        // Initialise the ODE system fields

        // Start time
        scalar xStart = 0.0;
        // time step
        const scalar dx = endTime/n;
        // Initial displacement and velocity
        scalarField yStart(ode.nEqns());
        yStart[0] = y0;
        yStart[1] = 0.0;
        // integration initial step
        scalar dxEst = 0.1;
        scalar xEnd = 0;

        // Required to store dydx
        scalarField dyStart(ode.nEqns());

        // Integration loop
        for (label i=0; i<n; i++)
        {
            xEnd = xStart + dx;
            ode.derivatives(xStart, yStart, dyStart);
            odeSolver->solve(xStart, xEnd, yStart, dxEst);
            xStart = xEnd;
            Info << xStart << "   " << yStart[0] << endl;
        }

        Info<< "\nEnd\n" << endl;

        return 0;
    }

Finally, compile the code and run it. Direct the output to a file so it could be
used for postprocessing.

.. code:: console

    $ wmake
    $ Test-ODE RKCK45 > log

Validation
----------

As mentioned before (`Equation of motion`_), this ODE has an exact solution.
It is basically a harmonic oscillation which could be represented as
(after applying the initial conditions and system properties),

    .. math::

        q(t) = 0.2 \cos(32t)

A quick and easy way to compare the results is by using gnuplot to visualise
the results.

.. figure:: /images/results.png
    :target: /images/results.png


:Gnuplot Script:

.. code:: bash

    # system properties
    mass = 2.0
    stiff = 2048.0
    x0 = 0.2

    set grid
    set samples 200
    set xlabel 'Time (s)'
    set ylabel 'Displacement (m)'
    set yrange [-0.3:0.3]

    f(x) = x0*cos(sqrt(stiff/mass)*x)
    plot 'log' u 1:2 w p lw 2 t "OpenFOAM", f(x) w l lw 2 t "Exact"

Final remarks
-------------

This post is meant to be minimal as possible and to be focused on solving ODE.
Therefore, there are many possibilities to improve the main program. The system
properties can be read from the command line directly or , even better, all the
inputs could be included in one ``dictionary`` (OpenFoam style). Also, the output
result could be formatted in a better elegant way. Moreover, the results can be
written to a file. Finally, the ``solve`` function is an overloaded function, so
it can be used with different input argument.

.. class:: alert alert-info

    Please feel free to comment below. Your feedback will be highly appreciated.

.. class:: alert alert-info

    - Update: 23 June 2016, scalar type added to myODE constructor as pointed out by Mohamed Ouda
    - Update: 28 June 2016, variables names are revised to be consistent and more clear according to Francisco Angel comments

.. [#]  OpenFOAM® and OpenCFD® are registered trademarks of OpenCFD Limited,
        the producer OpenFOAM software. All registered trademarks are property
        of their respective owners. This offering is not approved or endorsed
        by OpenCFD Limited, the producer of the OpenFOAM software and owner
        of the OPENFOAM® and OpenCFD® trade marks.
        Hassan Kassem is not associated to OpenCFD.

.. [#] Ferziger, Joel H. 1998. Numerical Methods for Engineering Applications. Wiley.

.. [#] Zongyuan, Gu 2009. `Introduction to ODE solvers and their application in OpenFOAM`_. Chalmers.

.. _Introduction to ODE solvers and their application in OpenFOAM: http://www.tfd.chalmers.se/~hani/kurser/OS_CFD_2008/ZongyuanGu/reportZongyuan.pdf

.. [#] `Numerical methods`_. OpenFOAM 2.3.0 relase notes 2014.
.. _Numerical methods: http://openfoam.org/release/2-3-0/numerics/


.. raw:: html

    <div data-social-share-privacy='true'></div>
