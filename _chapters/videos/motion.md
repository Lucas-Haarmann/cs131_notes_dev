---
title: Structure From Motion
keywords: (Motion, SfM, Structure, Incremental SfM)
order: 12
---

## SfM Problem Definition
Structure from Motion (SfM) is a computer vision technique that uses geometric principles to perform 3D reconstruction on a set of images. It asks the following questions:
* Can we do 3D reconstruction when we have nothing besides images (no intrinsics, no extrinsics, 3D points)?
* Can we generalize a 3D reconstruction solution to use more than two images?

#### Motivation: Why do we care about 3D reconstruction?
There are many use cases for 3D recon

1. **Navigation in Robotics, Drones, and Cars**. 3D reconstruction helps these devices  understand the 3D world better from 2D images captured by cameras.
2. **Augmented Reality (AR)**. 3D reconstruction helps localize your phone in the 3D world around it in order to place images realistically within a 3D scene.
3. **Movies, Digital Preservation, "Photo Tourism."** 3D reconstruction is also useful for special FX and digital scanning of artificats.

#### Review: Calibration and Triangulation/Multi-view Stereo
**Problem Set Up:** We have a set of known 2D points in an image and a set of its corresponding 3D points in the real world

**Task:** Estimate the camera parameters (K, R, t)

<img src="https://imgur.com/MYqOjCE.png" width="400"/>
Figure: Calibration problem setup ()

**Problem Set Up:** We have a 2D point represented in multiple images taken from different cameras. We also know the camera parameters for each camera.

**Task:** Estimate the corresponding 3D point.

<img src="https://imgur.com/TaJAfon.png" width="400"/>
Figure: Triangulation problem set up ()

#### Structure from Motion
**Problem Set Up:** We know nothing. We are just given a set of images from a static scene.

**Task:** Estimate the 3D points (i.e. the structure) and the camera parameters (i.e. motion)

<img src="https://imgur.com/HvaMLkb.png" width="400"/>
Figure: SfM problem set up ()

Given many images, we want to know how can we:
1. figure out where they were all taken from?
2. build a 3D model of the scene?

This is called the Structure from Motion problem

Input: images with 2D points $x_{ij}$ in correspondence
Output (solved simultaneously):
* Structure: 3D Location $X_{ij}$ 
* Motion: camera parameters $R_i$, $t_i$, and possibly $K_i$

Objective Function: Minimize reprojection error in 2D

One thing to note is that we don't know the ground truth labels. We are trying to find a way to explain the scene that is consistent with the geometry.

That means we don't know for sure if the camera parameters K, R, and T are correct. However, we do know if they're incorrect if they're inconsistent with gemoetric principles such as the epipolar lines. We can minimize that error in our objective function to find a best approximation of the camera parameters and corresponding 3D points.

## Finding 2D Correspondences
The first step of generating the input to SfM is detecting features in the image set, which is typically done using SIFT but could also be done with methods such as deep learning or ORB-SLAM. The local features are then matched between pairs of images by representing the interest points as vectors and finding their nearest neighbors. The matching is made more robust by using RANSAC to counteract noise caused by correspondence problems (e.g. textureless surfaces, occlusions, repetitions, specularities). The epipolar constraint, $x’^{T}Fx = 0$, is utilized in matching because $x$ and $x’$ correspond to the same fixed 3D point.

<img src="https://i.imgur.com/CkIRNJH.png" width="400"/>
Figure: Point correspondence geometry (lecture 11, slide 50)

Once the outliers have been removed, the fundamental matrix between each pair of images can be calculated.

#### Correspondence Estimation
A connected component is a 2D correspondence that is matched across multiple images like in the figure below that corresponds to a single 3D point present in all these images.
<img src="https://i.imgur.com/95J9mAt.png" width="400"/>
Figure: Graph of nearest neighbors using 2D correspondences in an image set (lecture 12, slide 22)

<img src="https://i.imgur.com/E9LggJh.png" width="400"/>
Figure: Connected component across images with 2D correspondences (lecture 12, slide 23)

These connected components are found by searching the graph created in feature matching.

## Solving for Cameras and 3D Points
#### Projective Structure from Motion
We will now consider multiple uncalibrated images of the same scene, from different cameras. Each image contains a certain number of fixed 3D points, which we will assume remain visible in all images. For the sake of this section, we will work with $m$ images with $n$ points. Therefore, we would like to compute a projection matrix, $P_i$, for each of the $m$ images. For a given 3D point $X_j$, the corresponding point on image $i$ is given by: $x_{ij}\sim P_iX_j$. We therefore need to estimate the values of the projection matrix $P_i$ and the 3D point $X_j$ using the 2D correspondences $x_{ij}$ alone.

Since we don't have any calibration information, we can only resolve the cameras and points up to a single projected transformation of of $X$, which we will call $Q$. In this case, we can replace $X$ with $QX$ and we can replace $P$ with $PQ^{-1}$. Then, returning to our original equation, we get $x\sim PQ^{-1}QX=PX$. We therefore cannot solve for $Q$. We therefore need extra information to find the exact values of $P$ and $X$.

We have $2mn$ constraints (since each $x_{ij}$ has two coordinate values). The matrix $P$ has $11m+3n-15$ degrees of freedom: we have $11$ parameters per camera, $3$ for the coordinates of each point and we subtract $15$ to compensate for the projective ambiguity. The equation is thus solvable in cases where $2mn \geq 11m+3n-15$.

##### Two-Camera Case
From the inequality above, we know that we can solve for the projective structure of an image when we have at least $n=7$ points.

