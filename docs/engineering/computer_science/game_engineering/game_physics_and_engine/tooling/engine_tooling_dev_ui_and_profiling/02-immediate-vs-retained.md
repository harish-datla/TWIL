# Immediate vs retained

You add a "wireframe" checkbox to a tools panel. Click it, and the box shows a tick, but the renderer still draws solid triangles, because the code that should copy the box's value into the renderer's flag never ran. That bug comes from one of the two ways to build a UI. Retained-mode keeps a persistent tree of widget objects you mutate through callbacks, and you must keep that tree in sync with your application's own state. Immediate-mode (IMGUI, a term coined by Casey Muratori; Dear ImGui, by Omar Cornut, is a widely used C++ library for it) re-issues the whole UI every frame straight from your live data, with no widget objects for the application to keep: the if (Button()) doThing() model[2].

Worked example

Retained: the checkbox is an object holding its own checked = true after the click, and a callback is supposed to set the renderer's showWireframe = true. Forget to register the callback and the two disagree for as long as the game runs. Immediate: every frame calls Checkbox("Wireframe", &showWireframe), which draws the box from the variable and writes the click straight into it. There's only one copy of the value, so the box shows what the renderer will use; at worst the drawn box is one frame (16.7 ms at 60 fps) behind a change some other code made to the flag.
It's about where UI state lives, not about being "faster"

In the words of a Hacker News write-up that the Dear ImGui wiki quotes, immediate mode is "a statement about the interface of the library, not the internals"[3]. Dear ImGui rebuilds its geometry every frame, which costs CPU; its win is programmer iteration speed and no UI state to keep in sync, not raw runtime performance. It doesn't "retain nothing" either: it caches layout, widget IDs and window state internally. And IMGUI is the paradigm; Dear ImGui is one implementation. The common industry pattern: immediate-mode for debug/tools UI, retained/custom for the shipping player HUD (Dear ImGui targets "content creation tools and visualization / debug tools (as opposed to UI for the average end-user)"[4]), though some games do ship it.

Toggle the modes and click. Retained keeps two stores you must sync (forget to, and they drift); immediate rebuilds the UI from one store, so there's no second copy to go stale:
retained
immediate
click the toggle
sync step
Pick a mode, then press click the toggle a few times; in retained mode, untick sync step and click again. Retained: a click updates the widget node, and a second step must update the app-data store. Turn off the sync step and the two drift apart: the widget shows one value and the data holds the other, the classic retained desync. Immediate: the UI is re-emitted each frame from one store, so there's no second copy to desync.
