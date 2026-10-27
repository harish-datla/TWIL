# What each module gave

When something breaks in the arena, the first question is which module owns it. Each module the capstone draws on, and what it uses it for:
ModuleWhat the capstone uses it for
Memory ModelThe acquire/release ordering the audio ring and the job queues rely on
AllocatorsArena/pool for per-frame and entity allocation; the handle backing store
Job SystemsFork/join within a frame: per-enemy AI inside the sim step; culling and skinning-palette builds for the render
Lock-free QueuesThe SPSC ring to the audio thread; the job pool's bounded MPMC queue
3D MathVec/Mat/Quat, the MVP chain, lerp for positions and slerp for render rotations
Floating PointWhy this single-player sim needn't be bit-deterministic; the netcode precondition
Game LoopThe one accumulator, dt/alpha, interpolation, the spiral clamp, the spine
Platform & WindowThe window, the OS event pump pumpEvents drains, the surface the swapchain presents to
InputThe per-frame snapshot, the action map (move/look/attack/dodge)
File StreamingAsync reads of the level package, so loading never blocks the frame
Asset PipelineMounting the package; the glTF/texture/skin/navmesh load; prefabs; hot-reload
CompressionDecompressing the package at load, where decode speed matters more than ratio
GPU PipelineThe rasterization model every pass configures: clip space, the depth test, perspective-correct interpolation
Vulkan TriangleThe device, swapchain, command buffers, and the semaphores and fences every frame submits and presents through
TexturesMaterial textures: images, samplers, descriptors, layout transitions, the sRGB pipeline
Going 3DPerspective camera, depth, glTF meshes, the scene graph (world = parent · local)
PBRThe GGX/Smith/Schlick BRDF the lighting pass's cookTorrance evaluates; HDR tonemapping
ShadowsThe directional shadow map + PCF, folded into the lighting pass
DeferredThe G-buffer, the full-screen lighting loop, the forward pass for transparency
AO/GIThe SSAO pass between geometry and lighting
Post/AATAA, then the bloom pyramid and tonemap, at the back of the frame
Skeletal AnimationSkinned characters: the matrix palette, vertex-shader skinning, clip slerp
Gameplay LayerGame objects, the event bus, generational handles, deferred destruction, prefabs, Lua
Behavior TreesEnemy decisions; the MoveTo leaf and the RUNNING/SUCCESS/FAILURE protocol
Navmesh & SteeringrequestPath/hasArrived, the funnel, seek/arrive, local avoidance
Graph TraversalA* over the navmesh dual graph
PhysicsGJK/SAT + the impulse solver; the shapes the character controller slides against
Spatial PartitioningBroad-phase pruning for the melee overlap query; frustum culling before the geometry pass
AudioThe mixer, voices, equal-power pan + distance attenuation, the hard-real-time callback
NetcodeThe "how it would slot in" section: the deterministic-sim precondition
ToolingThe ImGui overlay and the scoped-timer profiler watching the frame
Check Your UnderstandingQ1 of 2

The capstone runs physics, AI, and animation. How many accumulators drive the simulation, and what does the render do?
One accumulator drives a single fixed simulate(dt); physics, AI, and animation all run inside that one step, and the render is read-only, interpolating with alpha.
Each subsystem keeps its own accumulator so it can run at its own rate.
No accumulator; just simulate once per rendered frame.
0/2 correct

Wrong answers, and why: the loop owns one accumulator and the render is read-only/interpolated (independent per-subsystem clocks make every run timing-dependent; no accumulator couples the sim to the frame rate); and the AI stack is four distinct layers (steering alone gets stuck on walls; the behavior tree decides and requests, it doesn't path or move).
