# The Scene Graph

Objects rarely live directly in world space. A character owns a sword, and rotating the character must rotate the sword automatically. A scene graph stores local transforms in a parent/child hierarchy and computes world transforms top-down:

```text
world = parent.world · local
```

The parent must be computed before each child. Reversing the factors applies the parent’s offset after the child’s rotation and makes the child orbit the world origin instead of its parent.

An orbit example: a sun at `(4,0)` and a planet locally two units along X at a 90° angle should produce `(4,2)`. The wrong order rotates the sun’s offset too and produces `(0,6)`.

Use dirty flags. Recompute a node when its local transform or an ancestor moved; leave clean subtrees alone. In a seven-node graph where one planet and its moon move, dirty propagation recomputes two nodes instead of all seven. glTF node trees map naturally to this traversal.
