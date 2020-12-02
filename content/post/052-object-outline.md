---
date: "2020-11-25T00:00:00Z"
image: /assets/images/posts/052/result.gif
title: Object Outlines
---

I have made multiple tutorials about outlines already, doing them with a [inverted hull]({{< ref "post/020-hull-outline" >}}), as a [postprocessing effect using the depth and normal buffers]({{< ref "post/019-postprocessing-outlines" >}}) or by [sampling neighboring pixels of a sprite]({{< ref "post/049-sprite-outlines" >}}), but I want to add another technique to those. It uses the same base idea of sampling neighboring pixels as the sprite based one, but can be applied to 3d models. It uses a postprocessing effect, yet can be applied to distinct objects you choose. Because of those properties this technique is mostly useful to show selected objects in 3d contexts.

All this works by rendering the objects we want outlined into a separate texture and creating outlines based on that.

![](/assets/images/posts/052/result.gif)

## Building the Selection Buffer

We start by using a [simple postprocessing effect like in previous tutorials]({{< ref "post/016-postprocessing-basics" >}}) with OnRenderImage. But this time I want to use the `CommandBuffer` API, solely because it has one function not present in the `Graphics` class that will make our lives easier.

We start by creating a new commandbuffer, then getting a tempoary rendertexture. Because we don't juggle with `RenderTexture` instances in commandbuffer land, we need to get a shader property id that we can fill the new rendertexture into first. To define the size and other parameters of the render texture we can just pass in the RenderTextureDescriptor of the source texture that holds all relevant information.

```glsl
//setup stuff
var commands = new CommandBuffer();
int objectBuffer = Shader.PropertyToID("_SelectionBuffer");
commands.GetTemporaryRT(objectBuffer, source.descriptor);
```

With this set up we can continue by setting that new texture as the render target and clearing it with a transparent black. Those steps work similarly to the API in `UnityEngine.Graphics`.
Then we render the object we want outlined into said texture, if it exists. To do this we used the function I mentioned before, `DrawRenderer`, as the name suggests it allows us to just draw a renderer into the current rendertarget. If you want to allow multiple outlined objects this is where you'd render all of them in a loop.

```glsl
//render selection buffer
commands.SetRenderTarget(selectionBuffer);
commands.ClearRenderTarget(true, true, Color.clear);
if (OutlinedObject != null)
{
  commands.DrawRenderer(OutlinedObject, WriteObject);
}
```

We haven't defined `OutlinedObject` and `WriteObject` yet. The first one is a public Renderer we can assign from the inspector, the other one a simple material we can assign the same way. I used a material using the unlit shader from the very first tutorials, but for this simple implementation almost any material does the trick.

We'll change this later when we write the shader, but for now we'll blit the selection buffer we just made into the destination render texture. After thats done we release the buffer again so unity can re-use it.

```glsl
//apply everything and clean up in commandbuffer
commands.Blit(selectionBuffer, destination);
commands.ReleaseTemporaryRT(selectionBuffer);
```

All of this means the command buffer is complete. To execute it immediately we pass it into `Graphics.ExecuteCommandBuffer` and after its done its job we call `Dispose` on it to signal we don't need it anymore.

An optimization I would recommend, but is over the scope of this tutorial, is to create one command buffer at the start and keep reusing it, using `.Clear` to fill it anew, but even only do that when something actually changes and not every frame.

```glsl
//apply everything and clean up in commandbuffer
commands.Blit(selectionBuffer, destination);
commands.ReleaseTemporaryRT(selectionBuffer);

//execute and clean up commandbuffer itself
Graphics.ExecuteCommandBuffer(commands);
commands.Dispose();
```

With this work done, a simple material as well as a renderer in the camera frustum applied and the game running we should now see the renderer of our choice on a black background. 

![](/assets/images/posts/052/BunnyBuffer.png)

## Actually calculating Outlines based on that

To now use the buffer we just created, all we have to to on the C# side is to not blit the buffer to the destination, but instead blit the source and use a new material to do so, this material will use a shader which puts the buffer back into the equation. The new material in this case comes from a new public field. The core of the shader is the same as in the previous tutorial about [sprite outlines]({{< ref "post/049-sprite-outlines" >}}), but this time applied to the buffer we just prepared.

A interresting side-effect of using commandbuffers like we do is that theres no need to pass the buffer into the shader, just creating a field with the same name as the property we created the id from is enough. When transferring the shader from the sprite outlines to here we can simplyfy it a bit by only correcting the sample distance for the aspect ratio, worldspace shenenigans arent really possible here. Also after getting the core plus the outline we simply remove the core by subtracting it (and taking the maximum with 0 to not get negative values in some areas). With that done we can interpolate from the source color to the outline color using the outline we just calculated like before.

```glsl
fixed4 frag (v2f i) : SV_Target
{
  //sample directions
  #define DIV_SQRT_2 0.70710678118
  float2 directions[8] = {float2(1, 0), float2(0, 1), float2(-1, 0), float2(0, -1),
    float2(DIV_SQRT_2, DIV_SQRT_2), float2(-DIV_SQRT_2, DIV_SQRT_2),
    float2(-DIV_SQRT_2, -DIV_SQRT_2), float2(DIV_SQRT_2, -DIV_SQRT_2)};

  float aspect = _ScreenParams.x * (_ScreenParams.w - 1); //width times 1/height
  float2 sampleDistance = float2(_OutlineWidth / aspect, _OutlineWidth);
  
  //generate outline
  float maxAlpha = 0;
  for(uint index = 0; index<8; index++){
    float2 sampleUV = i.uv + directions[index] * sampleDistance;
    maxAlpha = max(maxAlpha, tex2D(_SelectionBuffer, sampleUV).a);
  }
  //remove core
  float border = max(0, maxAlpha - tex2D(_SelectionBuffer, i.uv).a);
  
  // sample the texture
  fixed4 col = tex2D(_MainTex, i.uv);
  col = lerp(col, _OutlineColor, border);
  return col;
}
```

