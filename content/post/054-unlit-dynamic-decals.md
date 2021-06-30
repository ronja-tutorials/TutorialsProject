---
date: "2021-06-30T00:00:00Z"
image: /assets/images/posts/054/SimpleResult.png
title: Unlit Dynamic Decals/Projection
---

Often when doing VFX you want stuff to stick to the floor. Even when that floor is uneven. Or decals to make existing geometry more interresting, or you want blob shadows on uneven floor, or some other use-cases in the same direciton.

{{< video src="/assets/images/posts/054/Result.mp4" width="300">}}
(note: I used the free "Nature Starter Kit 2" from the Unity Asset store throughout this tutorial)

Unity has the concept of a "projector" for that, but I have to admit I dont quite trust them. They work by finding all objects in the projector frustum and redrawing them, that means you quickly get a ton of drawcalls and overdraw (and I dont know how expensive the finding objects is in complex scenes). So what I tend to use instead is the depth buffer, reconstructing the worldspace and then object space position based on it and then using that as coordinates for whatever we want. I hope the concept will become clearer by the end of the tutorial. The first time I ran into this was the unity commandbuffer example where they use this tech to render into GBuffers before the deferred lighting pass, though out example will be simpler <https://docs.unity3d.com/Manual/GraphicsCommandBuffers.html>.

## Preparing the Data we need

Since this will be a simple unlit shader apart from the projection, we can start with just [a transparent shader]({{< ref "post/006-simple-transparency">}}). Though we dont need the UVs from the mesh itself, and we do need both the screen position to read from the depth buffer as well as a ray from the camera to the object to reconstruct the position(there are other ways, but this seemed easiest rn). I go deeper about screenspace coordinates and their implications in [my tutorial about them]({{< ref "post/039-screenspace-texture" >}}), for the ray I subtract the worldspace position by the camera worldspace position. As I explain in the tutorial, to unstretch the screenspace texture we need to divide by the w component of the vector.

```glsl
//the data thats passed from the vertex to the fragment shader and interpolated by the rasterizer
struct v2f{
	float4 position : SV_POSITION;
	float4 screenPos : TEXCOORD0;
	float3 ray : TEXCOORD1;
};
```

```glsl
//the vertex shader function
v2f vert(appdata v){
	v2f o;
	//convert the vertex positions from object space to clip space so they can be rendered correctly
	float3 worldPos = mul(unity_ObjectToWorld, v.vertex);
	o.position = UnityWorldToClipPos(worldPos);
	//calculate the ray between the camera to the vertex
	o.ray = worldPos - _WorldSpaceCameraPos;
	//calculate the screen position
	o.screenPos = ComputeScreenPos (o.position);
	return o;
}
```

```glsl
//the fragment shader function
fixed4 frag(v2f i) : SV_TARGET{
	//unstretch screenspace uv and get uvs from function
	float2 screenUv = i.screenPos.xy / i.screenPos.w;
	getProjectedObjectPos(screenUv, i.ray);
	//...
```

And with this we can get started using that data.

## Doing the Do

### Get Depth

I opted to put the code for the projection into its own function since it felt cleaner and is easier to copy between files/into a library include file. It takes the screen position as well as the ray we just created.

We start by sampling the depth texture, this is not the same as the depth buffer, its a separate texture thats generated if we signal to the camera that we want it to do that, many postprocessing effects already do that, but you might also have to do it yourself depending on your scene. I'm going to assume you have a depth texture, if not take a quick look into [my tutorial about postprocessing using that texture]({{< ref "post/017-postprocessing-depth#read-depth" >}}). Once thats enabled just adding `sampler2D_float _CameraDepthTexture;` as a uniform variable to our pass is enough for it to have access to that texture. Then in the function we use the `SAMPLE_DEPTH_TEXTURE` macro to read from the texture and `Linear01Depth (depth) * _ProjectionParams.z` to first get rid of the bias it uses for better encoding and then make it so its space reaches from 0 to the far clip plane, instead of 0 to 1.

```glsl
//get depth from depth texture
float depth = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, screenPos);
depth = Linear01Depth (depth) * _ProjectionParams.z;
```

### naive world reconstruction

With the depth taken care of we next reconstruct the world position by multiplying the normalized ray by the depth. Once we did that, getting the object space position is a matter of a simple matrix multiplication.

```glsl
float3 getProjectedObjectPos(float2 screenPos, float3 worldRay){
	//get depth from depth texture
	float depth = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, screenPos);
	depth = Linear01Depth (depth) * _ProjectionParams.z;
	//reconstruct world and object space positions
	float3 worldPos = _WorldSpaceCameraPos + normalize(worldRay) * depth;
	float3 objectPos =  mul (unity_WorldToObject, float4(worldPos,1)).xyz;
	return objectPos;
}
```

