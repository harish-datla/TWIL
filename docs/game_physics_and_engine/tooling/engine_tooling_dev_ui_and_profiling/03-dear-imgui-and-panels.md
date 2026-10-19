# Dear ImGui & panels

You want a slider for the sun angle and a button that reloads shaders, without writing a UI framework first. Dear ImGui "outputs optimized vertex buffers that you can render anytime in your 3D-pipeline-enabled application"[4]. It produces draw data, and you render it with your own backend. The architecture is three layers: a platform backend (window/input: GLFW, SDL), the core (API-agnostic), and a renderer backend (Vulkan, D3D, Metal). ImGui's draw data is just another set of draw calls in your command buffer.

Debug panels are sliders bound to live variables, an in-game console, and entity inspectors (iterate an entity's components, draw a widget per field):

Worked example

In the panel below, SliderFloat("Sun angle", &sunAngle, 0.0f, 3.14159f) maps the slider's track onto 0 to π radians. Drag the handle a quarter of the way and ImGui writes 3.14159 / 4 = 0.785 (45°) into sunAngle during that frame's UI pass, so the lighting code sees the new angle the next time it reads the variable. No event, no callback, no copy to sync.
A debug panel in the frame loop
C++ · Dear ImGui
Rust · egui

ImGui_ImplVulkan_NewFrame();                   // renderer backend first,
ImGui_ImplGlfw_NewFrame();                     // then platform backend (feeds input, display size)
ImGui::NewFrame();                              // begin the UI frame
ImGui::Begin("Debug");                          // a panel
ImGui::SliderFloat("Sun angle", &sunAngle, 0.0f, 3.14159f);  // bound to the live variable
ImGui::Checkbox("Wireframe", &showWireframe);   // edits write through now
if (ImGui::Button("Reload shaders"))             // the if(Button()) action model
    reloadShaders();                          // fires this frame, no callback to register
ImGui::Text("Frame: %.2f ms", frameTimeMs);    // a watch value
ImGui::End();

ImGui::Render();                               // build the vertex/index buffers (ImDrawData)
ImGui_ImplVulkan_RenderDrawData(ImGui::GetDrawData(), cmd);  // you draw it, in your render pass