![](/assets/images/posts/052/result.png)

## Choosing Outlines

The last bit I want to add to this is a system that allows putting an outline on objects by clicking on them. To do that I put a polygon collider on each of the gameobjects with meshes, added a "Selectable" [tag](https://docs.unity3d.com/Manual/Tags.html) and then added a small raycast functionality to the update method of our postprocessing behaviour that, if it hits a gameobject with the tag, assigns that objects renderer to the OutlinedObject field we've been using for outline rendering so far.

```cs
void Update() {
  if (Input.GetMouseButtonDown(0)) {
    var ray = Camera.main.ScreenPointToRay(Input.mousePosition);
    bool hitSelectable = Physics.Raycast(ray, out var hit) && hit.transform.CompareTag("Selectable");
    if (hitSelectable) {
      OutlinedObject = hit.transform.GetComponent<Renderer>();
    } else {
      OutlinedObject = null;
    }
  }
}
```

![](/assets/images/posts/052/BunnyInspector.png)

![](/assets/images/posts/052/result.gif)

## Source

- <https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/052_Object_Outline/ObjectOutline.cs>

```cs
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Rendering;

public class ObjectOutline : MonoBehaviour
{
	public Renderer OutlinedObject;

	public Material WriteObject;
	public Material ApplyOutline;

	void Update()
	{
		if (Input.GetMouseButtonDown(0))
		{
			var ray = Camera.main.ScreenPointToRay(Input.mousePosition);
			bool hitSelectable = Physics.Raycast(ray, out var hit) && hit.transform.CompareTag("Selectable");
			if (hitSelectable) {
				OutlinedObject = hit.transform.GetComponent<Renderer>();
			} else {
				OutlinedObject = null;
			}
		}
	}

	void OnRenderImage(RenderTexture source, RenderTexture destination)
	{
		//setup stuff
		var commands = new CommandBuffer();
		int selectionBuffer = Shader.PropertyToID("_SelectionBuffer");
		commands.GetTemporaryRT(selectionBuffer, source.descriptor);
		//render selection buffer
		commands.SetRenderTarget(selectionBuffer);
		commands.ClearRenderTarget(true, true, Color.clear);
		if (OutlinedObject != null)
		{
			commands.DrawRenderer(OutlinedObject, WriteObject);
		}
		//apply everything and clean up in commandbuffer
		commands.Blit(source, destination, ApplyOutline);
		commands.ReleaseTemporaryRT(selectionBuffer);
		
		//execute and clean up commandbuffer itself
		Graphics.ExecuteCommandBuffer(commands);
		commands.Dispose();
	}
	
}
```

- <https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/052_Object_Outline/ApplyOutline.shader>

```glsl
Shader "Unlit/ApplyOutline"
{
    Properties
    {
        [HideInInspector]_MainTex ("Texture", 2D) = "white" {}
        _OutlineWidth ("OutlineWidth", Range(0, 1)) = 1
        _OutlineColor ("OutlineColor", Color) = (1, 1, 1, 1)
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" }
        LOD 100

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            // make fog work
            #pragma multi_compile_fog

            #include "UnityCG.cginc"

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct v2f
            {
                float2 uv : TEXCOORD0;
                UNITY_FOG_COORDS(1)
                float4 vertex : SV_POSITION;
            };
            
            sampler2D _MainTex;
            float4 _MainTex_ST;
            
            float _OutlineWidth;
            float4 _OutlineColor;
            sampler2D _SelectionBuffer;

            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                o.uv = TRANSFORM_TEX(v.uv, _MainTex);
                UNITY_TRANSFER_FOG(o,o.vertex);
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                //sample directions
                #define DIV_SQRT_2 0.70710678118
                float2 directions[8] = {float2(1, 0), float2(0, 1), float2(-1, 0), float2(0, -1),
                  float2(DIV_SQRT_2, DIV_SQRT_2), float2(-DIV_SQRT_2, DIV_SQRT_2),
                  float2(-DIV_SQRT_2, -DIV_SQRT_2), float2(DIV_SQRT_2, -DIV_SQRT_2)};

                float aspect = _ScreenParams.x * (_ScreenParams.w - 1); //width times 1/height
                float2 sampleDistance = float2(_OutlineWidth / aspect, _OutlineWidth);
                
                //generate outline
                float maxAlpha = 0;
                for(uint index = 0; index<8; index++){
                  float2 sampleUV = i.uv + directions[index] * sampleDistance;
                  maxAlpha = max(maxAlpha, tex2D(_SelectionBuffer, sampleUV).a);
                }
                //remove core
                float border = max(0, maxAlpha - tex2D(_SelectionBuffer, i.uv).a);
                
                // sample the texture
                fixed4 col = tex2D(_MainTex, i.uv);
                col = lerp(col, _OutlineColor, border);
                return col;
            }
            ENDCG
        }
    }
}
```