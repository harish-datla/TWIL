# Animation Blueprints

Animation Blueprints are graph-based assets for determining the pose of a Skeletal Mesh during gameplay. They support animation blending, bone control, and logic that evaluates the final pose each frame.

## Requirements

The project needs a Skeletal Mesh to use as the Animation Blueprint's target.

## Create and Assign an Animation Blueprint

Create one from the Content Browser using **Add (+) > Animation > Animation Blueprint**, then select its target Skeleton. Alternatively, right-click a Skeletal Mesh asset and choose **Create > Anim Blueprint**. The first workflow also allows choosing a template and parent class.

To use the asset on a character, select its Skeletal Mesh component, set **Animation Mode** to **Use Animation Blueprint**, and choose the asset in **Anim Class**.

## Further Reading

- [Animation Blueprint Editor](https://dev.epicgames.com/documentation/unreal-engine/animation-blueprint-editor-in-unreal-engine)
- [Graphing in Animation Blueprints](https://dev.epicgames.com/documentation/unreal-engine/graphing-in-animation-blueprints-in-unreal-engine)
- [State Machines](https://dev.epicgames.com/documentation/unreal-engine/state-machines-in-unreal-engine)
- [Animation Node Reference](https://dev.epicgames.com/documentation/unreal-engine/animation-blueprint-nodes-in-unreal-engine)
- [Animation Slots](https://dev.epicgames.com/documentation/unreal-engine/animation-slots-in-unreal-engine)
- [Sync Groups](https://dev.epicgames.com/documentation/unreal-engine/animation-sync-groups-in-unreal-engine)
- [Animation Blueprint Linking](https://dev.epicgames.com/documentation/unreal-engine/animation-blueprint-linking-in-unreal-engine)

Source: [Animation Blueprints](https://dev.epicgames.com/documentation/unreal-engine/animation-blueprints-in-unreal-engine)
