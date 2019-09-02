# BLOSPRAY - OSPRay as a Blender render engine

| Blender scene, with proxy object for the volume | Final render by OSPRay through BLOSPRAY |
|-------|--------|
| ![](docs/blender.small.png) | ![](docs/blospray.small.png) | 

BLOSPRAY aims to provide high-quality rendering of scientific
data in [Blender](https://www.blender.org) 2.8x, with a specific focus on volumetric data and use in
an HPC (High-Performance Computing) context. To accomplish this BLOSPRAY integrates 
the [OSPRay](http://www.ospray.org/) ray tracing engine from Intel in Blender 
as an external renderer.

Currently, this software is somewhere between alpha and beta quality,
as it can already be used to produce useful visualization, but lots of
things are still missing and not all errors are gracefully handled.

In case you give it a try and/or have remarks or questions, let us know
at visualization@surfsara.nl!

## Goals 

- Flexible and high-quality rendering of large scientific datasets using OSPRay 
- Provide a way to add user-specific types of data as
  scene elements *without having to import that data into Blender*
- Make OSPRay features available to Blender users

Note that BLOSPRAY does not aim to compete with Cycles (the builtin renderer of Blender),
as it has a different focus. Cycles provides production rendering of animations
and stills, focusing (more or less) on artistic workloads. Instead, BLOSPRAY focuses 
on rendering scenes containing (large) scientific datasets, where 
efficient handling and rendering of the data is usually a more important goal 
and challenge than production of photo-realistic imagery.

## Features

### Supported scene elements and features

BLOSPRAY is still in its early stages of development, but the following 
basic functionality and integration with Blender features is already available:

* Supports OSPRay's SciVis and Path Tracer renderers
* Export and rendering of polygonal geometry (i.e. Blender meshes)
* Object transformations and parenting
* Instancing 
* Vertex colors on Blender meshes
* Point, sun, spot and area lights
* Perspective and orthographic cameras, plus OSPRay's panoramic camera (which is similar to Cycles` equirectangular camera, but without any parameters to tweak)
* Depth of field
* Border render (i.e. render only part of an image)
* Rudimentary support for volume, geometry and scene plugins

Major features that are currently missing:

* Interactive preview render (which is ironic, given that real-time interactive rendering
  is one of OSPRay's main features)
* Material editing
* Volume transfer function editing
* Motion blur (which is not supported by OSPRay itself)
* Parallel rendering mode through MPI

Other features that might be of interest in the HPC context that are missing:

* Texturing
* HDRI lighting

Integration within the Blender UI, mostly panels for editing properties and such, 
is also very rudimentary. Some properties are currently only settable using Blender
[custom properties](https://docs.blender.org/manual/en/latest/data_system/custom_properties.html) on objects and meshes.

### Render server

BLOSPRAY consists of two parts:

1. A Blender addon (directory `render_ospray`) that implements the Blender external render engine. It handles scene export to OSPRay, retrieving the rendered image from OSPRay and other things.
2. A standalone render server (`bin/ospray_render_server` after building) that receives the scene from Blender, calls OSPRay routines to do the actual rendering and sends back the image result.

The original reason for this two-part setup is that there currently is no 
Python API for OSPRay, so direct integration in Blender is not straightforward. 
Neither is there a command-line OSPRay utility that takes as input a scene 
description in some format and outputs a rendered image. 

Plus, the client-server setup also has some advantages:

- The separate render server can be run on a remote system, for example an HPC system 
  that holds a large scientific dataset to be rendered. This offloads most of the 
  compute-intensive rendering workload and memory/storage requirements of the data to be rendered 
  away from the system running Blender.

- It makes it feasible to use OSPRay's [Parallel Rendering with MPI](http://www.ospray.org/documentation.html#parallel-rendering-with-mpi) 
  mode, by providing a variant of the render server as an MPI program. Again,
  this parallel version of the server can be run remotely on an HPC system.
  Note that this mode is currently not implemented yet.

- The network protocol is currently not strongly tied to Blender, so the render server can be used in other contexts as well.

- BLOSPRAY development becomes slightly easier as both Blender and the render 
  server can be independently restarted in case of crashes or bugs.

Of course, this client-server setup does introduce some overhead, in terms of 
network latency and data (de)serialization. But in practice this overhead is 
small compared to actual render times. In future, caching of data on the server 
between renders can help in reducing the overhead even further.

Note that the render server currently doesn't support loading multiple different 
Blender scenes or serving different users at the same time. And there is also 
a manual action required to start/stop the render server.

### Plugins

There is a rudimentary plugin system that can be used to set up
custom scene elements that are represented by a proxy object in Blender,
but whose full representation and rendering is stored on the server side
in OSPRay. The original use case for plugins (and even BLOSPRAY itself) was
to make it easy to use OSPRay's high-quality volume rendering of scientific data in a 
Blender scene. Blender's own volume rendering support is geared towards the built-in 
smoke and fire simulations and isn't really a good fit for scientific datasets.
Plus it is very hard to get generic volume data (e.g. with multiple components
per cell) into Blender.

Apart from volumes, plugins can be used for other types of scene content as well, 
like polygonal geometry or OSPRay's builtin geometry types like spheres and
streamlines. Basically any scene element that can be created with the OSPRay API
can potentially be used during rendering through plugins.

The plugin system is especially useful when working with large scientific datasets 
for which it is infeasible or unattractive to load into Blender. Instead, one 
can use a proxy object, such as a cube mesh to represent a rectangular volume, 
in a Blender scene and attach a plugin to it. During rendering BLOSPRAY will call the 
plugin on the server to load the actual scene data associated with the proxy. 
In this way Blender scene creation, including camera animation and lighting, can be 
done in the usual way as the proxy object shows the bounding box of the data 
and can even be transformed and animated.

A nice benefit of writing a plugin in C++ is that it allows one to load a large 
dataset directly into memory, without having to go through the
Blender Python API in order to accomplish the same. The latter is usually
less efficient.

Note that BLOSPRAY plugins are different from OSPRay's own [Extensions](http://www.ospray.org/documentation.html#loading-ospray-extensions-at-runtime), 
that are also loadable at run-time. The latter are meant for extending OSPRay itself
with, for example, a new geometry type. BLOSPRAY plugins serve to extend *Blender*
with new types of scene elements that are then rendered in OSPRay.


## Known limitations and bugs

This project is developed for Blender 2.8x, the latest stable release.
Blender 2.7x is not supported.

As BLOSPRAY is in early development some things are currently suboptimal or missing:

* The addon provides some UI panels to set OSPRay specific settings, but in other cases we use Blender's [custom properties](https://docs.blender.org/manual/en/dev/data_system/custom_properties.html)
  to pass information to OSPRay. These can even be animated, with certain limitations, but are not a long-term solution. Note also that some builtin UI panels are disabled when the render engine
  is set to OSPRay as those panels can't directly be used with OSPRay (e.g. they contain Cycles-specific settings).
  
* In many cases only a subset of OSPRay parameters can be set from Blender, either using UI elements or using custom properties

* Scene management on the render server is currently non-existent. I.e. memory usage increases after each render.  

* Caching of scene data on the server, especially for large data loaded by plugins, is planned to be implemented, but isn't there yet. I.e. currently all scene data is re-sent when rendering a new image.

* Only a single (hard-coded) transfer function for volume rendering and slice planes are supported. Similar for materials on geometry.

* Only final (F12 key) renders are supported. Preview rendering is technically feasible, but is not implemented yet.

* Error handling isn't very good yet, causing a lockup in the Blender script in case the BLOSPRAY server does something wrong (like crash ;-))

* BLOSPRAY is only being developed on Linux at the moment, on other platforms it might only work after some tweaks

* Command-line (batch) rendering isn't supported in a nice way yet, as the lifetime of the BLOSPRAY server needs to be managed manually.

* Volumes can only use point-based values, not cell-based values (XXX this is a limitation of the volume_raw plugin)

* All Blender meshes are converted to triangle meshes before being passed to BLOSPRAY, even though OSPRay also supports quad meshes


### Limitations specific to OSPRay 

Some of these we can work around, some of these we can't:

* OSPRay's SciVis and Path Tracer renderer do not have the same set of features:

    - Only the Scivis renderer supports volume rendering, but that should change with OSPRay 2.0
    - Only the Path Tracer supports advanced materials, like the principled material

* In OSPRay structured grid volumes currently cannot be transformed with an 
  arbitrary affine transformation (see [this issue](https://github.com/ospray/ospray/issues/159)
  and [this issue](https://github.com/ospray/ospray/issues/48)). 
  We work around this limitation in two ways in the `volume_raw` plugin:
  
  - Custom properties `grid_spacing` and `grid_origin` can be set to influence the
    scaling and placement of a structured volume (these get passed to the `gridOrigin`
    and `gridSpacing` values of an `OSPVolume`). See **tests/grid.blend** for an example.
    
  - By setting a custom property `make_unstructured` to `1` on the volume object 
    the structured grid is converted to an unstructured one, whose vertices *can* be transformed. 
    Of course, this increases memory usage and decreases render performance, but makes 
    it possible to handle volumes in the same general way as other 
    Blender scene objects. See **tests/test.blend** for an example.
    
* Volumes are limited in their size, due to the relevant ISPC-based
  code being built in 32-bit mode. See [this issue](https://github.com/ospray/ospray/issues/239).
  
* Unstructured volumes can only contain float values (i.e. not integers).
  See [here](https://github.com/ospray/ospray/issues/289)
  
* The OSPRay orthographic camera does not support depth-of-field
  
* Blender supports multiple colors per vertex (basically one per vertex per face loop),
  while OSPRay only supports a single value per vertex (XXX need to double-check this). 
  During export the vertex colors are reduced to a single color per vertex
  
* OSPRay does not support motion blur

* All rendering is done on the CPU, because, well ... it's OSPRay ;-)

## Dependencies

For building:

* [OSPRay 2.x](http://www.ospray.org/), which is currently not released yet, so 
  use the "devel" branch
* [GLM](https://glm.g-truc.net/0.9.9/index.html)
* [OpenImageIO](https://sites.google.com/site/openimageio/home)
* [Google protobuf (C/C++ libraries)](https://developers.google.com/protocol-buffers/)

* The code uses the [JSON for Modern C++](https://github.com/nlohmann/json) library,
  which is included in the sources

For running with Blender:

* Numpy
* Google protobuf (Python modules)

## Building

BLOSPRAY uses CMake for building in the usual way. There's currently 
only a very small set of BLOSPRAY specific switches that can be tweaked.

The current way to build BLOSPRAY is to make a build directory within
the source directory and build there. A `make install` will then place
all files (server, plugins, test files) in a `bin` subdirectory of the sources:

```
$ cd blospray
$ mkdir build
$ cd build
$ cmake -GNinja ..
# Configure, Generate
$ ninja install
$ cd ../bin
$ ls
geometry_plane.so  ospray_render_server  t_json  volume_raw.so
```

## Installation

Part of BLOSPRAY consists of the Blender add-on, but this is not being distributed separately
as the focus currently is getting to a releasable state in terms of features.
Currently, the way to install it is to clone this repository and then
make a symlink to the `render_ospray` directory in the Blender addon directory:

```
$ cd <blender-2.8>/2.80/scripts/addons
$ ln -sf <blospray-repo>/render_ospray render_ospray
```

Within Blender enable the `Render: OSPRay` addon. You should now have a new `OSPRay`
entry in the `Render Engine` dropdown on the `Render` properties tab.

To make Blender find the necessary protobuf packages add symlinks to
`google` and `six.py` in Blender's python library dir:

  ```
  $ cd <blender-2.8>/2.80/python/lib/python3.7/site-packages
  $ ln -sf /usr/lib/python3.7/site-packages/six.py six.py
  $ ln -sf /usr/lib/python3.7/site-packages/google google
  ```

## Examples

The directory `tests/` contains some examples and tests. Some of these
need extra data files that are currently not included (mostly due to their size).

## License

BLOSPRAY is licensed under the Apache License, version 2.0 (the same
license as OSPRay). See LICENSE-2.0.txt for more information.

Some files, mostly separate test files, are not part of BLOSPRAY
itself. These files have license information included in their text,
where appropriate.

BLOSPRAY uses the [JSON for Modern C++](https://github.com/nlohmann/json) 
header-only library, which is included in this source distribution
as `core/json.hpp`. This file is not part of BLOSPRAY and has a different 
license, see its contents.
