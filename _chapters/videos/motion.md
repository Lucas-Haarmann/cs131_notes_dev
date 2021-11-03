# Structure From Motion
## SfM Problem Definition
## Finding 2D Correspondences
## Solving for Cameras and 3D Points
#### Projective Structure from Motion
We will now consider multiple uncalibrated images of the same scene, from different cameras. Each image contains a certain number of fixed 3D points, which we will assume remain visible in all images. For the sake of this section, we will work with $m$ images with $n$ points. Therefore, we would like to compute a projection matrix, $P_i$, for each of the $m$ images. For a given 3D point $X_j$, the corresponding point on image $i$ is given by: $x_{ij}\sim P_iX_j$. We therefore need to estimate the values of the projection matrix $P_i$ and the 3D point $X_j$ using the 2D correspondences $x_{ij}$ alone.

Since we don't have any calibration information, we can only resolve the cameras and points up to a single projected transformation of of $X$, which we will call $Q$. In this case, we can replace $X$ with $QX$ and we can replace $P$ with $PQ^{-1}$. Then, returning to our original equation, we get $x\sim PQ^{-1}QX=PX$. We therefore cannot solve for $Q$. We therefore need extra information to find the exact values of $P$ and $X$.

We have $2mn$ constraints (since each $x_{ij}$ has two coordinate values). The matrix $P$ has $11m+3n-15$ degrees of freedom: we have $11$ parameters per camera, $3$ for the coordinates of each point and we subtract $15$ to compensate for the projective ambiguity. The equation is thus solvable in cases where $2mn \geq 11m+3n-15$.

###### Two-Camera Case
From the inequality above, we know that we can solve for the projective structure of an image when we have at least $n=7$ points.

We can then begin by estimating the fundamental matrix $F$ between the two camera views. We then set the first camera matrix to $[I|0]$ as our starting point: all other locations are estimated relative to the first camera. The matrix for the second camera is then given by $[A|t]$, where the translation $t$ is the epipole (such that $F^Tt=0$) and $A$ is given by the cross product $[t]_{\times}F$.


#### Incremental Structure from Motion
We can also take an incremental approach, in which we will build our our belief of the structure based on the motion we capture. This incremental process involves multiple sequential steps as outlined below:
###### Initialization 
The first thing we would like to do is to utilize our fundamental matrix $F$ (as explained above), to analyze the motion between images. With this, we can also initialize our structure by triangulating the initial points in these images. As we will disucss later, having a good initialization for this SfM process is incredibly important to achieve accurate results.
###### Iteration
With each additional view that we have, we will incrementally improve and extend our structure. We do this in two steps, **calibration** and **triangulation**. We define both: \
**Calibration** involves better understanding the world around the camera according to this new image by determining the projection matrix $P$ in this new angle. We can do this by looking at our known 3D points and analyzing the points that are present in this view. One could image that some angles are only able to capture certain known points, and we can use this as a way of understanding (and hence, calibrating) our sense of direction in the world (via the projection matrix $P$). As we can see in these two illustrations below, as our camera's move around in the world, the points they have access to may vary— we use this to our advantage in our to calibrate via the projection matrix and by better understand our existing knowledge about points in the next process, triangulation.
![](https://i.imgur.com/zYvrshc.png) ![](https://i.imgur.com/uqB8nmd.png)
As mentioned, **triangulation** will involve improving our current understanding of the structure— we can use the information learned about the newly visible points, and re-optimize our understanding of the point we already know about. Perhaps, to better understand what triangulation is doing, let's consider a simple case— we have two cameras/images which have been fully calibrated (as described above, or in any other way), and we have one shared pixel (one in each image), triangulation will allow us to find and calculate the 3D location of any other point in the world. To formalize, let's consider two arbitrary calibrated images ${image}^1$ and ${image}^2$, both of which can "see" a point $p$. For ${image}^1$, the point $p$ is at relative (but calibrated) location $(x_1, y_1)$ and for ${image}^2$, the point $p$ is at relative (but calibrated) location $(x_2, y_2)$. Triangulation will allow us to use our newly calibrated Projection Matrices to find a system of equations to find the 3D location of a third point visible in ${image}^1$ and ${image}^2$.

Our iteration process will involve both calibrating and triangulating for each new view.

###### Adjustment
Lastly, we can non-linearly (as opposed to the linear system of equations from triangulation) refine all of our cameras and the points in a joint fashion— this is done in a process known as **bundle adjustment**. This process will involve non-linear optimization techniques that are outside the scope of these notes (such as gradient descent and other convex optimization strategies); however the idea is to minimize the total amount of error caused by re-projection. By minimizing a loss function that is a hueristic for re-projection error, we are able to refine both our structure ($X_j$) and our motion ($P_i$). \
We first need to introduce an indicator variable for whether a given point $j$ is visible from view $i$. We will let $w$ represent this variable s.t. $w_{i,j} = 1$ if and only if point $j$ is visible in view $i$ and $w_{i,j} = 0$ if it isn't.
With this, we have all of the notation needed to define our repredjction error/loss:
$$\text{Reprojection Error} = \sum_{i =1}^{m}\sum_{j =1}^{n}w_{i,j}d(\textbf{x}_{ij} - proj(P_iX_j))^2$$
We can use non-linear methods to find the optimal $X_j$ (structure) and our $P_i$ (motion) that *minimizes* this reprojection loss. For a deeper look into the Bundle Adjustment method, take a look at:

> Bill Triggs, Philip Mclauchlan, Richard Hartley, Andrew Fitzgibbon. Bundle Adjustment – A Modern
> Synthesis. International Workshop on Vision Algorithms, Sep 2000, Corfu, Greece. pp.298–372,
> ff10.1007/3-540-44480-7_21ff. ffinria-00548290f

###### Structure From Motion Caviats
Of course, with such a complex system, there are some details and nuances that should be addressed. With this iterative process, it is crucial that our inialization is viable— since each new view iterates on our understanding, it is important to have a good base for calibrating the rest of the process. It's also important for such a complex iterative procedure to be able to work around inconsistencies in our data/readings— for example, we should be sure to filter out any degenerate configurations or incorrect matches, as well as have systems in place to deal with and optimize for repititons and and symmetries. Lastly, this complex procedure involves many moving parts, and it's important to both be accurate (by insuring that our error minimization algorithm is effective) and efficient (vectorizing our gradient descent, leveraging sparcity, etc).
## SfM in Practice and Summary
When implementing SfM in practice, we must begin by choosing images with a large number of inliers. We can then initialize intrinsic parameters like focal length using EXIF data.
