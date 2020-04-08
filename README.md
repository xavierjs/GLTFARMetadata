# GLTF AR Metadata proposal

Proposal to standardize specific GLTF metadata to be able to use the 3D model in an AR context.

This document is not a specification draft.


## Introduction

GLTF is an open 3D file format tailored for realtime rendering. It is standardized by the Khronos Group (see [khronos.org/gltf](https://www.khronos.org/gltf/) ). It can be used to encode any kind of 3D model.

In an augmented reality application, the 3D model needs to track specific objects in the real world. In some cases it can even be deformed to adapt its shape to a specific real world item.

This repository has been created to draft a proposal to standardize the information about mapping the real world information to the 3D model. This information could be embedded in the GLTF as metadata.


## Terms definition
* the **tracker**: is the software part detecting and tracking a specific real-world object from an image or a video stream. It is not concerned by this standard proposal. However, we detail its minimal output to better understand how it can be used to set the *AR tracking*,
* the **tracked object** is the real-world object tracked by the *tracker*. It can be a face, a can, a target image, ...
* the **3D model** is the 3D model rendered above the *tracked object*. It is stored in a standard 3D file format, typically GLTF.
* the **AR tracking** is the association between the *tracked object* and the *3D model*. The 3D model is set to the right pose to match the *tracked object*. The 3D model can be deformed in some cases.
* A **pose** is a rigid 3D transform composed by a 3D translation, a 3D rotation and a scale factor. It can be represented by a 4x4 matrix.

## Use-cases
Here are some use-cases addressed by this proposal.

### Virtual try-on
The 3D model is a 3D model of glasses.
It should be correctly displayed on the user's face.

### Face filters
The 3D model is a 3D model of a mask.
It should be correctly displayed on the user's face.
it should deform to keep specific matchings between the user's face and 3D model keypoints.

### Object replacement
The 3D model is a 3D animated mesh representing a snake crawling around a cylinder.
It should be displayed around a specific detected can.


## Problem modeling

### The tracker outputs
The *Tracker* provides:

* a 2D oriented bounding box,
* a 6DoF pose estimation, 
* a list of labelled keypoints

#### The 2D oriented bounding box
The tracker analyse the portion of the image into an oriented bounding box. It will be defined by:
* Relative 2D coordinated of its center, in `[-1.0, 1.0]`. The Y axis is oriented up, and the X axis from left to right (like OpenGL),
* Rotation angle in radians,
* Relative scale in `[0.0, 1.0]`, along both axis.

#### The pose estimation
The pose estimation should follow specific rules depending on the tracked object to avoid any ambiguity about the position offset, the orientation and the scaling:
* If the tracked object is cylindrical:
  * set position offset: the Y axis coincide with the cylinder vertical axis. The origin is at the base of the cylinder,
  * set orientation: the Z axis points back the main feature of the cylinder,
  * set scale: the radius of the cylinder along the Z axis is equal to `1.0`.

* If the tracked object is a face:
  * set position offset: the origin is the virtual point between the center of the pupils,
  * set orientation: the X axis is normal to the plane of symmetry of the face, the Y axis is facing up and the chin most proheminent point belongs to the Y axis,
  * set scale: the head maximum width (along X axis) is equal to `1.0`.

### The keypoints
The keypoints positions are provided in 2D, in the coordinates of the oriented bounding box. Each coordinate is in `[-1, 1]`.


### The 3D model

### Rigid transform
The right pose needs to be applied to the 3D model.
The mesh is associated with a label, called `ARTRACKTYPE` to specify which tracker could work with this object.

For example if I have a 3D can model:
* Where the Y axis coincide with the can vertical axis. The origin is at the base of the can,
* The Z axis points back the main logo displayed on the can,
* the 3D model is scaled so that its radius is equal to `1.0,`
* It has the `ARTRACKTYPE=CYLINDER` among its metada.

Then this model will be correctly placed by a tracker implementing the standard.
If this model has to be displayed on a random ground plane, using SLAM capabilities, a specific 3D transform will be automatically applied to this object since we know its `ARTRACKTYPE`. It won't be necessary to compute its scale, its orientation with bounding box estimation for example, to display it at a coherent pose.

In order to be able to associate an object with multiple `ARTRACKTYPE`, we don't apply the pose transform directly to the 3D mesh, but we associate the object with an array `ARTRACK`. Each value contains:
* The type of the `ARTRACK`, `ARTRACKTYPE`,
* A movement matrix, `ARTRACKMATRIX` (the pose matrix) to bring the object to the correct pose. If this value is not filled, we assume that the object is alreay in the right pose (i.e. the default value is the identity matrix).


### Deformable transform
A deformable 3D mesh needs first to be placed and oriented according to the pose estimation, like a rigid one.
To compute the deformation, the keypoint positions will be used. The deformation needs to keep the matching between specific keypoints and specific 3D mesh vertices.

A deformation is represented by:
* An array of keypoints. Each keypoint is represented by:
  * Its label, to match it with the keypoints of the tracker,
  * Its indice in the geometry position vertices array
* The influences array. It contains `<geometryVerticesCount> * <keypointsCount>` float values. For each geometry vertices it provides the `<keypointsCount>` influences values. Each influence value is between `0` (the keypoint has no influence on the vertice deformation movement) and `1`. For each vertice, the sum of the keypoint influences should be smaller than `1.0`.

The keypoints labelling will depend on the `ARTRACKTYPE`.
For example, a deformable object can be a semi-transparent face mesh with makup around the lips and around the eyes.
The mesh needs to be deformed in order to follow the lips movements.


### Multiplicity
Each GLTF file could contain multiple objects targeting different trackers.
So the AR tracking possibilities will be store in an array. Each item stores the ID of the 3D mesh and the AR tracking possibilities.


## Metadata proposal

The GLTF asset stores a deformable mask.
We want to be able to do virtual try-on with the mask, and also to be able to drop it on the table and view it in SLAM based AR.


```JavaScript
ARTRACKING: [
  { // the mask is used on the user face
    ID: '<id of the 3D model>',
    NAME: 'Try-on the mask', 
    TYPE: 'FACE', // = ARTRACKTYPE
    MATRIX: [1,0,0,0, 0,1,0,0, 0,0,1,0, 0,0,0,1],

    DEFORMEDID: '<id of the deformed geometry>',
    DEFORMEDKEYPOINTS:[
      {
        label: 'LEFTLIPSINT',
        ind: 522
      },
      {
        label: 'RIGHTLIPSINT',
        ind: 52
      },
      ...
    ],
    DEFORMEDINFLUENCES: [0.0,0.0,0.0,0.5,0.5,0.0....]
  },

  // the mask is dropped on the table
  // Here we don't use deformation
  {
    ID: '<id of the 3D model>',
    NAME: 'View the mask on the table',
    TYPE: 'GROUNDPLANE',
    MATRIX: [...]
  }
]

```