---
layout: post
title: "Physical based rendering shader"
author: "Shtille"
categories: journal
tags: [thoughts,GLSL]
image: greasy-pan-2-preview.png
---

Physical based rendering has become a common lighting model for decent engines like Unreal Engine 4, Unity, etc. I've found several implementations on the web. And made a simple application based on Shtille Engine framework which allows to test its visual appearance.
The PBR material should be UE4-like (consist of 4 textures: albedo map, normal map, roughness map and metallic map).
There's a cool site with free PBR textures: https://freepbr.com/
I've decided to test some metal material, this is how it looks like:

<img src="{{ '/assets/img/greasy-pan-2-preview.png' | relative_url }}">

 So I've got the reference image, and now I'm able to start testing different approaches. 
Here's the results:

<img src="{{ '/assets/img/SS.2017.12.18.00.35.18.jpg' | relative_url }}">
<img src="{{ '/assets/img/SS.2017.12.18.01.24.34.jpg' | relative_url }}">
<img src="{{ '/assets/img/SS.2017.12.18.23.19.39.jpg' | relative_url }}">
<img src="{{ '/assets/img/SS.2017.12.24.17.09.06.jpg' | relative_url }}">

The third result is taken from Khronos PBR demo, it shows the best result.
Then I started to test mipmapped cubemap. It is used for realistic model.

```glsl
vec3 diffuseLight = SrgbToLinear(texture(u_diffuse_env_sampler, n)).rgb;
vec3 specularLight = SrgbToLinear(textureLod(u_specular_env_sampler, reflection, lod)).rgb;
```

But the lower mipmap level is, the bigger becomes artefacts on the edges. This way I concluded two things:

- Diffuse environment map is irradiance map
- Specular environment map is prefiltered map

I found the way to generate both of these maps, based on some cubemap texture. Here's the result with proper maps:

<img src="{{ '/assets/img/SS.2017.12.24.15.15.52' | relative_url }}">

As you may see, it's rather similar to the reference image.
This is the result I've decided to end with.