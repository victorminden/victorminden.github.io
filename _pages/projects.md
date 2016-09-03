---
title: "Projects"
layout: single
excerpt: "List of projects."
sitemap: true
permalink: /projects.html
---

This is a collection of course projects and the like relatively tangential to my primary research.  Enjoy.


## Graduate

### Multifrontal solver

![CME 335 Project](images/mf.png)

 [Write-up](docs/project_cme335.pdf), [Code](https://github.com/victorminden/updating-multifrontal)

In Winter 2015 as a final project for my advanced topics in numerical linear algebra course with [Jack Poulson](http://web.stanford.edu/~poulson/), I implemented a general multifrontal solver with nested-dissection ordering in C\+\+ for sparse linear systems.

- - -

### Recycled LSMR

![Math 310 Project](images/rlsmr.png)

 [Write-up](docs/project_math310.pdf)

For my final project in a course on the top 10 algorithms of the 20th century, I implemented a recycled-subspace variant of the LSMR algorithm for solving linear systems.  Unfortunately, recycling did not seem to make much of a difference for my test problems, but I believe there could still be something here.

- - -


### Complex Krylov methods

![CME 338 Project](images/golubkahan.png)

 [Write-up](docs/project_cme338.pdf)

For our final project in large-scale numerical optimization with [Michael Saunders](http://stanford.edu/~saunders/) (Spring 2013), [Austin Benson](http://stanford.edu/~arbenson/) and I created complex implementations in FORTRAN of the popular LSQR and LSMR algorithms for solving linear systems.  These are based heavily off of the original real-arithmetic implementations from the Stanford Systems Optimization Laboratory and can be found [here (LSQR)](http://web.stanford.edu/group/SOL/software/lsqr/) and [here (LSMR)](http://web.stanford.edu/group/SOL/software/lsmr/).

- - -

### Optimal Solar Sailing

![CME 304 Project](images/sailoutput.png)

[Write-up](docs/solarsail.pdf)

The purpose of this project, from my Winter 2013 numerical optimization course with [Walter Murray](http://web.stanford.edu/~walter/), is to assess the optimal control and minimum time necessary to bring a spacecraft employing simple solar sail technologies from the Earth’s orbit around the sun to the orbit of Mars and back again. The problem is modified as a two-dimensional system in polar coordinates about the sun and Newton’s laws are assumed to be sufficient to model the required kinematics.
To format the minimum-time kinematics problem as a standard optimization problem, we solve a sequence of sub-problems where the final time is held fixed and a feasible trajectory is calculated. We consider the objective function to be the squared difference between the final point of the trajectory and the desired final state, and use a penalty function to account for the nonlinear constraints given by Newton’s laws. The optimal control is subject to box constraints.
Solving this optimization problem is done with an active-set BFGS method employing a line-search satisfying the strong Wolfe conditions.

- - -






## Undergraduate

### Photoluminescence Characterization of Solar Cells

![Senior Project](images/pl.png)

[Slides](docs/APS-PL.pdf)

For our senior design project, Emir Salih Magden and I worked with Professor Tom Vandervelde of [Tufts Renewable Energy and Applied Photonics Labs](http://reap.ece.tufts.edu/) to create a product to characterize the material quality of a semiconductor, for use in a research environment. Using computer-controlled linear actuators, a tunable band-pass filter, lenses, mirrors, and an optical sensor, we created a device controlled by LabView to move different points of a semiconductor sample into the path of a laser and measure the resulting photoluminescence.

- - -

### Inverse Heat Conduction

![Heat Conduction](images/heat.png)

[Write-up](docs/inverseproject.pdf)

Inverting the heat equation is a problem of great interest in the sciences and engineering, particularly for modeling and monitoring applications. For my final project for my class in inverse problems at Tufts, I looked at the heat equation defined on some domain containing a point heat source at a known location. Assuming the magnitude of the heat source to be an unknown function of time and given noisy transient measurements at other points in the domain, I looked into the use of the conjugate gradient method to solve the adjoint problem and recover the heat source magnitude function.

UPDATE: due to popular demand, here is the piece of Matlab code not included in the write-up.

MNtoI = @(m,n,M,N)  (n-1)*M+m;  
 

- - -

### Mathematical Contest in Modeling

![The Team](images/mcm.jpg)

From left to right: V. Minden, L. Clegg, D. Brady, S. MacLachlan.

- - -

![MCM 2011](images/kills.png)

[Write-up](docs/mcm1.pdf)

In 2010, I competed for the first time in the [COMAP Mathematical Contest in Modeling](http://www.comap.com/undergraduate/contests/mcm/), a contest in which teams of undergraduates are given four days to model, simulate, and write a report about a real-world problem revealed to teams on the first day of the competition. With Dan Brady and [Liam Clegg](http://liamclegg.com/), I created a discrete model for quantitative criminology applications  —  given a set of spatio-temporal points representing crimes in a spree, predict the times and locations of future crimes. Our entry, From Kills to Kilometers, won the designation of “Outstanding” as well as an external prize from INFORMS.

- - -

![MCM 2010](images/snow.png)

[Write-up](docs/mcm2.pdf)

In 2011, I competed again with a slightly different team. With Stephen Bidwell and Liam Clegg, I developed a model for generating snowboard halfpipe designs, with the goal of maximizing attainable vertical air. Our entry, Pipe Dreams, was awarded Honorable Mention.

- - -

