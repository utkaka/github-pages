---
title: "Voxel rendering in Unity (Part 1)"
date: 2023-09-28
---

![Voxelized spider!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-09-28-voxel-rendering-in-unity-part-1/1-Landing.gif)
## Intro

I never thought that the challenges associated with a voxel game could captivate me so much. From an aesthetic point of view, voxels are clear to me, but not close. However, one day we decided to make a small prototype of a mobile game with destructible voxel monsters.

It was clear from the beginning that if we were to go down the path of treating voxels as individual objects with possible physics, we would quickly hit the performance limitations of mobile devices. So, it is necessary to explore all possible optimization methods to provide our players with as many voxels as possible and the joy of destroying them.

And I want to admit that solving this problem turned out to be much more interesting than I initially thought. Because, finally, I found a use for many algorithms that are not usually needed that often during game development.

In this series of articles, implementation on the **Unity** engine with its unique features will be discussed. However, I hope it will also be helpful in case of using other game engines.

And we will begin with content creation. After all, in order to render voxels, we first need to obtain them from somewhere.

## Creating voxel models

### Manual

The path of a perfectionist. We can begin by creating models, for example, in [MagicaVoxel](https://ephtracy.github.io/) and then bake them into a mesh or write our own converter. Unfortunately, a mesh will will not be suitable because we need not only the aesthetics of voxels but also precise information about each voxel so that we can destroy them later. We can develop some sort of intermediate converter, although we would prefer not to. However, the most significant drawback of this method is the substantial amount of labour required for creating each model.

- (+) Perfect voxel models.
- (-) High labor efforts.
- (-) The need to convert data from a voxel editor into the format we require.

### Automatic

Automatic generation of voxel models from regular ones appears much more attractive. This way, we can create content at maximum speed using regular models. However, we will need to write our own voxel generator, and on larger voxel sizes, our models may not look as appealing.

- (+) Rapid content creation.
- (-) Possibly lower-quality models.
- (-) The need to convert a regular mesh to voxels.

The choice is actually quite complex and entirely depends on the studio's sense of aesthetics and available resources. We opted for the automatic generation method because our goal is to create a game, not focus solely on voxel art.

## Voxelization methods

We will only consider the case where the mesh consists of triangles, as this will be the most common scenario. However, it's worth noting that [other variants](https://docs.unity3d.com/ScriptReference/MeshTopology.html) are possible. Therefore, our task boils down to [rasterizing triangles](http://www.sunshine2k.de/coding/java/TriangleRasterization/TriangleRasterization.html), but with the addition of the third dimension, which makes it slightly more complex.

Several algorithms will be discussed below. **However, not all possible options were explored in great detail**. Since our primary focus is game development, I didn't want to invest too much time in the preparatory stage. I simply sketched out different variants quickly and attempted to assess their performance and accuracy. There might be mistakes in some of the implementations, so my final choice may not be the most optimal.

### Bresenham's line algorithm

[This algorithm](https://en.wikipedia.org/wiki/Bresenham%27s_line_algorithm) came to my mind first because it's a fundamental rasterization algorithm, primarily because of its simplicity and the efficiency of integer calculations.

To begin with, let's add the [third dimension](https://www.geeksforgeeks.org/bresenhams-algorithm-for-3-d-line-drawing/) to the algorithm.

The subsequent steps are approximately as follows:

1. Obtain the voxels of each edge using the algorithm.

2. Sort the edges by their lengths.

![Bresenham Line Rasterisation!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-09-28-voxel-rendering-in-unity-part-1/2-Bresenham-1.png)

3. Use the algorithm to connect each voxel from edge A with every voxel from edge B.

![Bresenham Triangle Filling!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-09-28-voxel-rendering-in-unity-part-1/3-Bresenham-2.png)

It's a rather rough implementation, but I wanted to see how it would look. It doesn't look as precise as the other algorithms. The issues are not only with my rough triangle filling but also with the edges themselves. This is primarily due to the integer nature of the algorithm. But it was worth a try.

### Sampling

In this method, we use a fixed step to traverse all the points of the triangle and determine which voxel they belong to.

![Sampling Triangle Filling!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-09-28-voxel-rendering-in-unity-part-1/4-Sampling.png)

1. In the main loop, we move from vertex V0 to vertex V1 with a fixed step.

2. In the inner loop, we move from the previously obtained point in the direction of V1V2.

3. Calculate the voxel to which the point belongs.

The major drawback of this method is its reliance on the choice of the sampling step. A large step results in low accuracy, while a small step leads to an excessive number of calculations.

### Separating Axis Theorem

Here we will utilize the [Separating Axis Theorem](https://gdbooks.gitbooks.io/3dcollisions/content/Chapter4/aabb-triangle.html). Its significant advantage over the previous method is the precise result, which is independent of any external parameters.

1. Calculate the triangle's boundaries in voxel space.

2. Iterate through each voxel within the boundaries.

3. Using **SAT**, check whether the voxel intersects with the triangle.

![SAT Triangle Filling!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-09-28-voxel-rendering-in-unity-part-1/5-SAT.png)

### Preformance

Even though I had already made my choice during the experimentation phase, I also roughly measured the relative performance of each method. These measurements were conducted on a mesh with 1792 triangles in a grid of 148x82x166 voxels.

| Method                                   | Execution time      |
| ---------------------------------------- | ------------------- |
| Bresenham's line algorithm               | 200 ms              |
| Sampling (step = voxelSize)              | 70 ms               |
| Sampling (step =  0.25 * voxelSize)      | 350 ms              |
| Sampling (step = 0.125 * voxelSize)      | 1180 ms             |
| Sampling (step = 0.0625 * voxelSize)     | 4450 ms             |
| SAT                                      | 990 ms              |

## Voxelization

Our primary focus is voxelizing individual meshes, but we will also implement the ability to process any number of meshes within a unified voxel space. This will allow us to create complex voxel models from primitives.

For example, create a hybrid of a spider and a sphere.

![Spider With Sphere!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-09-28-voxel-rendering-in-unity-part-1/6-Spider-With-Sphere.png)

Let's formalize the final algorithm for mesh voxelization:

1. Determine the boundaries of the voxelization space and the number of voxels within these boundaries, taking into account the positions of the meshes.

2. Take each triangle from every mesh.

3. Calculate the triangle's boundaries in voxel space, considering the position of the mesh to which this triangle belongs.

4. Iterate through all the voxels within these boundaries.

5. Check if there is either no previously recorded triangle in this voxel or if the center of the current triangle is closer to the voxel's center than the previously recorded triangle's center. Otherwise, skip this voxel.

6. Check for intersection between the voxel and the triangle. If there is no intersection, skip this voxel.

7. Store information about the current triangle in the voxel.

8. After processing all the triangles, calculate UV coordinates for each voxel in its respective mesh.

9. Get the voxel's color from the texture of its respective mesh.

### Advanced Mesh API

Since we need to perform a substantial volume of computations on a large dataset, it makes sense to utilize the Unity Job System. It's convenient to work with the [Advanced Mesh API](https://docs.unity.cn/ScriptReference/Mesh.MeshData.html) for this purpose. Its primary benefit is that it provides us with the MeshData struct, which we can pass to a job for processing. You can find a little more information about this [here](https://catlikecoding.com/unity/tutorials/procedural-meshes/creating-a-mesh/#3).

As we have chosen to process multiple meshes simultaneously, [MeshDataArray](https://docs.unity.cn/ScriptReference/Mesh.MeshDataArray.html) will be useful.

With minor differences, this API can be used in the same way as the standard mesh API. For each mesh, we can use methods such as [GetIndices](https://docs.unity.cn/ScriptReference/Mesh.MeshData.GetIndices.html), [GetVertices](https://docs.unity.cn/ScriptReference/Mesh.MeshData.GetVertices.html), etc. However, this approach has two drawbacks:

- It requires additional memory allocation for NativeArrays that will be populated with the necessary data.

- Since NativeArrays cannot be created inside a job, we must process each mesh in separate jobs, and do so sequentially because each of these jobs needs to write to the shared voxels array.

Hence, we can utilize methods like [GetIndexData](https://docs.unity.cn/ScriptReference/Mesh.MeshData.GetIndexData.html) and [GetVertexData](https://docs.unity.cn/ScriptReference/Mesh.MeshData.GetVertexData.html) which allow us to read mesh data directly without memory allocations and data copying. However, to do this, we need to know the exact layout of the mesh. Since we are creating a converter for arbitrary meshes (while keeping in mind the triangle topology), we must be prepared for any layout.

We can view the specific mesh's structure in the inspector of the editor. For example, here is what my test spider looks like:

![Spider Inspector!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-09-28-voxel-rendering-in-unity-part-1/7-Spider-Inspector.png)

The API also provides appropriate methods to retrieve this information.

Unfortunately, JobSystem has certain inherent limitations. For instance, it's impossible to use interfaces because we might attempt to pass a managed object into them. Therefore, it's challenging to avoid code duplication. However, we will attempt to minimize it by using generics.

First, we need to read the indices of the mesh's vertices. This is relatively straightforward because there are only two possible formats: 16-bit and 32-bit. You can determine the format using the [indexFormat](https://docs.unity.cn/ScriptReference/Mesh.MeshData-indexFormat.html) property.

```c#
if (meshData.indexFormat == IndexFormat.UInt16) {
	var indexData = meshData.GetIndexData<short>();
	//Process submeshes
} else {
	var indexData = meshData.GetIndexData<int>();
	//Process submeshes
}
```

A mesh itself doesn't consist of triangles; it consists of submeshes, which can (and in our case, should) contain triangles. To do this, we use the [GetSubMesh](https://docs.unity.cn/ScriptReference/Mesh.MeshData.GetSubMesh.html) method, which returns [SubMeshDescriptor](https://docs.unity.cn/ScriptReference/Rendering.SubMeshDescriptor.html), and iterate through all the submeshes.

```c#
for (var j = 0; j < meshData.subMeshCount; j++) {  
    var subMeshDescriptor = meshData.GetSubMesh(j);  
    for (var s = subMeshDescriptor.indexStart; s < subMeshDescriptor.indexStart + subMeshDescriptor.indexCount; s += 3) {  
       var i0 = indexData[s];  
       var i1 = indexData[s + 1];  
       var i2 = indexData[s + 2];  
       //Process triangle with vertices i0, i1 and i2 
    }  
}
```

My test meshes had only one submesh each, so the correctness of this code should be verified with complex meshes. The official documentation does not provide a clear description of all the indices that can be retrieved from SubMeshDescriptor.

Now that we have the vertex indices for each triangle, we need to read their positions to implement the SAT algorithm.

In general, vertex attributes can vary in [data streams](https://docs.unity.cn/ScriptReference/Mesh.MeshData.GetVertexAttributeStream.html), have different [dimensions](https://docs.unity.cn/ScriptReference/Mesh.MeshData.GetVertexAttributeDimension.html), and use diverse [formats](https://docs.unity.cn/ScriptReference/Mesh.MeshData.GetVertexAttributeFormat.html). Additionally, it's important to note that certain meshes may lack specific attributes, such as [bone data](https://docs.unity.cn/ScriptReference/Rendering.VertexAttribute.BlendIndices.html).

Some attributes consistently have the same format and dimension. For instance, vertex positions are always in the format of Vector3. Since we anticipate the assistance of the Burst Compiler, we will use float3 for this purpose.

Here is the spider's mesh layout:

![Spider Layout!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-09-28-voxel-rendering-in-unity-part-1/8-Spider-Layout.png)

And here is the default Unity sphere:

![Sphere Inspector!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-09-28-voxel-rendering-in-unity-part-1/9-Sphere-Inspector.png)

![Sphere Layout!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-09-28-voxel-rendering-in-unity-part-1/10-Sphere-Layout.png)

And at this point, we use a generic struct to obtain the desired vertex attribute. Although it's generally not recommended to use unsafe code, I decided to try this approach. Here's the resulting logic:

1. Determine the stream to which the attribute belongs.

2. Obtain the attribute's offset from the start of its stream.

3. Obtain the stream's stride.

4. Obtain the stream as a byte array.

5. Obtain the unsafe pointer for this array and add the attribute's offset to it.

6. To get the vertex attribute's value by its index, multiply the index by the stride and add it to the attribute's start pointer.

7. At the address of the resulting pointer, read the required structure.

![Stream Reading!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-09-28-voxel-rendering-in-unity-part-1/11-Stream-Reader.png)

```c#
public readonly struct VertexAttributeReader {  
    private readonly int _streamStride;  
    private readonly unsafe byte* _streamPointer;  
  
    public VertexAttributeReader(Mesh.MeshData meshData, VertexAttribute vertexAttribute) {  
       var attributeStream = meshData.GetVertexAttributeStream(vertexAttribute);  
       var attributeOffset = meshData.GetVertexAttributeOffset(vertexAttribute);  
       _streamStride = meshData.GetVertexBufferStride(attributeStream);  
       unsafe {  
          _streamPointer = (byte*) meshData.GetVertexData<byte>(attributeStream).GetUnsafeReadOnlyPtr() + attributeOffset;     
       }  
    }  
  
    public unsafe T GetVertexAttribute<T>(int index) where T : unmanaged {  
       var vertexPointer = _streamPointer + index * _streamStride;  
       return *(T*)vertexPointer;  
    }  
}
```

And this is how reading the vertex position looks like:

```c#
public readonly struct VertexPositionReader {  
    private readonly VertexAttributeReader _attributeReader;  
      
    public VertexPositionReader(Mesh.MeshData meshData) {  
       _attributeReader = new VertexAttributeReader(meshData, VertexAttribute.Position);  
    }  
  
    public float3 GetVertexPosition(int index) {  
       return _attributeReader.GetVertexAttribute<float3>(index);  
    }  
}
```

Next, according to the earlier described algorithm we can check all the triangles. Just note a few moments:

- We determine the nearest triangle for the voxel through the sum of the squared distances from the triangle's vertices to the voxel's center because only relative value matters.

- Check if there is either no previously recorded triangle in this voxel or if the center of the current triangle is closer to the voxel's center than the previously recorded triangle's center before SAT to do less calculations.

- There is no ability to use multidimensional arrays in the JobSystem, therefore we will store the result in the plain array with length = x * y * z, where x, y and z is dimensions of meshes boundaries in voxels.


```c#
var voxelCenter = new float3(_voxelSize * k, _voxelSize * m, _voxelSize * n) +  
                  _boundsMin + _halfVoxel;  
var voxelIndex = k * _boxSizeYByZ + m * _boxSize.z + n;  
var existingVoxel = _voxels[voxelIndex];
var d0 = math.distancesq(p0, voxelCenter);;  
var d1 = math.distancesq(p1, voxelCenter);;  
var d2 = math.distancesq(p2, voxelCenter);;  
var distance = d0 + d1 + d2;  
if ((existingVoxel.MeshIndex == 0 || distance < existingVoxel.Distance) && SatTriangleIntersectsCube(p0, p1, p2, voxelCenter)) {  
    _voxels[voxelIndex] = new SatVoxel(voxelCenter, distance, meshIndex + 1, i0, i1, i2, p0, p1, p2);  
}
```

Now we have the array of voxels where each voxel stores information about it's mesh (index is 1 based, because 0 is used for blank voxels), positions and indices of the triangle's vertices.

Full code of this job can be found [here](https://github.com/utkaka/unity-instanced-voxels/blob/5d28d45f0fa94535e7a4d59d21a218a1d0fc5ab2/Assets/Scripts/InstancedVoxels/Voxelization/SatVoxelizerJob.cs).

Now, let's measure the performance of the jobified version of the SAT algorithm. To make the comparison more accurate, we will temporarily disable the calculation of the nearest triangle and the associated checks because these calculations were not present in my initial experiments with voxelization algorithms. The final execution time with Burst Compiler enabled is **50ms**, which is approximately 20 times faster than the original rough implementation.
### Voxel's color

Now that we know the ownership of each voxel, we can determine its color. To achieve this, we will also utilize the JobSystem.

First, we need to prepare the data. Unlike the Advanced Mesh API, Textures API don't offer a convenient way to obtain the "raw" data for multiple textures simultaneously. We could acquire them separately using the [GetPixelData](https://docs.unity3d.com/ScriptReference/Texture2D.GetPixelData.html) method. However, we won't be using it because there are too many different texture formats, making it impractical to create separate implementations for each. Therefore, we'll stick with the trusty [GetPixels](https://docs.unity3d.com/ScriptReference/Texture2D.GetPixels.html) method.

Similar to approach with the Mesh API, we could set up a series of separate jobs, each responsible for processing its own texture, however it is more reasonably to handle this task with a single job.

Therefore, we will store all the textures in a single large NativeArray and define the following structure for each mesh:

```c#
public readonly struct TextureDescriptor {  
    public int StartIndex { get; }  
    private readonly int _width;  
    private readonly int _height;  
  
    public TextureDescriptor(int startIndex, int width, int height) {  
       StartIndex = startIndex;  
       _width = width;  
       _height = height;  
    }  
  
    public int GetUvIndex(float2 uv) {  
       var x = (int)(_width * uv.x);  
       var y = (int)(_height * uv.y);  
       return StartIndex + _width * y + x;  
    }  
}
```

Some of our meshes might not have textures, so for now, we'll simply create a default white texture to assign to those meshes. One pixel is sufficient.

```c#
var whiteTexture = new Texture2D(1, 1, TextureFormat.RGBA32, false);  
whiteTexture.SetPixel(0, 0, Color.white);  
whiteTexture.Apply();
```

Now we need to determine the final size of the texture array. Since multiple meshes might share the same texture, we will only write it to the array once.

```c#
var texturesDictionary = new Dictionary<Texture2D, TextureDescriptor>();  
var texturesCopied = new Dictionary<Texture2D, bool>();  
  
texturesDictionary.Add(whiteTexture, new TextureDescriptor(0, 1, 1));  
texturesCopied.Add(whiteTexture, false);  
  
var texturesSize = 1;  
for (var i = 0; i < _textures.Length; i++) {  
    var texture = _textures[i];  
    if (texture == null) texture = whiteTexture;  
    if (texturesDictionary.ContainsKey(texture)) continue;  
    texturesDictionary.Add(texture, new TextureDescriptor(texturesSize, texture.width, texture.height));  
    texturesCopied.Add(texture, false);  
    texturesSize += texture.width * texture.height;  
}
```

Similar to the VertexPositionReader struct that we created to read vertex positions from the MeshData, we will create an almost identical VertexUVReader struct.

```c#
public readonly struct VertexUVReader {  
    private readonly VertexAttributeReader _attributeReader;  
      
    public VertexUVReader(Mesh.MeshData meshData) {  
       _attributeReader = new VertexAttributeReader(meshData, VertexAttribute.TexCoord0);  
    }  
  
    public float2 GetVertexUV(int index) {  
       return _attributeReader.GetVertexAttribute<float2>(index);  
    }  
}
```

I want to point out that I used float2 for UV. In most cases, that's correct. However, according to the documentation, UV can also be float3 or even float4. Therefore, additional checks for the dimensions of the TexCoord0 attribute and corresponding reading structures are necessary. Currently, I have skipped their implementation.

Now let's initialize all the necessary arrays.

```c#
var textureDescriptors = new NativeArray<TextureDescriptor>(meshData.Length, Allocator.TempJob,  
    NativeArrayOptions.UninitializedMemory);  
var vertexUvReaders = new NativeArray<VertexUVReader>(meshData.Length, Allocator.TempJob,  
    NativeArrayOptions.UninitializedMemory);  
var textures =  
    new NativeArray<Color>(texturesSize, Allocator.TempJob, NativeArrayOptions.UninitializedMemory);
```

Since we will be populating these arrays entirely ourselves, we can save a bit on their initialization by specifying the option NativeArrayOptions.UninitializedMemory. When we create a NativeArray without this option, Unity, by default, fills the allocated memory block with zeroes to prevent unexpected values.

Next, we'll populate these arrays.

```c#
for (var i = 0; i < _textures.Length; i++) {  
    var texture = _textures[i];  
    if (texture == null) texture = whiteTexture;  
    var descriptor = texturesDictionary[texture];  
    textureDescriptors[i] = descriptor;  
    vertexUvReaders[i] = new VertexUVReader(meshData[i]);  
    if (texturesCopied[texture]) continue;  
    texturesCopied[texture] = true;  
    var texturePixels = texture.GetPixels();  
    NativeArray<Color>.Copy(texturePixels, 0, textures, descriptor.StartIndex, texturePixels.Length);  
}
```

Now we can implement the job. When working with MeshData, we couldn't parallelize the job because multiple triangles could correspond to a single voxel. However, in this scenario, we can execute the job using multiple parallel threads because at each step, we are only reading and writing to a single voxel.

Job algorithm:

1. Iterate over the voxels obtained during the voxelization stage.

2. If voxel's MeshIndex is 0, skip it, as it's an empty voxel.

3. Use the VertexUVReader corresponding to the voxel's mesh to read the UV coordinates of the triangle's vertices.

4. Using [Barycentric coordinates](https://discussions.unity.com/t/calculate-uv-coordinates-of-3d-point-on-plane-of-meshs-triangle/60938), determine the UV coordinates for the voxel's center while employing one of the [properties of the cross product](https://en.wikipedia.org/wiki/Cross_product#Geometric_meaning).

![UV!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-09-28-voxel-rendering-in-unity-part-1/12-UV.png)

5. Read the color from the texture that corresponds to the voxel with resulting UV coordinates.

Full code of this job can be found [here](https://github.com/utkaka/unity-instanced-voxels/blob/5d28d45f0fa94535e7a4d59d21a218a1d0fc5ab2/Assets/Scripts/InstancedVoxels/Voxelization/ReadVoxelColorJob.cs).
### Summary

While we're still a long way from creating a game, we've achieved a promising result - the automatic and relatively fast voxelization of arbitrary (almost) meshes. We can already put the results to use by either generating a single voxelized mesh (since we've learned to use the Advanced Mesh API), or by instantiating individual voxel GameObjects, as I did for my demo GIF. With some material adjustments and enabling [GPU Instancing](https://utkaka.github.io/blog/2023/06/05/gpu-instanced-grass-in-unity.html), it may even run smoothly without significant lag.

You can review the complete source code [here](https://github.com/utkaka/unity-instanced-voxels).

I'd like to say that in the [next part](https://utkaka.github.io/blog/2023/11/16/voxel-rendering-in-unity-part-2.html), we'll be handling the runtime rendering of these voxels, but it's not time yet. In the upcoming section, we'll continue preparing the voxels for runtime use. Currently, we've generated only the 'outer' voxels of the model, but we also need the 'inner' ones (after all, hollow model destruction isn't as enjoyable). We'll also be adding bones and animations to our models. Stay tuned.