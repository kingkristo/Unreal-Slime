# _Slime For Unreal Engine By Alex Christo_

## Features

- Create slime blob characters, which will warp and "ooze" in response to objects around them.
- Slime trails behind slime characters.
- Objects which can be "slimed" by a slime character.
- Make a variety of slimes by modifying the exposed material parameters.
- Extra effects such as bubbles and jiggle physics.
- A cell shader to give your slime scene a more stylized look.

## Installation
- Create a new unreal project or navigate to an existing one. _[NOTE: this plugin was tested and verified using ver 4.25.1, performance on other versions cannot be guaranteed]_
- Create a folder called _Plugins_ within the main project folder (at the same level as content and config folders). If a Plugins folder already exists do not create a new one.
- Paste the _SlimeBlobs_ Folder into this Plugins folder.
- Open your project and navigate to _edit_ -> _plugins_ -> _installed_ and make sure the slime plugin is visible and enabled.
- Next steps will vary depending on if your project already has project settings you do not wish to overwrite.
>For new project / default project settings
- Navigate to _edit_ -> _project settings_
- Navigate to _Maps & Modes_ -> _Import_ and select the _MapsModesDefaults_ config file included with this plugin.
- Navigate to _Rendering_ -> _Import_ and select the _RenderingDefaults_ config file.
- Navigate to _Input_ -> _Import_ and select the _InputDefaults_ config file.
- Ensure that in the content browser you navigate to _View Options_ and Enable _Show Plugin Content_
- Restart unreal and open the "SlimeExampleMap" to check everything is working as expected.
>For existing project settings
- Navigate to _edit_ -> _project settings_
- Navigate to _Maps & Modes_ and change the default character to _BP_SlimePlayer_ (if player slime is desired).
- Navigate to _Rendering_ and enable _Generate Mesh Distance Fields_ (this will require a restart).
- Navigate to _Input_ and ensure that default unreal 3rd person bindings are present, or create your own (see SlimePlayer blueprint for which inputs are needed).
- Ensure that in the content browser you navigate to _View Options_ and Enable _Show Plugin Content_
- Restart unreal and open the "SlimeExampleMap" to check everything is working as expected.

## Usage
> Example Map

There are two maps provided with the plugin: one for testing and one for demo purposes. The _SlimeExampleMap_ level contains examples of slime characters, slimeable objects, slime trail effects and the cell shader. Play around in this level to get an idea of what this plugin is capable of. Use _WASD_ to move and _space_ to jump. You can view the blueprints for the example objects used in this scene to get an idea of how to create your own objects and characters using the blueprints and materials provided. 

> Slime Character

The _BP_SlimePlayer_ blueprint will demonstrate the various components required to create a good looking slime character.

- The body of the slime can be found in the mesh folder. It is just a sphere with a simple rig which makes it "jiggle" when it is moved. The slime body must have _affect distance field lighting_ disabled, having this enabled will cause the slime blob to try and react to its own distance field. _Render CustomDepth Pass_ must be enabled also in order for the slime to draw to the scene capture which drives "sliming" of the floor and slimable objects.

- A small collision capsule is placed at the center of the slime ball, this is the only thing that collides with objects in the world, so the slime will only be blocked if something collides with its center. This is a crude way of dealing with collisions but gives fairly convincing results as a starting point. Play around with the size/shape of this depending on how much you want your slime ball to be able to "absorb" objects around it.

- The Slime Player blueprint contains the logic responsible for "sliming" objects. This blueprint chunk must be added to any character you want to be able to interact with slimable objects. It checks for overlapping slimable objects and passes its position to them after unwrapping their texture (more details on this further down).

- A bubble emmitter creates bubbles at the center of the slime sphere. This can be found in the _effects_ folder and should be self explanatory. To change the look of bubbles (colour and glow) you can make a material instance of the _M_Bubbles_ material and apply this to the bubble mesh in the _Mesh_ folder.  

