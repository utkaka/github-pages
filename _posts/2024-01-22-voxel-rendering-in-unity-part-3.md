---
title: Voxel rendering in Unity (Part 3)
date: 2024-01-22
---

![Spider demo](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/1-Landing.gif)

Now, we've arrived at the most intriguing part - runtime rendering of the voxel model. In the [previous article's](https://utkaka.github.io/blog/2023/11/16/voxel-rendering-in-unity-part-2.html) conclusion, we already confirmed that using individual GameObjects for each voxel severely limits the number of voxels available to us. Therefore, we will explore all possible optimization techniques to enable us to offer a greater number of voxels to a wider range of mobile devices.

## Performance tests

We'll proceed step by step, measuring performance each time using the [Unity Performance Testing Package](https://docs.unity3d.com/Packages/com.unity.test-framework.performance@3.0/manual/index.html). This will enable us to continually evaluate the success of our actions and make decisions about the next steps more easily.

Generally, whenever we discuss optimization, performance tests are the most convenient means of evaluation. They do, of course, require a bit of extra effort to create tests, but they are much more accurate than a simple FPS counter. Moreover, unlike the profiler, they enable us to make comparative analyses of multiple solutions simultaneously.

Since we are tackling a comprehensive task that involves both CPU and GPU, we will be measuring frame time. This approach is, in essence, not significantly different from an FPS counter. However, tests allow us to additionally measure specific [ProfilerMarkers](https://docs.unity3d.com/Packages/com.unity.test-framework.performance@3.0/manual/measure-profile-markers.html).

The issue arises when rendering voxels, as we quickly reach the performance limits of the GPU, especially on mobile devices. Hence, our aim is to distribute a portion of the workload to the CPU, ensuring it remains evenly balanced. Otherwise, we might encounter a scenario where we completely alleviate the GPU's workload while overburdening the CPU, resulting in an unexpected decrease in FPS rather than an increase. For this very reason, a standard FPS counter is not suitable for evaluating performance. This is because successful GPU optimization may go unnoticed, as the problem might reside in the CPU, and the overall frame time remains unchanged.

By default, frame time in performance tests is measured in milliseconds. However, we will examine timings in more detail, using microseconds. Therefore, we are interested in two markers:

1. _PlayerLoop_ - nearly matches the frame time (CPU + GPU). 
2. [_Gfx.WaitForPresentOnGfxThread_](https://docs.unity3d.com/Manual/profiler-markers.html#rendering) - this marker enables us to assess GPU workload. For StandaloneOSX tests, it should be replaced with _GfxDeviceMetal.WaitForLastPresent_.

Thus, by comparing the difference between these two markers, we can identify the bottleneck.

To avoid the need to copy and paste code for each rendering variant, we will create an abstract class with all the necessary initialization:

```c#
public abstract class VoxelRendererPerformanceTest<T> where T : MonoBehaviour, IVoxelRenderer {  
    protected IEnumerator Test(string voxelSize) {  
       var sampleGroups = new []{  
          new SampleGroup("PlayerLoop", SampleUnit.Microsecond),   
          //new SampleGroup("Gfx.WaitForPresentOnGfxThread", SampleUnit.Microsecond),  
          new SampleGroup("GfxDeviceMetal.WaitForLastPresent", SampleUnit.Microsecond)  
       };  
         
       var cameraTransform = new GameObject("Main Camera", typeof(Camera)).transform;  
       cameraTransform.gameObject.tag = "MainCamera";  
       cameraTransform.position = new Vector3(0.38f, 1.17f, -1.44f);  
       cameraTransform.rotation = Quaternion.Euler(32.06f, -12.81f, 0.0f);  
  
       var renderer = new GameObject("Renderer", typeof(T))  
          .GetComponent<T>();  
         
       var voxels = Resources.Load<Voxels>($"Voxel_{voxelSize}");  
       renderer.Init(voxels);  
       yield return Measure.Frames()  
          .WarmupCount(60)  
          .ProfilerMarkers(sampleGroups)  
          .MeasurementCount(60)  
          .Run();  
       Object.Destroy(renderer.gameObject);  
       Object.Destroy(cameraTransform.gameObject);  
    }  
}
```

Therefore, a test for each renderer will appear something like this:

```c#
public class GameObjectVoxelRendererTests : VoxelRendererPerformanceTest<GameObjectVoxelRenderer> {  
    private static IEnumerable TestCases() {  
       yield return "008";  
       yield return "004";  
       yield return "002";  
    }  
  
    [UnityTest, Performance]  
    public IEnumerator PerformanceTest([ValueSource(nameof(TestCases))]string voxelSize) {  
       return Test(voxelSize);  
    }  
}
```

## GameObjectVoxelRenderer

We will start with a basic code that I wrote for demonstrations in the previous article. Each voxel is represented as a separate GameObject, placed within another GameObject corresponding to the voxel's bone. It's quite straightforward. However, in the previous article, I didn't explain how the animation is played, so I will address that now.

During the voxelization process, we saved two arrays: the translation of each bone in every animation frame and the same for rotation.

![Animation Layout](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/2-Animation-Layout.png)

The algorithm for playing the animation:

1. Based on the animation time during the current rendering frame, we determine the indices of the current and the next animation frames within the range \[0, AnimationLength).

```c#
_animationTime += _animationFrameRate * Time.deltaTime;
if (_animationTime >= _animationLength) {
	_animationTime -= _animationLength;
}
_animationCurrentFrame = (int)_animationTime;
_animationNextFrame = (_animationCurrentFrame + 1) % _animationLength;
```

2. Calculate the position of the animation between these two frames.

```c#
_animationLerpRatio = _animationTime - _animationCurrentFrame;
```

3. For the translation and rotation of each bone, we take values from the pre-baked arrays and perform a lerp between them.

```c#
_animationCurrentFrame += _boneIndex * _voxelsAnimation.FramesCount;  
_animationNextFrame += _boneIndex * _voxelsAnimation.FramesCount;
var animationPosition = math.lerp(_voxelsAnimation.AnimationBonesPositions[_animationCurrentFrame],  
    _voxelsAnimation.AnimationBonesPositions[_animationNextFrame], _animationLerpRatio);  
var animationRotation = math.lerp(_voxelsAnimation.AnimationBonesRotations[_animationCurrentFrame],  
    _voxelsAnimation.AnimationBonesRotations[_animationNextFrame], _animationLerpRatio);
```

Due to lerp the animation looks much more smooth and natural.

![Lerp Animation](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/3-Lerp-Animation.gif)

However, in general, it is possible to skip it and use only the current frame. This may better suit the visual style of certain games.

![Simple Animation](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/4-Simple-Animation.gif)

Not everyone can see the difference on the gifs, but in reality, it is very clear.

Now let's examine the test results. From here onwards, the tests will be conducted using a model of a voxelized unit cube. Median values are used as timings. The numbers in the test names indicate the voxel count.

|Test Name|PlayerLoop (µs)|GfxDeviceMetal.WaitForLastPresent (µs)|
|---|---|---|
|2197|2221.37|0.20|
|16900|19997.06|0.33|
|130050|193833.25|0.41|

Nothing surprising, really. As the number of GameObjects increases, the CPU becomes more heavily loaded, while the GPU doesn't even notice the load.

## InstancedCubesRenderer

Next, I took the code from [my old article about grass rendering](https://utkaka.github.io/blog/2023/06/05/gpu-instanced-grass-in-unity.html) and made some modifications. In particular, I had to add animation logic to the shader. In essence, it's similar to what was described earlier, with the exception that now we don't have parent objects for bones, so we need to rotate each voxel around its bone individually. And we will accomplish this within the shader.

```c
const float3 animation_position = lerp(bone_positions_animation_buffer[current_animation_frame], bone_positions_animation_buffer[next_animation_frame], animation_lerp_ratio);
const float4 animation_rotation = lerp(bone_rotations_animation_buffer[current_animation_frame], bone_rotations_animation_buffer[next_animation_frame], animation_lerp_ratio);
const float3 bone_position = bone_positions_buffer[voxel_bone];
const float3 offset_point = voxel_vertex_position - bone_position;
const float3 rotated_point = mul_quaternion(animation_rotation, offset_point) + bone_position;
voxel_position_out = rotated_point + animation_position;
```

|Test Name|PlayerLoop (µs)|GfxDeviceMetal.WaitForLastPresent (µs)|
|---|---|---|
|2197|265.75|0.29|
|16900|199.64|0.20|
|130050|676.93|422.79|
|**1020100**\*|4486.10|2942.14|
\* **New test**

In the case of instancing, noticeable issues only begin to appear with 1,020,100 voxels. It's evident from the table that the load starts shifting towards the GPU. Once again, nothing surprising, as the cube consists of 24 vertices, and our scene, despite its visual simplicity, contains 24 * 1,020,100 = 24,482,400 vertices. Therefore, it is crucial to reduce this number.

![Frame debugger](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/5-Frame-Debugger.png)

Switching from GameObjects to GPU instancing significantly expanded our capabilities. Therefore, for further tests, I decided to introduce a case with a voxel size of 1/255. This is because our serialization allows us to store a model with a maximum size of 255x255x255 voxels. It was during this phase that I noticed several errors in the voxelizer. The most noticeable one was related to the insufficient precision of the SatVoxelizerJob. Initially, out of habit, I used floats, but in reality, this level of precision was insufficient. Consequently, I had to perform all calculations in this job using [double](https://github.com/utkaka/unity-instanced-voxels/commit/2bed271426a05af09da89c3c57fcf7b685097615).

## Inner voxels culling

The first thing that comes to mind is culling the inner voxels, as we cannot see them by definition. In the previous article, we determined them using breadth-first search, but this time we will take a different approach. Partly because when we make our model destructible, some of the inner voxels will become outer after destruction. Performing a breadth-first search on all the voxels each time can be quite costly. It would be beneficial to have a solution that can be executed in parallel.

Therefore, we will individually check each voxel. If it has non-empty neighbors on all 6 sides, then it is inside the model and is invisible. Allow me to remind you that we store only non-empty voxels in memory, just as we serialized them during voxelization. However, this makes finding voxel neighbors more challenging.

To simplify this task, a three-dimensional array with sizes of \[xSize, ySize, zSize\] will assist us. For each non-empty voxel, we will assign a value of 1 in its corresponding integer coordinates. Thus, to check for the presence of a left neighbor, we need to examine the value at indices \[x - 1, y, z\].

Since we strive to perform all possible calculations using the Job System, we are deprived of the straightforward option of using multidimensional arrays. Therefore, we will initialize a simple one-dimensional array with a size of xSize*ySize*zSize. We will then transform a three-dimensional index into a one-dimensional index using the following formula:

```c#
public readonly int GetVoxelIndex(byte3 voxelPosition) {  
    return voxelPosition.x * _sizeYByZ + voxelPosition.y * _size.z + voxelPosition.z;  
}
```

It's worth mentioning that we used this approach during the voxelization process, and we have a special structure called [VoxelsBox](https://github.com/utkaka/unity-instanced-voxels/blob/f1e3a036c45b446df706367845b4ce5590fa3497/Assets/Scripts/com.utkaka.InstancedVoxels/Runtime/VoxelData/VoxelsBox.cs) for all calculations of that kind.

We can obtain the indices of neighboring voxels using the function above, but we can also do without unnecessary multiplications. It is sufficient to understand the displacements along each of the coordinate axes in the grid:

![Coordinates to index](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/6-Coordinates-To-Index.png)

```c#
[MethodImpl(MethodImplOptions.AggressiveInlining)]  
public readonly int GetLeft(int voxelIndex) {  
    return voxelIndex - _sizeYByZ;  
}
...
[MethodImpl(MethodImplOptions.AggressiveInlining)]  
public readonly int GetBottom(int voxelIndex) {  
    return voxelIndex - _size.z;  
}
... 
[MethodImpl(MethodImplOptions.AggressiveInlining)]  
public readonly int GetBack(int voxelIndex) {  
    return voxelIndex - 1;  
}  
...
```

But here, a potential problem arises. We need to additionally check that a voxel is not on the grid's edges. Otherwise, we can get the wrong neighbour:

![Wrong index](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/7-Wrong-Index.png)

Or we can go beyond the array's bounds:

![Wrong index](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/8-Wrong-Index.png)

We will call these functions frequently and extensively, so we'd like to avoid additional checks and code branching. Therefore, we will virtually extend the grid's boundaries by 1 voxel on each side:

![Extended box](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/9-Extended-Box.png)

The 0-based coordinates in the source grid will become 1-based in the new grid. Therefore, we will add one more function for proper conversion:

```c#
public readonly int GetExtendedVoxelIndex(byte3 voxelPosition) {  
    return (voxelPosition.x + 1) * _sizeYByZ + (voxelPosition.y + 1) * _size.z + voxelPosition.z + 1;  
}
```

Next, we simply replace the serialized VoxelBox with the new extended one:

```c#
var box = new VoxelsBox(_voxels.Box.Size + new int3(2, 2, 2));
```

And the array that we will use to store information about empty/non-empty voxels, we will initialize with the size of the new VoxelBox:

```c#
var voxelBoxMasks = new NativeArray<byte>(box.Count, Allocator.TempJob);
var voxelBoxBones = new NativeArray<byte>(box.Count, Allocator.TempJob);
```

You may notice that we are creating two arrays here. I've already mentioned the reason in the previous article. Gaps may appear between neighboring bones during the animation playback. Therefore, we won't discard voxels that have neighbors on all six sides, but at least one of these neighbors belongs to another bone.

Let's consider an ideal scenario with a single bone: a voxelized cube with 255x255x255 voxels. And calculate how much the scene will be lightened with this optimization. This is precisely why I chose a cube for the tests: it's much simpler to perform theoretical calculations on paper and then verify them in practice.

First, let's consider a 2D scenario visually:

![Voxels count](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/10-Voxels-Count.png)

Now, let's calculate. Initially, the model has:
1. 255\*255\*255 = 16581375 voxels.
2. 16581375\*24 = 397953000 vertices.

Without inner voxels: 
1. 255\*255\*2 + 255\*253\*2 + 253\*253\*2 = 130050 + 129030 + 128018 = 387098 voxels. Such a complex formula arises because the faces share common edges and, consequently, common rows of voxels.
2. 387098\*24 = 9290352 vertices.

Now, let's check the Frame Debugger:

![Frame debugger](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/11-Frame-Debugger.png)

We've simplified geometry by 397953000 / 9290352 ≈ 42 times!

It's time to check the tests:

|Test Name|PlayerLoop (µs)|GfxDeviceMetal.WaitForLastPresent (µs)|
|---|---|---|
|2197|208.83|0.16|
|16900|216.79|0.29|
|130050|269.35|0.29|
|1020100|349.97|96.08|
|**16581375**\*|1473.81|1172.83|
\* **A new test**

GPU workload has decreased by ≈ 30 times!
## Replace cubes with quads

Now, it's time to remember that in addition to inner voxels, there are also inner faces of outer voxels, which are also invisible and shouldn't be rendered.

But it will require us to render a voxel with six quads instead of a single cube, which will, of course, introduce some overhead. However, the benefits gained from simplifying the geometry should outweigh this.

First of all, let's establish the criteria for when a voxel face should not be rendered. This occurs when there is a neighboring voxel with the same bone on the same side. Consequently, each face renderer will have its own list of voxels to render.

Here, I will provide a more detailed description of the culling process, which I deliberately omitted in the previous step. Of course, we can perform a straightforward check of neighboring voxels, something along these lines:

```c#
isVisibleFromLeft = voxelBoxMasks[_voxelBox.GetExtendedVoxelIndex(voxelPosition - new byte3(-1, 0, 0))] == 1;
```

But, as a reminder, our voxels will be destroyed, and during such moments, we'll need to recalculate culling. In the previously described approach, we would have to traverse all the voxels to check the state of their neighbors. However, it would be more efficient if a voxel could inform its neighbors about its destruction.

"An observant reader may have noticed earlier that the auxiliary array for culling is named _**voxelBoxMasks**_. This is because we will utilize [bitmasks](https://en.wikipedia.org/wiki/Mask_(computing)) to store comprehensive information about a voxel's neighbors. In this array, we won't merely store 1/0 values to represent empty and non-empty voxels.

1. We will preassign a corresponding bit for each side of the cube.

![Bitmask](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/12-Bitmask.png)

2. Go through all the non-empty voxels and set the value to 1 in the corresponding index of _**voxelBoxMasks**_.
3. Once again, iterate through all the non-empty voxels and check their neighbors. If there is a neighbor on a particular side, then set the corresponding bit in the mask to 1.

![Bitmask example](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/13-Bitmask-Example.png)

```c#
var mask = 1;  
var neighbourIndex = _voxelsBox.GetLeft(voxelIndex);  
mask |= (_voxelBoxMasks[neighbourIndex] & 1) << 1;  
neighbourIndex = _voxelsBox.GetRight(voxelIndex);  
mask |= (_voxelBoxMasks[neighbourIndex] & 1) << 2;  
neighbourIndex = _voxelsBox.GetBack(voxelIndex);  
mask |= (_voxelBoxMasks[neighbourIndex] & 1) << 3;  
neighbourIndex = _voxelsBox.GetFront(voxelIndex);  
mask |= (_voxelBoxMasks[neighbourIndex] & 1) << 4;  
neighbourIndex = _voxelsBox.GetBottom(voxelIndex);  
mask |= (_voxelBoxMasks[neighbourIndex] & 1) << 5;  
neighbourIndex = _voxelsBox.GetTop(voxelIndex);  
mask |= (_voxelBoxMasks[neighbourIndex] & 1) << 6;
_voxelBoxMasks[voxelIndex] = (byte)mask;
```

4. In separate jobs for each side, we can check if this side of the voxel should be added to the rendering list:

```c#
if ((_voxelBoxMasks[_voxelsBox.GetExtendedVoxelIndex(voxelIndex)] & _sideMask) == _sideMask) return;  
_visibleVoxelsIndices.AddNoResize(index);
```

Furthermore, when a voxel is destroyed, we can simply modify the bitmasks of its neighbors to account for the change. This way, we won't need to iterate through all the voxels again with checks.

The last remaining question is how to handle the border voxels between bones. For this purpose, we've previously created a second array called _**voxelBoxBones**_, where we will also store bitmasks. However, this time these bitmasks will indicate whether a voxel's bone matches the bone of its neighbor.

```c#
var mask = 1;  
var bone = _voxelBoxBones[voxelIndex];  
var neighbourIndex = _voxelsBox.GetLeft(voxelIndex);  
mask |= math.select(0, 2, bone == _voxelBoxBones[neighbourIndex]);  
neighbourIndex = _voxelsBox.GetRight(voxelIndex);  
mask |= math.select(0, 4, bone == _voxelBoxBones[neighbourIndex]);  
neighbourIndex = _voxelsBox.GetBack(voxelIndex);  
mask |= math.select(0, 8, bone == _voxelBoxBones[neighbourIndex]);  
neighbourIndex = _voxelsBox.GetFront(voxelIndex);  
mask |= math.select(0, 16, bone == _voxelBoxBones[neighbourIndex]);  
neighbourIndex = _voxelsBox.GetBottom(voxelIndex);  
mask |= math.select(0, 32, bone == _voxelBoxBones[neighbourIndex]);  
neighbourIndex = _voxelsBox.GetTop(voxelIndex);  
mask |= math.select(0, 64, bone == _voxelBoxBones[neighbourIndex]);  
_boneMasks[index] = (byte)mask;
```

The corresponding bit for the side will be set to 1 if the neighbor from that side has the same bone.

It will help us adjust the visibility mask from the code above:

```c#
var mask = 1;  
var neighbourIndex = _voxelsBox.GetLeft(voxelIndex);  
mask |= (_voxelBoxMasks[neighbourIndex] & 1) << 1;  
neighbourIndex = _voxelsBox.GetRight(voxelIndex);  
...
mask &= _boneMasks[index];
_voxelBoxMasks[voxelIndex] = (byte)mask;
```

![Bitmask with bones](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/14-Bitmask-With-Bones.png)

```c#
if ((_voxelBoxMasks[_voxelsBox.GetExtendedVoxelIndex(voxelIndex)] & _sideMask) == _sideMask) return;  
_visibleVoxelsIndices.AddNoResize(index);
```

It's worth mentioning that this part was actually implemented during the previous step of culling inner voxels. However, the culling condition was slightly different because we were concerned with the presence of all 6 neighbors from all 6 sides at once:

```c#
if (_voxelBoxMasks[voxelIndex] >= 127) return;  
_positions.AddNoResize(_inputPositions[index]);
```

Now, let's calculate how many vertices we've saved. The total number of voxels remains unchanged, but the number of vertices should decrease by approximately a factor of 6:
1. 255\*255\*6 = 390150 quads.
2. 390150\*4 = 1560600 vertices.

![Frame debugger](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/15-Frame-Debugger.png)

Now let's see what this has given us:

|Test Name|PlayerLoop (µs)|GfxDeviceMetal.WaitForLastPresent (µs)|
|---|---|---|
|2197|297.14|0.45|
|16900|287.25|0.37|
|130050|285|0.37|
|1020100|290.22|0.41|
|16581375|572.16|316.6|

GPU workload has decreased by about 4 times, total frame time decreased by 2.5 times!

Let's additionally compare the rendering of cubes and quads when only the inner voxels are culled in both cases, but the inner faces of quads are not culled, resulting in the same number of vertices in both scenarios.

|Test Name|PlayerLoop (µs)|GfxDeviceMetal.WaitForLastPresent (µs)|
|---|---|---|
|InstancedCubesRenderer (2197)|208.83|0.16|
|InstancedQuadsRenderer (2197)|286.12|0.33|
|InstancedCubesRenderer (16900)|216.79|0.29|
|InstancedQuadsRenderer (16900)|291.41|0.37|
|InstancedCubesRenderer (130050)|269.35|0.29|
|InstancedQuadsRenderer (130050)|292.33|0.39|
|InstancedCubesRenderer (1020100)|349.97|96.08|
|InstancedQuadsRenderer (1020100)|564.64|299.27|
|InstancedCubesRenderer (16581375)|1473.81|1172.83|
|InstancedQuadsRenderer (16581375)|2930.70|2541.25|

As the number of voxels increases, quads perform worse. Without additional culling of invisible faces, they would serve no purpose. Overall, the result is quite predictable, but I wanted to provide evidence for this.

And of course, the question arises: can we improve this result? Yes, we can, but not to such a significant extent, and it won't be as straightforward.

## Back-face culling

Up to this point, we have been culling voxels and faces that are guaranteed to be invisible regardless of the position of the model and the camera. However, there are faces whose visibility depends on the camera. So, let's consider how we can implement [back-face culling](https://en.wikipedia.org/wiki/Back-face_culling).

For starters, we need to understand how we can determine the visibility of a face.

![Side visibility](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/16-Side-Visibility.png)

"It's worth noting that it depends on the angle between the vector from the camera to the face and the normal vector of the face itself. If this angle falls within the range (90°, 270°), then the face is visible.

![Side visibility angle](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/17-Side-Visibility-Angle.png)

And we don't even need to calculate the exact angle value, as it would be computationally expensive. It's enough to recall the [geometric definition of the dot product](https://en.wikipedia.org/wiki/Dot_product#Geometric_definition). This definition implies that the sign of the dot product of two vectors depends on the cosine of the angle between them. The cosine is negative within the range (90°, 270°).

So, for a voxel face to be visible, the dot product of the vector from the camera to it and its normal vector should be less than 0.

Additionally, we can cull the voxels that are behind the camera. In this case, the dot product between the camera's forward vector and the vector from the camera to the voxel should be positive.

![Side visibility camera](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/18-Side-Visibility-Camera.png)

Don't forget to apply animation rotation and translation to the voxel. As a result, we obtain the following function for checking the visibility of the voxel's face:

```c#
private bool IsVisible(byte3 voxel, float3 bonePosition, float3 animationPosition, float4 animationRotation) {  
    var voxelPosition = new float3(voxel.x, voxel.y, voxel.z) * _voxelSize + _startPosition;  
    var quaternion = new quaternion(animationRotation);  
    var offsetPoint = voxelPosition - bonePosition;  
    var rotatedPoint = math.mul(quaternion, offsetPoint) + bonePosition;  
    voxelPosition = rotatedPoint + animationPosition;  
    var cameraToVoxel = voxelPosition - _cameraPosition;  
    return math.dot(_cameraForward, cameraToVoxel) >= 0 && math.dot(math.mul(quaternion, _sideNormal), cameraToVoxel)  <= 0.0f;  
}
```

And here is another challenge. In the previous culling step, we reduced the number of faces to 390150. However, it would be inefficient to calculate the dot product for each of them. Moreover, in a game, the camera can move quite dynamically, and ideally, we should perform these calculations every frame. Otherwise, players will occasionally see disappearing voxels.

Ideally, we would like to perform these calculations for a large group of voxels simultaneously. However, we need to take into account that the model may consist of different bones, each with its own rotation and translation.

Initially, I calculated the visibility of faces specifically for the bones:

1. Take the position of the bone.
2. Apply the translation and rotation of the current frame of the animation to it.
3. Calculate the dot product of its normal and the vector from the camera to the resulting position for each face.
4. Check if this face is visible for the bone of this voxel during the culling of the voxel's faces..

In the case of the test spider, this approach worked acceptably. At least not noticeably to the eye. But this was possible mainly because the spider has 14 bones, and the voxels are relatively compactly distributed within them.

Let's see what happens when voxels of the same bone are located at a significant distance from each other.

![Big bone](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/19-Big-Bone.png)

It's noticeable that within one bone, each face has its own visibility boundaries. And we need to somehow calculate them.

Let's begin with the simplest two-dimensional case in the image above.

As we can see, along the Y-axis, the visibility boundaries of the face align with the model's boundaries. However, along the X-axis, they are smaller, and we need to find them. Of course, we can iterate through all the voxels along the axis and find the point where the face's visibility changes. Additionally, our model can have no more than 255 voxels in any of the dimensions. However, as we will come to understand later, we will need to perform many such passes due to the third dimension, possible rotations, and the number of bones (remember that each bone will have separate boundaries for each of the 6 faces). So, let's take another look at the task:

![Side visibility plain](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/20-Side-Visibility-Plain.png)

And for the opposite face, the scenario will be as follows:

![Other side visibility plain](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/21-Other-Side-Visibility-Plain.png)

In front of us, there is an ordered array. Let's assume that invisibility of the face corresponds to a lower value, while visibility corresponds to a higher one. Then, on the first image, we have a decreasing array, and on the second one, we have an increasing array.

I believe it's possible to mathematically prove that if we initially have invisible voxels followed by visible voxels, then invisible voxels cannot reappear afterward. However, this article is not intended to be a scientific paper, so let's simply assume that we are working with ordered arrays. When we need to find an element in an ordered array, what do we do? Of course, we use [binary search](https://en.wikipedia.org/wiki/Binary_search_algorithm).

In the case of a "decreasing" array, we will search for the rightmost visible voxel, and in the case of an "increasing" one, we will search for the leftmost visible. Thus, we have the following function:

```c#
private int2 GetAxisBounds(byte3 start, byte3 axis, int boxAxisSize, float3 bonePosition, float3 animationPosition, float4 animationRotation) {  
    var left = 0;  
    var right = boxAxisSize + 1;  
    if (IsVisible(start, bonePosition, animationPosition, animationRotation)) {  
       //rightmost visible  
       while (left < right) {  
          var middle = (left + right) / 2;  
          if (!IsVisible(start + axis * middle, bonePosition, animationPosition, animationRotation)) {  
             right = middle;  
          } else {  
             left = middle + 1;  
          }  
       }  
       return new int2(0, right - 1);  
    }  
    //leftmost visible  
    while (left < right) {  
       var middle = (left + right) / 2;  
       if (!IsVisible(start + axis * middle, bonePosition, animationPosition, animationRotation)) {  
          left = middle + 1;  
       } else {  
          right = middle;  
       }  
    }  
    return new int2(left, boxAxisSize);  
}
```

Now, let's return to the two-dimensional scenario. We considered an ideal case above. However, due to the rotations of the bone and the camera, and the fact that we also cull voxels behind the camera, a less convenient scenario is possible:

![Rotated camera](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/22-Rotated-Camera.png)

Boundaries along one axis are clearly not sufficient, and we need to find boundaries along both axes. The question is how to do it, as the visibility of the voxel now depends on both axes **simultaneously**.

First, we will find the square area where all voxels are guaranteed to be visible. To do this, we will perform a binary search along the diagonal:

![Binary diagonal](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/23-Binary-Diagonal.png)

Next, we can search for boundaries along each axis separately because we are confident that the values of the second axis in this area do not affect the visibility of the voxel. Again, due to rotations, we will perform two searches for each axis within this area.

![Binary borders](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/24-Binary-Borders.png)

To ensure that no border voxel is accidentally culled, we conservatively take the minimum and maximum values from all four searches.

![Total area](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/25-Total-Area.png)

And now we have got such function:

```c#
private int4 GetQuadBounds(byte3 start, byte3 axis1, byte3 axis2, int boxAxisSize1, int boxAxisSize2, int boneIndex, float3 bonePosition,  
    float3 animationPosition, float4 animationRotation) {  
    var squareSize = math.min(boxAxisSize1, boxAxisSize2);  
    var squareBounds = GetAxisBounds(start, axis1 + axis2, squareSize, bonePosition,  
       animationPosition, animationRotation);  
    var axis1Bounds1 = GetAxisBounds(start + axis2 * squareBounds.x, axis1, boxAxisSize1, bonePosition,  
       animationPosition, animationRotation);  
    var axis1Bounds2 = GetAxisBounds(start + axis2 * squareBounds.y, axis1, boxAxisSize1, bonePosition,  
       animationPosition, animationRotation);  
    var axis2Bounds1 = GetAxisBounds(start + axis1 * squareBounds.x, axis2, boxAxisSize2, bonePosition,  
       animationPosition, animationRotation);  
    var axis2Bounds2 = GetAxisBounds(start + axis1 * squareBounds.y, axis2, boxAxisSize2, bonePosition,  
       animationPosition, animationRotation);  
    return new int4(math.min(axis1Bounds1.x, axis1Bounds2.x), math.max(axis1Bounds1.y, axis1Bounds2.y),  
       math.min(axis2Bounds1.x, axis2Bounds2.x), math.max(axis2Bounds1.y, axis2Bounds2.y));  
}
```

Finally, we need to obtain the visibility boundaries in three-dimensional space. Again, we are relying on the ordered visibility of voxels. Therefore, using the functions above, we need to find the visibility boundaries on two pairs of opposite faces of the voxel grid.

![Cube borders](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/26-Cube-Borders.png)

And again, take the minimum and maximum values along all axes.

![Cube total area](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/27-Cube-Total-Area.png)

```c#
var quadBounds1 = GetQuadBounds(new byte3(0, 0, 0), byte3.right(), byte3.up(), _voxelsBox.Size.x,  
    _voxelsBox.Size.y, index, bonePosition, animationPosition, animationRotation);  
var quadBounds2 = GetQuadBounds(new byte3(0, 0, 0), byte3.forward(), byte3.up(), _voxelsBox.Size.z,  
    _voxelsBox.Size.y, index, bonePosition, animationPosition, animationRotation);  
var quadBounds3 = GetQuadBounds(new byte3(0, 0, (byte)_voxelsBox.Size.z), byte3.right(), byte3.up(), _voxelsBox.Size.z,  
    _voxelsBox.Size.y, index, bonePosition, animationPosition, animationRotation);  
var quadBounds4 = GetQuadBounds(new byte3((byte)_voxelsBox.Size.x, 0, 0), byte3.forward(), byte3.up(), _voxelsBox.Size.z,  
    _voxelsBox.Size.y, index, bonePosition, animationPosition, animationRotation);  
  
var minPoint = new int3(  
    math.min(quadBounds1.x, quadBounds3.x),  
    math.min(quadBounds1.z, math.min(quadBounds2.z, math.min(quadBounds3.z, quadBounds4.z))),  
    math.min(quadBounds2.x, quadBounds4.x));  
  
var maxPoint = new int3(  
    math.max(quadBounds1.y, quadBounds3.y),  
    math.max(quadBounds1.w, math.max(quadBounds2.w, math.max(quadBounds3.w, quadBounds4.w))),  
    math.max(quadBounds2.y, quadBounds4.y));  
  
_visibilityBounds[index] = new VoxelsBounds(minPoint, maxPoint);
```

That's it. After the exhaustive calculations for each of the six faces for each bone, we can check if the voxel falls within its bone's visibility boundaries during the final face culling.

The following gif will best demonstrate this algorithm:

![Back-face culling demo](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/28-Back-face-culling-demo.gif)

It's time for testing. It's important to note that up to this point, we've been conducting purely static culling, which isn't influenced by any external factors, but rather depends solely on the model's structure, which remains unchanged at this stage. However, now we're introducing dynamic culling, which ideally should occur every frame. As a result, I've conducted two separate tests:

1. Back-face culling every frame during update.
2. Back-face culling only at start.

With a static camera, we won't notice any difference. However, in the case of a moving camera, in the second scenario, if the camera's position is significantly different from the initial position, we may not see some voxels that should be visible.

I did this because I wanted to gauge how much performance improvement we would achieve with this geometry simplification. It will also serve as a performance benchmark that I'll aim to approach in the subsequent optimizations of calculations for each frame.

Calculating the precise number of visible faces and vertices is now not so straightforward since they depend on the camera's position. We can only provide a rough estimate:

- In the best case scenario, the camera is looking at only one face of the cube:
	1. 255\*255 = 65025 quads.
	2. 65025\*4 = 260100 vertices.
- In the worst case scenario, the camera is looking at three faces of the cube simultaneously.
	1. 255\*255\*3 = 195075 quads.
	2. 195075\*4 = 780300 vertices.

For comparison, the table includes the results of the test in which we statically culled only the inner voxels.

|Test Name|PlayerLoop (µs)|GfxDeviceMetal.WaitForLastPresent (µs)|
|---|---|---|
|Quads Cull InnerSides (2197)|297.14|0.45|
|Quads Cull Backface (2197)|277.68|0.20|
|Quads Cull Backface Update (2197)|293.74|0.29|
|Quads Cull InnerSides (16900)|287.25|0.37|
|Quads Cull Backface (16900)|231.68|0.29|
|Quads Cull Backface Update (16900)|293.22|0.29|
|Quads Cull InnerSides (130050)|285|0.37|
|Quads Cull Backface (130050)|283.41|0.29|
|Quads Cull Backface Update (130050)|301.10|0.33|
|Quads Cull InnerSides (1020100)|290.22|0.41|
|Quads Cull Backface (1020100)|288.12|0.37|
|Quads Cull Backface Update (1020100)|410.29|0.37|
|Quads Cull InnerSides (16581375)|572.16|316.6|
|Quads Cull Backface (16581375)|288.67|0.23|
|Quads Cull Backface Update (16581375)|2097.93|0.29|

We have **certainly** eased the GPU's workload. In the ideal scenario without updating every frame, we've also reduced the frame time by 2 times. However, with calculations in the update, the frame time has increased by 4 times. Was it all for nothing?

## Rescuing the situation

Not so fast. We won't give up that easily. We haven't even tested it on mobile devices yet, which usually have much weaker GPUs, and test results can vary significantly there. Furthermore, we still have room for optimizations.

Up until now, I was trying to support all the rendering variants described above to enable running tests simultaneously and get a comprehensive view. However, it's now time to streamline and fully focus on the last variant. [This commit](https://github.com/utkaka/unity-instanced-voxels/commit/136e4f2977e43b7ebed814a75593cddeec760558) is the final one where you can run all the renderer tests yourself.

As seen from the tests, dynamic culling has significantly increased CPU workload. Therefore, it is necessary to investigate which specific calculations are the most resource-intensive. It's time to check the Profiler:

![Profiler 1](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/29-Profiler.png)

![Profiler 2](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/30-Profiler.png)

Contrary to my concerns, the issue does not lie in the calculation of visibility boundaries. Binary search is indeed a highly efficient algorithm, and there is usually no need to worry about it. What troubled me was the frequency of its execution. However, the problem lies in the final job that populates arrays sent to the GPU for rendering, particularly in these lines:

```c#
_positions.AddNoResize(_inputPositions[index]);  
_colors.AddNoResize(_inputColors[index]);  
_bones.AddNoResize(bone);
```

Because we are using _**NativeList**_ here, we cannot utilize _**IJobParallelFor**_ and are restricted to _**IJobFor**_. While it is technically possible to employ _**NativeList.ParallelWriter**_, attempting to fill three lists simultaneously in a single job proved to be a poor idea. The order of elements in these arrays did not align. Furthermore, parallel writing to a _**NativeList**_ is not consistently faster than sequential writing. Based on my experience, it is often the opposite.

The second problem is that creating **ComputeBuffers** from these lists and passing them to the GPU is also not a very inexpensive operation.

Here, it's worth explaining where these three arrays came from. Initially, our voxels were intended for use on mobile devices, which are often severely limited in GPU capabilities, as I mentioned earlier. Therefore, I wanted to precalculate the position and color of each voxel on the CPU to offload the GPU as much as possible, allowing us to include as much geometry as possible.

During the voxelization process, we recorded the following information for each voxel:

1. byte3 for the voxel coordinates in the model grid.
2. byte3 for the sRGB color of the voxel.

Whereas for rendering, we need::

1. float3 for the voxel's world position, determined by the following formula:
```c#
_positions[index] = new float3(bytePosition.x, bytePosition.y, bytePosition.z) * _voxelSize +_startPosition;
```
2. float3 for the voxel's color (although using half3 would have been sufficient, it's unlikely that this would have significantly altered the situation). Due to our serialization of the sRGB color, and given that our project employs a linear color space, we need to convert the color using the following formula:
```c#
public static float3 GammaToLinearSpace(byte3 sRGB) {  
    var sRGBFloat = new float3(sRGB.x, sRGB.y, sRGB.z) / 255.0f;  
    // Approximate version from http://chilliant.blogspot.com.au/2012/08/srgb-approximations-for-hlsl.html?m=1  
    return sRGBFloat * (sRGBFloat * (sRGBFloat * 0.305306011f + 0.682171111f) + 0.012522878f);  
}
```

Therefore, I initially decided to perform these calculations on the CPU during the model's initialization, and subsequently pass the ready-made values for rendering. Consequently, for each voxel, we store and transmit 28 bytes:

1. float3 for the position = 12 bytes.
2. float3 for the color = 12 bytes.
3. uint for the bone index = 4 bytes.

Well then, let's try to conduct all the calculations in the shader, thereby reducing the volume of data transferred. In the [official documentation](https://docs.unity3d.com/Manual/SL-DataTypesAndPrecision.html), it is stated that: ***"Direct3D 11, OpenGL ES 3, Metal, and other modern platforms have proper support for integer data types, so using bit shifts and bit masking works as expected."***

We are tasked with passing byte3 coordinates, byte3 color, and a single byte for the bone index to the shader. However, we can't directly pass bytes to the shader. Therefore, we need to devise a method to pack them into a different type that is compatible with shader transmission. Since we had no intention of supporting OpenGL ES 2.0, it's relatively safe to rely on integers and bitwise operations. We'll pack the bone index and coordinates into one integer (4 bytes) and the color into another.

To achieve this, we will create a structure in which all the necessary bytes are packed into two ints:

```c#
public struct ShaderVoxel {  
    public readonly int PositionBone;  
    public readonly int Color;  
  
    public int GetBone() => PositionBone & 255;  
  
    public byte3 GetPosition() => new((byte) ((PositionBone & 65280) >> 8), (byte) ((PositionBone & 16711680) >> 16),  
       (byte) ((PositionBone & 4278190080) >> 24));  
  
    public ShaderVoxel(byte3 position, byte bone, byte3 color) {  
       PositionBone = bone | position.x << 8 | position.y << 16 | position.z << 24;  
       Color = color.x | color.y << 8 | color.z << 16;  
    }  
}
```

Consequently, in the culling job, we will now only populate a single array with these structures. The remaining calculations will then be performed in the shader:

```c
struct Voxel {  
    uint position_bone;  
    uint color;  
};

float3 voxel_position;  
int voxel_bone;  
half4 voxel_color;

void configure_procedural () {  
    #if defined(UNITY_PROCEDURAL_INSTANCING_ENABLED)  
    Voxel voxel = voxels_buffer[unity_InstanceID];  
    voxel_position = float3((voxel.position_bone & 65280u) >> 8, (voxel.position_bone & 16711680u) >> 16, (voxel.position_bone & 4278190080u) >> 24);    voxel_bone = voxel.position_bone & 255u;    half3 input_color = half3(half(voxel.color & 255u), half((voxel.color & 65280u) >> 8), half((voxel.color & 16711680u) >> 16)) / half(255);  
    // Approximate version from http://chilliant.blogspot.com.au/2012/08/srgb-approximations-for-hlsl.html?m=1  
    input_color = input_color * (input_color * (input_color * half(0.305306011) + half(0.682171111)) + half(0.012522878));  
    voxel_color = half4(input_color.x, input_color.y, input_color.z, half(1));  
    #endif  
}
```

A quick test:

|Test Name|PlayerLoop (µs)|GfxDeviceMetal.WaitForLastPresent (µs)|
|---|---|---|
|Quads Cull InnerSides (16581375)|567.81|290.5|
|Quads Cull Backface Update (16581375)|559.08|0.31|

Yes, that's much better. The total frame time in both scenarios has become roughly the same, with a significantly reduced GPU workload. Let's take another look at the profiler:

![Profiler 3](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/31-Profiler.png)

One might wonder: if the total frame time is roughly the same, what is the purpose of all these additional manipulations? There are two reasons for this:

1. All the tests were conducted on a MacBook, and the results may vary on mobile devices.
2. Actually, these two rendering approaches are not operating under equal conditions. In the first approach, voxels are transferred to the GPU just once during model initialization, whereas in the second approach, this happens every frame. There are no visual differences until we implement voxel destruction. But then, the second approach will consistently update the voxel states after each destruction every frame, while the first will still display the original model. Consequently, we will need to integrate a similar logic for updating the voxel buffer into the first approach. At that point, the first approach will almost certainly be less efficient.

However, is it necessary to update the voxel buffer every frame? The visibility of voxels depends on the relative position of the camera and the model. It's unlikely that the camera will move so quickly as to change the visibility boundaries of the voxels in every frame.

Therefore, after calculating the visibility boundaries, we will implement an additional simple check: have they changed since the previous frame? If not, then updating the buffer for that face can be skipped.

We will conduct the final measurements on the [ZTE Blade A51](https://www.gsmarena.com/zte_blade_a51-11239.php) smartphone. The test case involving 16 million voxels had to be removed because the application was crashing on the device due to memory issues. We'll investigate this matter later. We'll either conclude that so many voxels aren't necessary on mobile devices or find a way to optimize memory usage. Instead, we've added a test with a voxelized spider comprising 1222324 voxels, which is much more representative of the actual models used in games than a cube.

|Test Name|PlayerLoop (µs)|Gfx.WaitForPresentOnGfxThread (µs)|
|---|---|---|
|Quads Cull InnerSides (Cube_1020100)|16401.82|12133.82|
|Quads Cull Backface Update (Cube_1020100)|16577.44|11820.25|
|Quads Cull InnerSides (Spider_1222324)|49485.63|45207.67|
|Quads Cull Backface Update (Spider_1222324)|32567.51|26292.99|

Despite the fact that the 'Cull InnerSides' option does not update the voxel state, as mentioned earlier, it still noticeably lags behind the variant with back-face culling.
## Result

The article turned out to be much more extensive than I had originally planned. Therefore, I would like to summarize the entire process of how we ultimately render voxels.

Principal Components:

1. [InstancedQuadsRenderer](https://github.com/utkaka/unity-instanced-voxels/blob/391337a921bb0acff83364237c0f27e2ca6753ba/Assets/Scripts/com.utkaka.InstancedVoxels/Runtime/Rendering/InstancedQuad/InstancedQuadsRenderer.cs) is the primary rendering class. It initializes the model, updates the animation state, calculates the voxel visibility boundaries, and handles the rendering of the faces.
2. [QuadRenderer](https://github.com/utkaka/unity-instanced-voxels/blob/391337a921bb0acff83364237c0f27e2ca6753ba/Assets/Scripts/com.utkaka.InstancedVoxels/Runtime/Rendering/InstancedQuad/QuadRenderer.cs) is responsible for the rendering of a specific face. It carries out the final selection of voxels for which the face is visible and directly invokes _**Graphics.DrawMeshInstancedProcedural**_.
3. [SetupRuntimeVoxelsJob](https://github.com/utkaka/unity-instanced-voxels/blob/391337a921bb0acff83364237c0f27e2ca6753ba/Assets/Scripts/com.utkaka.InstancedVoxels/Runtime/Rendering/Jobs/SetupRuntimeVoxelsJob.cs) - Initial model setup. It initializes [ShaderVoxel](https://github.com/utkaka/unity-instanced-voxels/blob/391337a921bb0acff83364237c0f27e2ca6753ba/Assets/Scripts/com.utkaka.InstancedVoxels/Runtime/Rendering/ShaderVoxel.cs) for each serialized voxel and sets up the voxel's neighbor mask.
4. [MaskSameBoneJob](https://github.com/utkaka/unity-instanced-voxels/blob/391337a921bb0acff83364237c0f27e2ca6753ba/Assets/Scripts/com.utkaka.InstancedVoxels/Runtime/Rendering/Jobs/MaskSameBoneJob.cs) - Computes masks for neighboring voxels' bones.
5. [MaskVoxelSidesJob](https://github.com/utkaka/unity-instanced-voxels/blob/391337a921bb0acff83364237c0f27e2ca6753ba/Assets/Scripts/com.utkaka.InstancedVoxels/Runtime/Rendering/Jobs/MaskVoxelSidesJob.cs) - Calculates masks for neighboring voxels' presence.
6. [CullInnerVoxelsJob](https://github.com/utkaka/unity-instanced-voxels/blob/391337a921bb0acff83364237c0f27e2ca6753ba/Assets/Scripts/com.utkaka.InstancedVoxels/Runtime/Rendering/InstancedQuad/CullInnerVoxelsJob.cs) - Culls inner voxels to reduce the workload of subsequent jobs.
7. [CalculateVisibilityBoundsJob](https://github.com/utkaka/unity-instanced-voxels/blob/391337a921bb0acff83364237c0f27e2ca6753ba/Assets/Scripts/com.utkaka.InstancedVoxels/Runtime/Rendering/InstancedQuad/CalculateVisibilityBoundsJob.cs) - Calculation of visibility boundaries for each face of every bone.
8. [CheckVisibilityBoundsChangedJob](https://github.com/utkaka/unity-instanced-voxels/blob/391337a921bb0acff83364237c0f27e2ca6753ba/Assets/Scripts/com.utkaka.InstancedVoxels/Runtime/Rendering/InstancedQuad/CheckVisibilityBoundsChangedJob.cs) - Checks if the visibility boundaries of the face have changed for any of the bones.
9. [CullInvisibleSidesIndicesJob](https://github.com/utkaka/unity-instanced-voxels/blob/391337a921bb0acff83364237c0f27e2ca6753ba/Assets/Scripts/com.utkaka.InstancedVoxels/Runtime/Rendering/InstancedQuad/CullInvisibleSidesIndicesJob.cs) - Voxels culling based on the masks of neighboring voxels for each face.
10. [CullBackfaceJob](https://github.com/utkaka/unity-instanced-voxels/blob/391337a921bb0acff83364237c0f27e2ca6753ba/Assets/Scripts/com.utkaka.InstancedVoxels/Runtime/Rendering/InstancedQuad/CullBackfaceJob.cs) - Final voxel culling based on visibility boundaries and the population of an array that will be transmitted to the GPU for rendering.

![UML Start](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/32-UML-Start.png)

![UML Update](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/33-UML-Update.png)

Of course, it didn't go without unpleasant surprises. At the very last stage of testing, I ran the application on the [Xiaomi Redmi 8A](https://www.gsmarena.com/xiaomi_redmi_8a-9897.php) and discovered some unpleasant artifacts:

![Artifacts](https://raw.githubusercontent.com/utkaka/blog/main/_posts/2024-01-22-voxel-rendering-in-unity-part-3/34-Artifacts.png)

The problem is clearly somewhere on the shader side, but I couldn't quickly pinpoint its cause. I really dislike bugs that only occur on specific devices and involve shaders. For now, we'll just ignore it and try to investigate later.

In the next article, we will focus on voxel destruction, teach the voxel model to move around the scene, and perhaps consider how to efficiently render multiple models simultaneously. Stay tuned.