![](/assets/images/posts/054/FullIntersection.png)

### clipping stuff thats behind

That does already look like positions based on the depth buffer and object position, but it doesnt just shit the position where the cube is, but also behind where the cube is. Since we know the default unity cube we're using here is 1x1 units big, we know the coordinates inside the cube go from -0.5 to 0.5, so lets discard all pixels outside of that. The `clip` function discards all pixels in which it gets fed a value smaller than 0, if we give it values with multiple components, it discards pixels where any of them is below 0. So to solve our current problem we can subtract the absolute(to catch negative values) of the object space position from `0.5` (which will be interpreted as a vector where each component is `0.5`) and feed the result to the `clip` function.

```glsl
clip(0.5 - abs(objectPos));
```

![](/assets/images/posts/054/WrongWsPos.png)

### fixing worldspace reconstruction

Now we get a square on the ground thats closer to what we imagined, but the square doesnt quite seem to behave, towards the corner of the screen it seems to slip upwards. This is because we made a small mistake in the worldspace reconstruction, we assumed that the depth texture has the distance from the camera, but it doesnt, the depth is kinda parallel to the camera in a way (this is because how modern realtime graphics work with their matrix multiplications, but thats a talk for another day).

![](/assets/images/posts/054/DistanceExplanation.png)

So, what can we do about that? We could choose a different approach that sidesteps the problem, but what I chose to do instead is to take the dot product between the (normalized)ray and the camera forward direction. In the middle of the screen thats going to be 1 and it decreases as the ray points less and less towards the direction of the camera. And as you can see in [this](https://twitter.com/freyaholmer/status/1200807790580768768) wonderful visualisation by Freya Holmér the amount it becomes less is exactly how long one vector is projected onto the other, or for our case, the inverse of the length we want. So if we divide the ray by this dot product we'll end up with a vector thats 1 in the length of where the camera is looking. Longer than 1 because it'll also point in other directions, but thats what we need.

We can get the camera forwards vector by multiplying `(0, 0, 1, 0)` by the view matrix since the view matrix is built from the transform of the camera. But since all thats going on in that matrix multiplication is a bunch of multiplying by 1 or 0, we can simplify this by just taking the 3rd row out of the matrix and use `-UNITY_MATRIX_V[2].xyz` (I dont know why I had to add the minus here, but it makes it work).

```glsl
float3 getProjectedObjectPos(float2 screenPos, float3 worldRay){
	//get depth from depth texture
	float depth = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, screenPos);
	depth = Linear01Depth (depth) * _ProjectionParams.z;
	//get a ray thats 1 long on the axis from the camera away (because thats how depth is defined)
	worldRay = normalize(worldRay);
	//the 3rd row of the view matrix has the camera forward vector encoded, so a dot product with that will give the inverse distance in that direction
	worldRay /= dot(worldRay, -UNITY_MATRIX_V[2].xyz);
	//with that reconstruct world and object space positions
	float3 worldPos = _WorldSpaceCameraPos + worldRay * depth;
	float3 objectPos =  mul (unity_WorldToObject, float4(worldPos,1)).xyz;
	//discard pixels where any component is beyond +-0.5
	clip(0.5 - abs(objectPos));
	return objectPos;
}
```

![](/assets/images/posts/054/TrueOsCoordinates.png)

And with this all the hard work is done!

What I did next was subtract 0.5 from the position before returning it so we're in 0 to 1 space if we use the default unity cube. Since thats a nice space to be in when we work with textures. If you want to do cool stuff with signed distance fields instead you might not want this and preserve the coordinate center at the cube center.

```glsl
//get -0.5|0.5 space to 0|1 for nice texture stuff if thats what we want
objectPos += 0.5;
```

And in the `frag` function we can directly feed the x and z components (assuming y is up) of the output to the `tex2D` function as uv coordiantes.

```glsl
//the fragment shader function
fixed4 frag(v2f i) : SV_TARGET{
	//unstretch screenspace uv and get uvs from function
	float2 screenUv = i.screenPos.xy / i.screenPos.w;
	float2 uv = getProjectedObjectPos(screenUv, i.ray).xz;
	//read the texture color at the uv coordinate
	fixed4 col = tex2D(_MainTex, uv);
	//multiply the texture color and tint color
	col *= _Color;
	//return the final color to be drawn on screen
	return col;
}
```

![](/assets/images/posts/054/SimpleResult.png)

And the very last tweak that doesnt change the visuals, but makes the shader more robust is decreasing the material priority by a bit, so it doesnt take priority over transparent shaders that dont cling to the depth anyways. And disabling batching as that would mean the center of the batched object is at the world center, and we kinda need our center to be where the original object is. Both of those settings are in the Subshader Tags.

```cs
Tags{ "RenderType"="Transparent" "Queue"="Transparent-400" "DisableBatching"="True"}
```

## Ideas for Improvements

Right now if we put the camera inside the stencil object, the stencil disappears. We can avoid that by culling front faces instead of backfaces and disabling the zbuffer test, but this does come at the cost of more overdraw since you'll draw fragments behind walls that will all be discarded, a LOD system that switches out shaders to the always draw variant when the camera is very close is good here.

Another weakness is that right now the stencils affect *everything* that renders to the depth buffer, that includes dynamic characters, moving projectiles, opaque particles... My choice to fight that would be to use [stencil buffers]({{< ref "post/022-stencil-buffers" >}}) to choose one stencil value to either mask out the decals, or to only allow them at that value.

Also the tech might be a bit agressive about drawing onto stuff that is orthogonal to itself, drawing streaks as the texture is stretched, to fight that you can either [tell the camera to also render a normals texture]({{< ref "post/018-postprocessing-normal" >}}) and compare your "up" to the normals and fade away if theyre too different, or try to reconstruct the normals cheaply from the depth buffer using partial derivatives.

And then, those are only unlit decals, you can make decals with lighting using the same tech, but sadly not using surface shaders which means its some major effort, especially in the builtin render pipeline since it has multiple passes for multiple lights.

## Source

- <https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/054_Unlit_Decals/UnlitDynamicDecal.shader>

```glsl
Shader "Tutorial/054_UnlitDynamicDecal"{
	//show values to edit in inspector
	Properties{
		[HDR] _Color ("Tint", Color) = (0, 0, 0, 1)
		_MainTex ("Texture", 2D) = "white" {}
	}

	SubShader{
		//the material is completely transparent and is rendered before other transparent geometry by default (at 2500)
		Tags{ "RenderType"="Transparent" "Queue"="Transparent-400" "DisableBatching"="True"}

		//Blend via alpha
		Blend SrcAlpha OneMinusSrcAlpha
		//dont write to zbuffer because we have semitransparency
		ZWrite off

		Pass{
			CGPROGRAM

			//include useful shader functions
			#include "UnityCG.cginc"

			//define vertex and fragment shader functions
			#pragma vertex vert
			#pragma fragment frag

			//texture and transforms of the texture
			sampler2D _MainTex;
			float4 _MainTex_ST;

			//tint of the texture
			fixed4 _Color;

			//global texture that holds depth information
			sampler2D_float _CameraDepthTexture;

			//the mesh data thats read by the vertex shader
			struct appdata{
				float4 vertex : POSITION;
			};

			//the data thats passed from the vertex to the fragment shader and interpolated by the rasterizer
			struct v2f{
				float4 position : SV_POSITION;
				float4 screenPos : TEXCOORD0;
				float3 ray : TEXCOORD1;
			};

			//the vertex shader function
			v2f vert(appdata v){
				v2f o;
				//convert the vertex positions from object space to clip space so they can be rendered correctly
				float3 worldPos = mul(unity_ObjectToWorld, v.vertex);
				o.position = UnityWorldToClipPos(worldPos);
				//calculate the ray between the camera to the vertex
				o.ray = worldPos - _WorldSpaceCameraPos;
				//calculate the screen position
				o.screenPos = ComputeScreenPos (o.position);
				return o;
			}

			float3 getProjectedObjectPos(float2 screenPos, float3 worldRay){
				//get depth from depth texture
				float depth = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, screenPos);
				depth = Linear01Depth (depth) * _ProjectionParams.z;
				//get a ray thats 1 long on the axis from the camera away (because thats how depth is defined)
				worldRay = normalize(worldRay);
				//the 3rd row of the view matrix has the camera forward vector encoded, so a dot product with that will give the inverse distance in that direction
				worldRay /= dot(worldRay, -UNITY_MATRIX_V[2].xyz);
				//with that reconstruct world and object space positions
				float3 worldPos = _WorldSpaceCameraPos + worldRay * depth;
				float3 objectPos =  mul (unity_WorldToObject, float4(worldPos,1)).xyz;
				//discard pixels where any component is beyond +-0.5
				clip(0.5 - abs(objectPos));
				//get -0.5|0.5 space to 0|1 for nice texture stuff if thats what we want
				objectPos += 0.5;
				return objectPos;
			}

			//the fragment shader function
			fixed4 frag(v2f i) : SV_TARGET{
				//unstretch screenspace uv and get uvs from function
				float2 screenUv = i.screenPos.xy / i.screenPos.w;
				float2 uv = getProjectedObjectPos(screenUv, i.ray).xz;
			  //read the texture color at the uv coordinate
				fixed4 col = tex2D(_MainTex, uv);
				//multiply the texture color and tint color
				col *= _Color;
				//return the final color to be drawn on screen
				return col;
			}

			ENDCG
		}
	}
}
```