- The most important part of a slime is the _M_SLime_ Material. This is responsible for the oozing behaviour of the slime ball. Creating a material instance of this material will expose a number of parameters used to tweak the look and behaviour of the slime. These are self explanatory and should be enough for most users to create their desired effect. You can see an example of how this material can be used to create different effects in the example map. Those who want more control can modify the material blueprint itself, a brief overview of its implementation follows: 
- - A simple ray marching algorithm is responsible for the effect of the slime ball absorbing objects around it [1]. In brief detail, the slime ball is represented by a sphere signed distance function [2]. Rays are fired from the center to see if there is a nearby object generating a distance field (thus any object not generating distance fields will be ignored). If there is a nearby object a smooth union will be calculated between the sphere and the object, resulting in the sphere appearing to melt into the other object. Playing around with the ray marching parameters will lead to slightly different effects and can be used to reduce compute cost in exchange for some rough edges. The result of this ray march is used to mask out the opacity of the slime ball, making it appear to change shape with no actual displacement.
     
- - On top of ray marching, actual mesh displacement is used via modifying the _world position offset_. The slime will both displace towards objects and a simple grass wind is used to make the slime randomly "ripple". 

- - Frensel is used to give greater opacity at the edge of the slime ball and create refraction on objects viewed through the slime.

- - Offsetting the slime texture from the camera creates a parallax effect, this gives an illusion of depth to the slime body. [3]
  
- - Time is plugged into a sin wave which drives a lerp between different colours applied to the slime texture (which can be replaced by the user) as well as to create a rippling effect on the edge of the slime where it is touching another object.
  
> Slimable Objects

The blueprint _BP_SimableObject_ should be used for any object that you want a slime blob to be able to cover in slime. When set up correctly a slime blob should be able to partially cover an object and leave slime behind on the area where it overlapped. New slimable objects should be made by creating a child of this base blueprint.

- The static mesh can be replaced to change the object. _[NOTE: Due to the fact that sliming works by unwrapping an objects texture, any mesh with repeating textures is not compatible and will not behave as expected]_

- The material of this static mesh should be an instance of _M_Slimable_Object_ (more on using this material later).

- The object collisions can be modified if you want the slime to be able to fully "absorb" the object or be blocked by it. Setting  _world dynamic_ collisions to block, will block slimes. Setting it to Overlap will allow a slime to pass through.

- Each slimable object has a scene capture component attached _[NOTE: This scene capture component must have its orientation locked]_. Each object creates dynamic render textures which are used to first unwrap the objects texture and then use the position provided by the slime character to mask out an area of this texture where it is intersected by the player. This is done via a sphere mask, the size of which is driven by the _size_ parameter of the _M_Slime_Object_Intersect_ material. Modifying this will change how large the "slimed" area will be when the slime ball comes into contact. It should be kept to roughly the same size as the slime sphere mesh itself.

> Slime Floor

The slime trail for the floor works in a similar way to slimable objects but without dynamic render textures. Instead the _BP_FloorTrailSceneCapture_ blueprint must be dragged into the scene and placed beneath the terrain. The Position of the player is written to a render texture which is used to mask out a slime trail on the floor [4]. This recording of the players path is offset with the player position, such that the render texture is always centered on the player. This means the player can go as far as they want and they will not stop leaving a trail. However this does mean that the trail will not stay permanently once the player goes out of range, and it will not reappear if they come back. The size and resolution of the render texture can be modified to give a larger area around the player, although this will be more expensive.

> Slimable Object Material

Both the floor and any slimable objects should have an instance of the _M_Slimable_Object_ material. Creating an instance of this material will expose a series of parameters which will allow for customization of both the slime trail itself and the material that slime is applied to. These should all be self explanatory upon exploration. 

- If a hit render texture is passed in from a slimable object this will be used to apply slime, otherwise it will use the render texture used for the slime trail on the floor. As such you can create crude slime effects on objects even if they are not slimable objects.

