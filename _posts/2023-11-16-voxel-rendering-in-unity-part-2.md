![Voxelized spider!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-11-16-voxel-rendering-in-unity-part-2/1-Landing.gif)

Last time, we learned how to construct hollow voxel models from arbitrary (almost) meshes. However, we still lack animation support (static voxels are dull) and 'inner' voxels (which, in addition to aesthetics, will play a crucial role in subsequent runtime optimization). In this article, we'll address these aspects.

## Bones

As our primary focus is on skeletal animations (though regular MeshRenderer with Unity animations will also be supported; we'll simply consider them as models with a single bone), we need to determine which bone each voxel belongs to. In many ways, this procedure is similar to determining the color of a voxel. However, there are two differences that complicate things significantly:

1. We calculated a voxel's color using [barycentric coordinates](https://discussions.unity.com/t/calculate-uv-coordinates-of-3d-point-on-plane-of-meshs-triangle/60938), but we can't use the same approach for bones. This is because we now need to select a specific bone with the maximum weight for the given voxel.
2. Each vertex of a mesh has only one color (almost always), but there can be multiple bones with different weights for it.

I spent a considerable amount of time pondering how to apply the [Boyer–Moore majority vote algorithm](https://en.wikipedia.org/wiki/Boyer%E2%80%93Moore_majority_vote_algorithm). This would have allowed us to use constant space and process all the voxels in parallel. Unfortunately, it proved unsuccessful. The challenge lies in the fact that a voxel may lack a bone with the 'majority of votes.' In such cases, we must simply choose the bone with the highest weight, requiring us to sum the weights of all bones from the 3 vertices.

Therefore the final version of the algorithm is:

1. Read the information about bones of all vertices of the triangle which the voxel belongs to.
2. By the index of the bone we save it's weight multiplied by the vertex weight for the voxel.
3. If the bone with such index is already presents in the HashSet (during processing of previous vertex), then we add a new weight to the old one.
4. Traverse HashSet and determine the bone with the maximum weight.

Definitely, we could limit the number of bones in the project's settings or in the import settings of the mesh. Also, we could change the algorithm to determine the most weighted bone of each vertex at first. And only then select one from the three remaining.

![Single Bone Per Vertex!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-11-16-voxel-rendering-in-unity-part-2/1-Single-Bone-Per-Vertex.png)

But I deemed this approach less precise and opted for a more intricate solution.

![Multiple Bones Per Vertex Sum!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-11-16-voxel-rendering-in-unity-part-2/2-Multiple-Bones-Per-Vertex-Sum.png)

```c#
public void Execute(int index) {  
    _boneWeights.Clear();  
    var weightedVoxel = _weightedVoxels[index];  
    var meshIndex = weightedVoxel.MeshIndex - 1;  
    if (meshIndex < 0) return;  
    var bonesReader = _bonesReaders[meshIndex];  
  
    FillBoneWeights(bonesReader, weightedVoxel.VertexIndex0, weightedVoxel.VertexWeight0);  
    FillBoneWeights(bonesReader, weightedVoxel.VertexIndex1, weightedVoxel.VertexWeight1);  
    FillBoneWeights(bonesReader, weightedVoxel.VertexIndex2, weightedVoxel.VertexWeight2);  
      
    var maxBoneWeight = float.MinValue;  
    var maxBoneIndex = 0;  
    foreach (var bone in _boneWeights) {  
       if (bone.Value <= maxBoneWeight) continue;  
       maxBoneWeight = bone.Value;  
       maxBoneIndex = bone.Key;  
    }  
  
    _maxBoneIndex[0] = math.max(maxBoneIndex, _maxBoneIndex[0]);  
    _voxelBones[index] = maxBoneIndex;  
}  
  
private void FillBoneWeights(VertexBonesReader bonesReader, int vertexIndex, float vertexWeight) {  
    unsafe {  
       var vertexBoneIndexPointer = bonesReader.GetVertexBoneIndexPointer(vertexIndex);  
       var vertexBoneWeightPointer = bonesReader.GetVertexBoneWeightPointer(vertexIndex);  
       for (var i = 0; i < bonesReader.BonesCount; i++) {  
          var boneIndex = bonesReader.GetVertexBoneIndex(vertexBoneIndexPointer);  
          var boneWeight = bonesReader.GetVertexBoneWeight(vertexBoneWeightPointer) * vertexWeight;  
          if (!_boneWeights.ContainsKey(boneIndex)) _boneWeights.Add(boneIndex, boneWeight);  
          else _boneWeights[boneIndex] += boneWeight;  
          vertexBoneIndexPointer += bonesReader.IndexSize;  
          vertexBoneWeightPointer += bonesReader.WeightSize;  
       }  
    }  
}
```

Regrettably, employing NativeHashSet significantly complicates parallel execution. While theoretically feasible, it comes with a substantial memory overhead. Hence, we'll settle for sequential calculations within IJobFor.

Not for the first time, we find ourselves making performance compromises. Hence, I opted to take it a step further and utilize [Function pointers](https://docs.unity3d.com/Packages/com.unity.burst@1.8/manual/csharp-function-pointers.html) for reading bones. Undoubtedly, creating a distinct job for each conceivable scenario would be more optimal. However, given the abundance of cases with bones, such an approach would become overly intricate.

As I mentioned earlier, the JobSystem comes with some notable limitations when dealing with dynamic data structures, such as the inability to use interfaces. Function pointers provide a way to overcome at least some of these limitations with minimal loss.

Thus, the structure responsible for reading the bones of mesh vertices will look something like this:

```c#
private readonly FunctionPointer<VertexAttributeReadFunctions.ReadInt> _bonesIndexReader;
private readonly int _indexStride;
private readonly unsafe byte* _indexPointer;
private readonly int _indexSize;

...

public VertexBonesReader(Mesh.MeshData meshData) {
	_bonesIndexReader =
		VertexAttributeReadFunctions.GetIntFunctionPointer(meshData, VertexAttribute.BlendIndices);

	if (meshData.HasVertexAttribute(VertexAttribute.BlendIndices)) {
		_bonesCount = meshData.GetVertexAttributeDimension(VertexAttribute.BlendIndices);
		var indexStream = meshData.GetVertexAttributeStream(VertexAttribute.BlendIndices);
		var indexOffset = meshData.GetVertexAttributeOffset(VertexAttribute.BlendIndices);
		_indexStride = meshData.GetVertexBufferStride(indexStream);
		_indexSize = VertexAttributeReadFunctions.GetAttributeSize(meshData, VertexAttribute.BlendIndices);
		unsafe {
			_indexPointer = (byte*) meshData.GetVertexData<byte>(indexStream).GetUnsafeReadOnlyPtr() +
							indexOffset;
		}
	} else {
		_bonesCount = 0;
		_indexStride = 0;
		_indexSize = 0;
		unsafe {
			_indexPointer = (byte*) 0;
		}
	}
	
	...
}

public unsafe int GetVertexBoneIndex(byte* pointer) => _bonesIndexReader.Invoke(pointer);

public unsafe byte* GetVertexBoneIndexPointer(int vertexIndex) => _indexPointer + vertexIndex * _indexStride;

...
```

And obtaining the function pointer looks like this:

```c#
public unsafe delegate int ReadInt(byte* streamPointer);  

public static unsafe FunctionPointer<ReadInt> GetIntFunctionPointer(Mesh.MeshData meshData, VertexAttribute vertexAttribute) {  
    if (!meshData.HasVertexAttribute(vertexAttribute))  
       return BurstCompiler.CompileFunctionPointer<ReadInt>(ReadZeroInt);  
    var format = meshData.GetVertexAttributeFormat(vertexAttribute);  
    return format switch {  
       VertexAttributeFormat.UInt8 => BurstCompiler.CompileFunctionPointer<ReadInt>(ReadIntFromUInt8),  
       ...  
       _ => throw new ArgumentOutOfRangeException()  
    };  
}
  
[BurstCompile]  
[MonoPInvokeCallback(typeof(ReadInt))]  
public static unsafe int ReadZeroInt(byte* streamPointer) => 0;  
  
[BurstCompile]  
[MonoPInvokeCallback(typeof(ReadInt))]  
public static unsafe int ReadIntFromUInt8(byte* streamPointer) => *streamPointer;
```

## Obtaining inner voxels

At this stage, we've extracted all the necessary information from the original mesh. Now, we need to generate the missing data, specifically – color the inner voxels and assign bones to them. (I'm sure you'll agree, it would seem strange if only the external part of the voxel model is animated). To achieve this, we must identify which voxels precisely qualify as 'inner'.

Two possible algorithms come to my mind.

#### Scanline

Some sort of [Scanline Fill Algorithm](https://www.geeksforgeeks.org/scan-line-polygon-filling-using-opengl-c/). But simplier, because we are dealing with finite and limited set of voxels:

1. Iterate over X axis inside voxels bounds.
2. In the inner loop iterate over Y axis.
3. In the inner loop iterate over Z axis.
4. Find first non empty voxel.
5. FInd second non empty voxel.
6. Mark all voxels between them as inner.

![Scanline Fill!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-11-16-voxel-rendering-in-unity-part-2/3-Scanline.png)

However, this approach has two significant drawbacks. Firstly, it is necessary to visit each voxel inside the bounds. Secondly, there are possible border cases.

![Scanline Undefined 1!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-11-16-voxel-rendering-in-unity-part-2/4-Scanline-Undefined.png)

![Scanline Undefined 2!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-11-16-voxel-rendering-in-unity-part-2/5-Scanline-Undefined-2.png)

We can try to work around it by changing the axis order. But it doesn't look like a reliable solution. So let's move on to the second algorithm.

#### Breadth-first search

A simpler option appears to be the standard [breadth-first search](https://en.wikipedia.org/wiki/Breadth-first_search). However, we cannot start it 'from the inside' because we don't have any information about the inner voxels and wish to determine them. Instead, we will identify the outer voxels:

1. Add any outer voxel to the queue.
2. While the queue is not empty, take a voxel from it, mark it as outer, and add to the queue all its empty neighbors.

Here, we may encounter two problems:

1. Where should we begin the traversal? Attempting to start with a voxel at coordinates (0, 0, 0) is an option, but there's no guarantee it will be empty. Thus, we need to iterate through all the voxels until we find the first empty one.
2. Within the voxel grid boundaries, there may be multiple isolated areas of outer voxels.

![BFS Problems!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-11-16-voxel-rendering-in-unity-part-2/6-BFS-Problems.png)

We can solve both of these problems easily by simply expanding the voxel grid boundaries by one on each side!

![BFS Extended!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-11-16-voxel-rendering-in-unity-part-2/7-BFS-Extended.png)

However, there is at least one drawback: the mesh has to be convex; otherwise, the algorithm will simply traverse inside, and there won't be any 'inner' voxels at all. This particular issue can be addressed by using a scanline algorithm and experimenting with axis orders. But, as per the plan, all our meshes will be convex (and if not, we can always make them so). Therefore, we choose to overlook this drawback.

## Bones of inner voxels

Now we can confidently say that any empty voxel not classified as an outer one is an inner voxel. Unlike the initial voxelization, we will now begin with bone calculation.

We will do this in several stages.

1. First, we need to traverse all the non-empty voxels and determine the bounds of each bone in the voxel space.
![Bones Bounds!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-11-16-voxel-rendering-in-unity-part-2/8-Bones-Bounds.png)
2. Next, we iterate over the inner voxels and determine which bone bounds they fall within.
3. If there is only one bone, we assign it to the voxel. If there are multiple bones, we add the voxel to a separate queue.
![Bones Single Bound!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-11-16-voxel-rendering-in-unity-part-2/9-Bones-Single-Bound.png)
4. Next, we take each voxel from the queue and examine its neighbors. If all six neighbors have a bone assigned to them, or if at least **2\*** neighbors share the same bone, we assign the voxel the most frequently occurring bone. Otherwise, we return the voxel to the queue for later processing.
![Bones Multiple Bounds!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-11-16-voxel-rendering-in-unity-part-2/10-Bones-Multiple-Bounds.png)

*\*In theory, the number here should be 3, as if 3 neighbours have the same bone, there cannot be another bone with a greater value. However, in some borderline cases, the algorithm got stuck in an infinite loop, so I had to relax the condition. Despite the slight compromise in mathematical precision, the result is acceptable.*

## Colors of inner voxels

For simplicity, we will paint inner voxels with the color red. It might seem that we can iterate over all inner voxels without assigned colors. However, there's a nuance. During the animation, gaps may appear between different bones, revealing inner voxels. In the case of the color red, this can be quite noticeable.

![Visible Inner Voxels!](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2023-11-16-voxel-rendering-in-unity-part-2/11-Visible-Inner-Voxels.png)

Thus, it would be useful to paint voxels on the borders between bones with colors representing the outer skin:

1. Iterate over inner voxels without a color.
2. If the voxel's bone differs from that of any neighbours, add it to the painting queue.
3. In the subsequent loop, take a voxel from the queue. If any of its neighbours has a color, assign that color to the voxel. Otherwise, return it to the queue.

And only after that, we can iterate over the remaining voxels without a color and assign them the red color.

## Animation

Now that we precisely know which bones we are interested in, we can proceed to bake the animations. We will extract only the bone positions and rotations from the source animations, as scaled voxels might appear less appealing. For the simplicity of the sampling process, we will use legacy animations. The process shares similarities with [GPU animation baking](https://github.com/chenjd/Render-Crowd-Of-Animated-Characters/tree/master).

1. Save the initial position and rotation of each bone.
2. Among all the animations of the meshes we are baking, determine the maximum frame rate and animation duration.
3. Sample each frame of every animation.
4. Save the translation and rotation of each bone relative to their initial values.

During the process, I encountered a small challenge. The issue arises from the fact that we voxelize the mesh in its original form as exported by the modeler (e.g., T-pose). However, in animations, bones can be significantly translated from their initial positions. While this isn't a major concern for SkinnedMeshRenderer, where triangles simply stretch, voxels cannot adapt in the same way. Consequently, gaps between limbs occur, and the animation looks less appealing. To address this, I had to sample the first frame of the animation before the voxelization process. This approach significantly enhances the visual result.

However, we voxelize meshes using MeshData, while animation is sampled on the SkinnedMeshRenderer, which does not affect the source MeshData. Here, [SkinnedMeshRenderer.BakeMesh](https://docs.unity3d.com/ScriptReference/SkinnedMeshRenderer.BakeMesh.html) comes in handy, but it loses the information about bone assignments for vertices. Not very elegant, but we will assign them manually:

```c#
var bakedMesh = new Mesh();
skinnedMesh.BakeMesh(bakedMesh);
bakedMesh.boneWeights = skinnedMesh.sharedMesh.boneWeights;
```

## Serialization

Now that we have the complete set of necessary data that needs to be preserved for future runtime usage. Storing data for each voxel could result in a significantly larger asset size. And it leads to increased build size, prolonged loading times, and additional inconvenience in the editor. Therefore, we will take the following steps:

1. There's no need to store information about every voxel within the model boundaries. It's sufficient to save data for non-empty voxels only.
2. Using 32 bits for each color channel is excessive. Furthermore, we don't require the alpha channel (transparent voxels would complicate future runtime optimizations and are not of interest at this stage). Therefore, we only need 3 bytes (one per channel) to store voxel colors.
3. We will currently constrain the voxel space to dimensions of 255x255x255. Therefore, one byte per coordinate is sufficient. If needed, dynamic space dimensions can be added later.
4. In the vast majority of cases, the number of bones will be significantly below 255, so allocating one byte for the bone index is sufficient. As with coordinates, we will keep the possibility of dynamic dimensions in mind for the future.
5. We will serialize this data into byte arrays to save space on text serialization in the editor. Additionally, this approach will significantly simplify implementation in case we decide to add dynamic dimensions to attributes in the future.

## Summary

We now have a complete voxel model with animation. I've created test classes to render these models using regular GameObjects. If you attempt to run the demo with a voxel size of 0.01, you'll likely experience performance issues on your PC. Considering our goal to render this on mobile devices, our next step will focus on optimizing the runtime rendering.