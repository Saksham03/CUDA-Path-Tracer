CUDA Path Tracer
================

**University of Pennsylvania, CIS 565: GPU Programming and Architecture, Project 3**

* Saksham Nagpal  
  * [LinkedIn](https://www.linkedin.com/in/nagpalsaksham/)
* Tested on: Windows 11 Home, AMD Ryzen 7 6800H Radeon @ 3.2GHz 16GB, NVIDIA GeForce RTX 3050 Ti Laptop GPU 4096MB

## Summary
This project is a learning attempt turned into a definitive milestone in my journey of learning CUDA. The aim of making a path tracer using CUDA and C++ is to replicate the behaviour of the graphics pipeline while manually being in-charge of the GPU-side kernel invocations. Using CUDA, I map each of the steps in the graphics pipeline (i.e. vertex shader, rasterization, fragment shader, etc.) into equivalent kernel invocations, thus solidfying both experience with CUDA as well as knowledge of the graphics pipeline. This project also turned out to be a good way of keeping me true to my understanding of the core graphics concepts such as coordinate transformations and barycentric interpolation. By implementing a combination of visually pleasing and computationally accelerating features, I was able to generate some fun renders such as these:

## Representative Outcomes  
_247,963 triangles, 5000 iterations, max 10 bounces per ray_
 ![](images/finalrender1.png)  

_Stanford Dragon - 871,306 triangles, 5000 iterations, max 8 bounces per ray_
![](images/cornell.2023-10-11_16-31-13z.5000samp.png) 

## Features Implemented

1. BSDFs - Diffuse, Perfectly Specular, Perfectly Reflective, Imperfect Specular/Diffuse, Refractive
2. Acceleration Structure - Bounding Volume Heirarchy (BVH)
3. Stochastic Sampled Antialiasing
4. Physically-Based Depth of Field
5. Support for loading glTF meshes
6. Texture Mapping for glTF meshes
7. Stream Compaction for ray path termination
8. First bounce caching
9. Material Sorting
10. Reinhard operator & Gamma correction

## Path Tracer

A path tracer, in summary, is an effort to estimate the **Light Transport Equation (LTE)** for a given scene. The LTE is a methematical representation of how light bounces around in a scene, and how its interactions along the way with various kinds of materials give us varied visual results.

The Light Transport Equation
--------------
#### L<sub>o</sub>(p, &#969;<sub>o</sub>) = L<sub>e</sub>(p, &#969;<sub>o</sub>) + &#8747;<sub><sub>S</sub></sub> f(p, &#969;<sub>o</sub>, &#969;<sub>i</sub>) L<sub>i</sub>(p, &#969;<sub>i</sub>) V(p', p) |dot(&#969;<sub>i</sub>, N)| _d_&#969;<sub>i</sub>

The [PBRT](https://pbr-book.org/3ed-2018/Light_Transport_I_Surface_Reflection/The_Light_Transport_Equation#) book is an exceptional resource to understand the light transport equation, and I constantly referred it throughout the course of this project.

# Performance Analysis

Here I will document the attempts I made to boost the performance of my pathtracer - my hypothesis on why I thought the change could result in a potential performance improvement, and my actual findings along with supporting metrics.

## 1. First Bounce Caching

As a rudimentary optimization, one could try caching the first ray intersections with the scene geometry. The first set of rays generated from the camera per pixel is deterministic, and hence the first intersections would always be the same. Therefore, one can expect a small performance boost from this optimization. Note that this optimization would be invalidated as soon as **Multi-Sampled Anti Aliasing (MSAA)** is implemented, since in that case the first set of generated rays would no longer remain deterministic.

### Metrics:

<img src="images/firstbouncecache.png" width=500> 

As expected, caching the first bounce indeed results in a small performance gain. However, this gain becomes less and less significant with increasing trace depth of rays, potentially offset due to more computationally expensive intersections.

## 2. Stream Compaction for Ray Path Termination

For each iteartion, we bounce a ray around the scene until it terminates, i.e., either hits a light source or reaches some terminating condition (such as max depth reached, no scene geometry hit, etc.). After each iteartion of scene intersection, if we were to compact the ray paths, then potential performance gains could result due to the following reasons:
1. **Reducing the number of kernel threads:** Instead of calling the scene intersection kernel always with number of threads equal to the number of pixels in our scene, we could invoke the kernel only with as mnay threads as there are unterminated rays. This should result in less copy-over to and from the device to host, and thus giving us a better memory bandwidth.
2. **Coalescing Memory:** In our array of rays, each iteration would cause rays at various random indices to get terminated. If we were to compact away the terminated rays and partition our non-terminated rays into contiguous memory, then that should enable warps to retire and be available to the scheduler for more work. Coherent memory should also hasten access times, thereby potentially adding to the performance gain.

### Metrics:

<img src="images/streamcompactionopen.png" width=500> 

* **Open Scene:** For the open scene, an actual benefit is seen at higher depth values. This makes sense, since in an open scene there is a high probability for a ray to hit no scene geomtery, and therefore not reach the light source, In such a scenario, removing all the terminated rays at each iteartion would give us a much smaller subset of rays to work with in the next iteration. Since Stream Compaction's performance for our case is memory bound, it makes sense for the offset to occur at higher depths wherein termination of paths actually starts making up for the cost of stream compaction. For our case, looking at the graph we see that stream compaction starts becoming more performant than naive path termination _scene depth = 6_ onwards.


<img src="images/streamcompactionclosed.png" width=500>

* **Closed Scene:** For a closed scene, there is a much higher probability for rays to reach the light source by bouncing around the scene numerous times. This is corroborated by the grpah, wherein we do not see the number of unterminated rays decreasing with depth count unlike the open scene case. Hence, stream compaction happens to be just an unnecessary overhead in this case, and is never able to offset the naive path termination's performance.


## 3. Material Sorting

Another potential candidate for a performance boost is sorting the rays and intersections based on the material indices. Every path segment in a buffer is using one large shading kernel for all BSDF evaluations. Such differences in materials within the kernel will take different amounts of time to complete, making our kernel highly divergent. Therefore, sorting the rays to make paths interacting with the same material contiguous in memory before shading could potentially give us better performance by reducing **Warp Divergence**.

### Metrics:

<img src="images/materialsort.png" width=500> 

We see that for our case, at such a small level, material sorting is not really giving us a benefit. In fact, it is actually incurring an extreme overhead. The reason for this could be the small scale and the simplpicity of our scenes, where  there is no significant gain that could offset the hit we take from sorting each ray and intersection based on material.

## 4. Acceleration Structure - Bounding Volume Heirarchy (BVH)

BVHs are widely used acceleration structures, and I had always wanted to implement one. There are very many versions out there of BVHs, both in terms of implementation as well as traversal. I implemented a very basic version referring to [PBRT](https://pbr-book.org/3ed-2018/Primitives_and_Intersection_Acceleration/Bounding_Volume_Hierarchies).

**Construction**
1. Assign each geometry(primitive/triangle) its own **Axis-Aligned Bounding Box (AABB)**.
2. Start with the root node, which is assigned an AABB that is a union of the AABBs of all the primitives in the scene.
3. At every iteration, determine the axis which has the longest spanning range for all the AABBs' centroids. Split the geometry on that axis, and the two halves are made the two child nodes of the current node.
4. The above process is repeated until nodes containing single primitives are encounterd, which are then made then leaf nodes of the tree.
5. Once this tree is constructed, a **linearized version of the tree** is constructed so that it can be stored in an array and sent over to the GPU for traversal.

**Traversal**
1. On the GPU, a stack-based traversal is sued to determine the ray's intersection with the scene.
2. The root node is first pushed onto the stack.
3. When the traversal begins, the stack is popped, and the popped element is checked if it is a leaf node, and if it is, then scene intersection is calculated. If this turns out to be the closest hit yet, we replace the current closest hit parameters with the results from this intersection.
4. If the popped node is not a leaf, then both its children's AABBs are checked for intersection against the ray. If a child's AABB is intersected by the ray, it is pushed onto the stack.

### Metrics:

To test the performance of the BVH, I start with a simple scene containg 682 triangles, and then I keep adding instances of the same mesh at every iteration of the test.

| ![](images/bvh_1.png) | ![](images/bvh_2.png) | ![](images/bvh_3.png) | ![](images/bvh_4.png) | ![](images/bvh_6.png) |
|:--:|:--:|:--:|:--:|:--:|
| *1st Test: 682 triangles* | *2nd Test: 1364 triangles* |  *3rd Test: 2046 triangles* |  *4th Test: 2728 triangles* |  *5th Test: 4092 triangles* |

The following graph clearly demonstartes the significant FPS boost we receive from using a BVH. Not only do we get a performance gain, but it also offsets the naive performance by more than 2X.

<img src="images/bvh_perf2.png" width=500> 

To compare the FPS trends of the with- and without-BVH cases, I used a logarithmic scale for a better comparison of the trend:

<img src="images/bvh_perf.png" width=500> 

The log scale clearly shows how well the FPS fall-off is counter-acted by the Bounding Volume Heirarchy. In fact, many of the renders in this project would not even have been possible without implementing an acceleration structure.