- The selected render texture used will mask out a slime trail. On any areas of the mesh which are facing the floor slime will "drip" downwards using simple wind displacement.

> Slime Decal Emitter

The slime decal emitter found in the _effects_ folder offers an alternative method of leaving a slime trail on objects. Unreal does not explicitly support decal emitters but one has effectively been created here. The emitter drops cubes which have the _M_SlimeDecal_ material applied. This material projects a slime texture onto surrounding surfaces, giving the illusion of a decal style stamp on the geometry of the world [5]. The texture is projected in an axis, while removing smearing in the other two axis, repeated for all 3 possible axis. This results in a clean decal application with no smearing but is a somewhat expensive operation. An example of this can be found in the example map.

- The benefit of this is that it can easily work with any scene geometry with no need for slimable objects or render textures. However, it looks worse visually due to a lack of control over effects such as roughness and translucency. In addition, the projection is a little expensive meaning that unless the trail disappears relatively quickly slowdown will occur. As with other materials the look can be modified by creating an instance.


> Cell Shader

The final addition to this plugin is a cell shader [6]. While not directly related to the other slime assets it can be applied to a scene to give it a more stylized cartoony look, and gives the slime an amusing aesthetic. 

- The cell shader has two main functions. It bands scene colour into 8 "buckets" and multiplies this by the flat scene colour. This gives hard edges between areas in different light levels rather than a smooth transition. It also adds lines around the edges of objects by detecting changes in scene depth. Together this gives a cartoon style effect.

- This cell shader can be simply used by applying the material to a post process volume. For an example of this see the example map.

## Testing

This project was developed in accordance with TDD principals as much as possible, although some aspects did not lend themselves to this style.

- The basic functionality of the tool was mapped out on paper first, then as implementation details became clearer unit tests were written for the main blueprint functionality. These tests can be found in the level blueprint of the _TestMap_ level. In order to run these tests the _Functional Testing Editor_ must be enabled from the plugin menu. Upon restarting the editor navigate to _Window_ -> _Test Automation_ and enable the 6 tests listed under _Project_. Click _Start Tests_ to run the tests. _[NOTE: Test number 4 will occasionally fail incorrectly, but will work upon a retest. Additionally a warning will often state that physics is not enabled for the slime ball object, even though this is not the case upon inspection. The cause of both of these phenomena is unclear]_. 

- Much of this project is based around material creation, unreal does not support unit testing of these and due to their visual nature a simpler method of testing was employed. Upon creation of each new feature a video of it was taken for reference, these were then compared with later iterations to check for unwanted inconsistency. A sample recording is included with this project to ensure that the plugin is behaving as intended.

## Refrences

[1] Elalaoui, M., 2022. UE4 Tutorial - Blob with Raymarching. [video] Available at: <https://www.youtube.com/watch?v=grmZ0I5-CgA&t=3341s> [Accessed 23 May 2022].

[2] Quilez, I., 2022. Inigo Quilez. [online] Iquilezles.org. Available at: <https://iquilezles.org/articles/distfunctions/> [Accessed 23 May 2022].

[3] Cloward, B., 2022. Volumetric Ice Shader - UE4 Materials 101 - Episode 11. [video] Available at: <https://www.youtube.com/watch?v=X5WASspig3g&ab_channel=BenCloward> [Accessed 23 May 2022].

[4] Wenderlich, R., 2022. Creating Snow Trails in Unreal Engine 4. [online] raywenderlich.com. Available at: <https://www.raywenderlich.com/5760-creating-snow-trails-in-unreal-engine-4> [Accessed 23 May 2022].

[5] Knowledge, R., water?, H. and →, n., 2020. [Niagara 4.25] Particle Decals Mini Tutorial. [online] Real Time VFX. Available at: <https://realtimevfx.com/t/niagara-4-25-particle-decals-mini-tutorial/13177> [Accessed 23 May 2022].

[6] Cell shader adapted from one created for Slug Game Group Project. 2022. [Game] Directed and created by Alex Christo.



