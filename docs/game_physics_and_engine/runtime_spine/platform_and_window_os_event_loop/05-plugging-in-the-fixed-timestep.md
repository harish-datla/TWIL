# Plugging in the Fixed Timestep


You've written a fixed-timestep loop for a GLFW build, and now the Rust port's handler gets called once per frame by winit instead. The accumulator from the Game Loop tutorial drops into both shapes. The algorithm is identical; only where the state lives and who calls the body change.

The same loop, two control-flow shapes (complete programs)
C++ · GLFW (poll)
Rust · winit (event-driven)

// A complete GLFW host: a window, the loop you own, a fixed-timestep
// accumulator, and the framebuffer size a renderer needs. Nothing is drawn
// yet; with no vsync to wait on, this loop spins as fast as the CPU allows.
#define GLFW_INCLUDE_NONE   // no OpenGL headers; a Vulkan build defines GLFW_INCLUDE_VULKAN instead
#include <GLFW/glfw3.h>
#include <cstdio>

struct Game {
    double accumulator = 0.0;
    long long steps = 0, frames = 0;
    int framebufferWidth = 0, framebufferHeight = 0;
    bool swapchainStale = false;   // set on resize; the renderer rebuilds before its next present
};

void update(Game& game, double dt) { (void)dt; ++game.steps; }          // the simulation, at a fixed dt
void render(Game& game, double alpha) { (void)alpha; ++game.frames; }   // draw, blended by alpha

int main() {
    if (!glfwInit()) return 1;
    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);   // no OpenGL context: Vulkan brings its own
    glfwWindowHint(GLFW_SCALE_TO_MONITOR, GLFW_TRUE);  // Windows, X11: grow the window with the display scale
    GLFWwindow* window = glfwCreateWindow(1280, 720, "Engine", nullptr, nullptr);
    if (!window) { glfwTerminate(); return 1; }

    Game game;
    glfwSetWindowUserPointer(window, &game);        // how a C callback finds the game again
    // Render at the framebuffer size, in pixels. On macOS and Wayland at 2x, a 1280x720 window
    // has a 2560x1440 framebuffer; on Windows and X11 GLFW's screen coordinates are pixels already.
    glfwGetFramebufferSize(window, &game.framebufferWidth, &game.framebufferHeight);
    glfwSetFramebufferSizeCallback(window, [](GLFWwindow* resized, int width, int height) {
        Game* resizedGame = static_cast<Game*>(glfwGetWindowUserPointer(resized));
        resizedGame->framebufferWidth = width;
        resizedGame->framebufferHeight = height;
        resizedGame->swapchainStale = true;
    });
    float scaleX = 1.0f, scaleY = 1.0f;
    glfwGetWindowContentScale(window, &scaleX, &scaleY);
    int windowWidth = 0, windowHeight = 0;
    glfwGetWindowSize(window, &windowWidth, &windowHeight);   // above 1280x720 on a scaled Windows or X11 display
    std::printf("window %dx%d (screen coordinates), framebuffer %dx%d, content scale %.2f\n",
                windowWidth, windowHeight, game.framebufferWidth, game.framebufferHeight, scaleX);

    const double dt = 1.0 / 60.0;
    double previous = glfwGetTime();
    while (!glfwWindowShouldClose(window)) {        // you own the loop
        glfwPollEvents();                           // handle what has arrived and return; a Win32 drag blocks in here (§8)
        double now = glfwGetTime();
        double frameTime = now - previous;
        previous = now;
        if (frameTime > 0.25) frameTime = 0.25;     // after a long stall, catch up at most 15 steps
        game.accumulator += frameTime;
        while (game.accumulator >= dt) { update(game, dt); game.accumulator -= dt; }
        render(game, game.accumulator / dt);
        if (now >= 2.0) glfwSetWindowShouldClose(window, GLFW_TRUE);   // demo: quit after 2 s
    }
    std::printf("%lld fixed steps in %.2f s; the loop ran %lld times\n", game.steps, glfwGetTime(), game.frames);
    glfwDestroyWindow(window);
    glfwTerminate();
}

In GLFW you call the loop body inside your while; in winit the body is a method (RedrawRequested) and its state lives on the struct, because winit calls you. The accumulator math is identical. Both programs quit after two seconds and print about 120 fixed steps, plus a loop count in the hundreds of thousands or millions: nothing is drawn yet, so nothing makes the loop wait. Presenting with vsync (the Vulkan module) or a frame cap (§6) is what paces it. The size callbacks mark the swapchain (the images the GPU draws into and the display shows) stale; §8 covers why.

Worked example

Declare the accumulator as a local inside the RedrawRequested arm instead of a field, and it starts at zero on every call. With dt = 1/60 s (16.67 ms) at 144 Hz, each call adds 6.94 ms, never a whole step, so update never runs and the world freezes while the screen keeps redrawing. At 40 fps each call adds 25 ms, runs one step and throws away 8.33 ms, so the game runs at 16.67 / 25 = 67% speed. As a field on Game, the leftover carries to the next call and both cases keep real time.

[Back to Platform & Window: the OS Event Loop](index.md)
