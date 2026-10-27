# What's next

In this series the next page is Spatial Partitioning, which builds the grids and trees behind the broad phase in §5. Beyond rigid bodies, the directions physics engines take next:

    Position-Based Dynamics (PBD/XPBD). Müller et al. 2006[7]. Solves constraints by directly moving positions instead of computing impulses, which makes cloth and soft bodies easy to keep stable. Extended PBD (XPBD, Macklin et al. 2016) adds compliance so stiffness no longer depends on the iteration count and timestep; the PhysX documentation cites it to justify the approximations in its own solvers[12].
    GPU physics. PhysX runs both of its rigid-body solvers on the GPU as well as the CPU[12], and NVIDIA's Flex did unified particle-based simulation. GPU pipelines scale to very large particle and body counts, but kernel-launch and transfer overheads make them a poor fit for small scenes.
    Deformable bodies. The Finite Element Method (FEM) for volumetric deformation, fracture, bending, and tearing. Real-time use depends on coarse tetrahedral meshes and a small number of simulated objects.
    MPM (Material Point Method). Hybrid particle-grid method for snow, mud, sand, and granular materials. Disney used MPM for snow in Frozen (Stomakhin et al. 2013). Real-time MPM is an active research area.
    Fluid simulation. SPH (Smoothed Particle Hydrodynamics) for Lagrangian fluids, and FLIP/APIC for hybrid particle-grid methods. FLIP is the usual choice for offline VFX liquids (Houdini's FLIP solver); games mostly use cheaper height-field and wave models for water surfaces.