We can then begin by estimating the fundamental matrix $F$ between the two camera views. We then set the first camera matrix to $[I\mid 0]$ as our starting point: all other locations are estimated relative to the first camera. The matrix for the second camera is then given by $[A\mid t]$, where the translation $t$ is the epipole (such that $F^Tt=0$) and $A$ is given by the cross product $[t]_{\times}F$.


#### Incremental Structure from Motion
We can also take an incremental approach, in which we will build our our belief of the structure based on the motion we capture. This incremental process involves multiple sequential steps as outlined below:
##### Initialization 
The first thing we would like to do is to utilize our fundamental matrix $F$ (as explained above), to analyze the motion between images. With this, we can also initialize our structure by triangulating the initial points in these images. As we will discuss later, having a good initialization for this SfM process is incredibly important to achieve accurate results.
##### Iteration
With each additional view that we have, we will incrementally improve and extend our structure. We do this in two steps, **calibration** and **triangulation**. We define both: \
**Calibration** involves better understanding the world around the camera according to this new image by determining the projection matrix $P$ in this new angle. We can do this by looking at our known 3D points and analyzing the points that are present in this view. One could image that some angles are only able to capture certain known points, and we can use this as a way of understanding (and hence, calibrating) our sense of direction in the world (via the projection matrix $P$). As we can see in these two illustrations below, as our camera's move around in the world, the points they have access to may vary— we use this to our advantage in our to calibrate via the projection matrix and by better understand our existing knowledge about points in the next process, triangulation.
 ![](https://i.imgur.com/zYvrshc.png) ![](https://i.imgur.com/uqB8nmd.png)

Figure: Camera in motion capturing points (lecture 12, slide 36,37,38)

As mentioned, **triangulation** will involve improving our current understanding of the structure— we can use the information learned about the newly visible points, and re-optimize our understanding of the point we already know about. Perhaps, to better understand what triangulation is doing, let's consider a simple case— we have two cameras/images which have been fully calibrated (as described above, or in any other way), and we have one shared pixel (one in each image), triangulation will allow us to find and calculate the 3D location of any other point in the 3D world. To formalize, let's consider two arbitrary calibrated images ${image}^1$ and ${image}^2$, both of which can "see" a point $p$. For ${image}^1$, the point $p$ is at relative (but calibrated) location $(x_1, y_1)$ and for ${image}^2$, the point $p$ is at relative (but calibrated) location $(x_2, y_2)$. Triangulation will allow us to use our newly calibrated Projection Matrices to find a system of equations to find the 3D location of a third point visible in ${image}^1$ and ${image}^2$.

Our iteration process will involve both calibrating and triangulating for each new view.

##### Adjustment
Lastly, we can non-linearly (as opposed to the linear system of equations from triangulation) refine all of our cameras and the points in a joint fashion— this is done in a process known as **bundle adjustment**. The idea here is to minimize the total amount of error caused by re-projection. By minimizing a loss function that is a hueristic for re-projection error, we are able to refine both our structure ($X_j$) and our motion ($P_i$). \
We first need to introduce an indicator variable for whether a given point $j$ is visible from view $i$. We will let $w$ represent this variable s.t. $w_{i,j} = 1$ if and only if point $j$ is visible in view $i$ and $w_{i,j} = 0$ if it isn't.
With this, we have all of the notation needed to define our repredjction error/loss:
$$\text{Reprojection Error} = \sum_{i =1}^{m}\sum_{j =1}^{n}w_{i,j}d(\textbf{x}_{ij} - proj(P_iX_j))^2$$

We can use non-linear methods to find the optimal $X_j$ (structure) and our $P_i$ (motion) that *minimizes* this reprojection loss. For a deeper look into the Bundle Adjustment method, take a look at:

> Bill Triggs, Philip Mclauchlan, Richard Hartley, Andrew Fitzgibbon. Bundle Adjustment – A Modern
> Synthesis. International Workshop on Vision Algorithms, Sep 2000, Corfu, Greece. pp.298–372,
> ff10.1007/3-540-44480-7_21ff. ffinria-00548290f

##### Structure From Motion Caveats
Of course, with such a complex system, there are some details and nuances that should be addressed. With this iterative process, it is crucial that our inialization is viable— since each new view iterates on our understanding, it is important to have a good base for calibrating the rest of the process. It's also important for such a complex iterative procedure to be able to work around inconsistencies in our data/readings— for example, we should be sure to filter out any degenerate configurations or incorrect matches, as well as have systems in place to deal with and optimize for repititons and and symmetries. Lastly, this complex procedure involves many moving parts, and it's important to both be accurate (by ensuring that our error minimization algorithm is effective) and efficient (vectorizing our gradient descent, leveraging sparcity, etc).

## Incremental SfM in Practice and Summary
##### Incremental SfM

In practice, implenting SfM involves first choosing a pair of images with a large number of inliers and initializing the intrisic parameters such as focal length and principal point from the EXIF data. Extrinsic parameters R and t can then be estimated from the five-point algorithm, and triangulation can initialize the model points. After this, we simply iterate on any remaining images. For each image, once an image with many feature matches with images currently in the model is found, we use RANSAC on the feature matches and then triangulate new points that are visible from the camera but not yet in the model. Finally, we re-optimize the points by minimizing the error through bundle adjustment. An optional next step is using GPS to correlate relative transformations with the corresponding absolute positions.

##### Summary
SfM is a non-linear least squares optimization problem where we use reprojection error with respect to estimated 2D correspondences to optimize camera poses and 3D points. Local features are the only data we can leverage for 2D images so we use feature detection and feature matching to find 2D correspondences. To resolve projective ambiguity we require additional information about the world, which we can do by incrementally adding new cameras and points and periodically performing bundle adjustment to keep everything consistent.
