---
layout: post
title: Lumberyard Gems
tags:
- lumberyard
- gamedev
canonical_url: 
series: Amazon Lumberyard
---

https://docs.aws.amazon.com/lumberyard/latest/userguide/debugging-intro.html

![](/assets/lmbr_savelevelstats.png)

Here's __Static Geometry__:  
![](/assets/lmbr_savelevelstats_static_geom.png)


| Column | Description
|-|-|
| Filename | 
| Refs | 
| Mesh Size | (MB) includes vertices, indices, and other vertex streams such as tangents, normals, etc.
| Texture Size
| Draw Calls
| LODs
| Sub Meshes
| Vertices
| Tris
| Physics Tris
| Physics Size
| LODs Tris | Number of triangles in each LOD mesh
| Mesh Size Loaded | (MB) For all LODs, includes vertex and index buffers as well as duplicated data kept in system memory, but not engine overhead from `CRenderMesh`, `CRenderChunk`, etc.
| Split LODs

[indexed geometry](http://www.opengl-tutorial.org/intermediate-tutorials/tutorial-9-vbo-indexing/)

Worst case `#tris = #verts / 3`, i.e. each triangle is made of 3 unique vertices- index buffer contains no duplicated values.