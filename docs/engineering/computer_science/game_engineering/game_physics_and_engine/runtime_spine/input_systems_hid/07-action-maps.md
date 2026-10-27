# Action Maps


A player wants Jump on F instead of Space, and the same game has to work with a gamepad. If gameplay code checks for the Space key directly in dozens of places, both requests mean touching all of them. So don't let gameplay name keys. Let it ask "is Jump triggered?" and answer that through an action map, a table from physical inputs to named actions, scoped by context (menu, on foot, in a vehicle). This is what makes rebinding, multi-device support, and accessibility remapping possible.

Worked example

The on-foot map holds three rows: scancode 26 (W) → MoveForward, scancode 44 (Space) → Jump, and gamepad button South → Jump. The player presses South on the pad; the lookup finds Jump, and gameplay sees Jump without knowing a pad was involved. The player rebinds Jump to F: the Space row becomes scancode 9 (F), and no gameplay code changes.

Steam Input states the principle bluntly: in its native mode the game receives input "in the form of 'actions'" and "has no knowledge of what actual inputs are driving said actions, simply that the actions are happening"[11]. Unreal's Enhanced Input models it as Input Actions plus Mapping Contexts you push and pop, with per-binding modifiers and triggers[12]. Unity's Input System uses Action Maps with bindings and processors[13]. The minimal data shape is a tagged physical input, so one action can bind a key or a pad button:
Physical input → action
C++
Rust

enum class Action { MoveForward, Jump, Fire };

struct PhysicalInput {                // tagged so one action binds keyboard OR pad
    enum class Source { Scancode, GamepadButton } source;
    int code;                          // SDL_Scancode (a position) or a button
    bool operator==(const PhysicalInput&) const = default;
};
// One context; swap the whole map for menu/vehicle.
std::unordered_map<PhysicalInput, Action, Hash> bindings;


[Back to Input Systems (HID)](index.md)
