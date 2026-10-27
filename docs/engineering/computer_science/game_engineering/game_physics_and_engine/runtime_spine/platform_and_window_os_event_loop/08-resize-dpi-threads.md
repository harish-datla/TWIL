# Resize, DPI, Threads


The player drags the window border and the game freezes until they let go; on a laptop with a high-DPI screen the image comes out blurry or crammed into one corner; a worker thread creates the window and the program crashes on macOS. These are three loose ends the platform layer surfaces, and the rest of the engine has to respect them.

    Resize. The window reports a new size (Win32 WM_SIZE; winit WindowEvent::Resized). The Vulkan swapchain is sized to the old extent and goes stale, so it must be recreated. The mechanism (handling VK_ERROR_OUT_OF_DATE_KHR) belongs to the Vulkan module; here, just know resize is an event that invalidates GPU-side state. On Win32 it also arrives inside a modal loop, a nested message loop the OS runs itself: while the user drags the border, the default window procedure runs its own message pump (between WM_ENTERSIZEMOVE and WM_EXITSIZEMOVE), and your outer loop doesn't run until the drag ends[24].
    DPI. DPI (dots per inch) is a screen's pixel density. There are two sizes: the logical window size in screen coordinates, and the physical framebuffer size, the grid of pixels the GPU actually renders into. They differ on high-DPI displays. Render targets and viewports use pixels (the framebuffer size), not logical size, or everything is blurry or clipped[25]. On Windows the process must also declare itself DPI-aware, preferably as Per-Monitor V2, the mode where the app sizes itself for each monitor's scale and Windows never stretches it. It declares this in its application manifest (an XML resource embedded in the .exe) or by calling SetProcessDpiAwarenessContext before creating any window. An unaware process is rendered at 96 DPI and bitmap-stretched by the OS, which is its own blur[26]. A Per-Monitor V2 process gets WM_DPICHANGED instead when the window moves to a monitor with another scale, with a suggested new window rectangle it's expected to apply[27]. GLFW and winit opt in for you; a hand-written Win32 layer has to.
    Threads. Window creation and event handling belong on the main thread; macOS's AppKit requires it, and winit panics if you create the event loop elsewhere (Windows and Linux allow an opt-out through a platform extension; macOS doesn't). On Win32 a window's messages go to the queue of the thread that created it, so that thread must pump them. Rendering can move to other threads, but the window and its events stay put.

Worked example

On a Mac with a Retina display (content scale 2.0), a window created at 1280×720 has a 2560×1440 framebuffer. Set the Vulkan viewport from the window size and the scene fills only the top-left quarter of the image, 1280×720 of its 2560×1440 pixels[25].

On Windows at 150% scaling (144 DPI), a process that never declared itself DPI-aware is told its 1280×720 window is 1280×720, renders that many pixels, and Windows stretches the result to 1920×1080 on screen: every pixel smeared over 1.5 screen pixels, which is the blur.

The widget renders a test pattern (1-pixel lines, a disc, some text) the way each setup would, then shows the window's pixels magnified:
100%
125%
150%
200%
DPI-aware process
viewport = framebuffer
viewport = window size
logical window 1280×720pixels on screen 1920×1080app renders 1920×1080one rendered pixel per screen pixel: sharp
A 1280×720 window modeled at 1/8 scale, then magnified so single pixels are visible. Change the display scale, try both viewports, then turn off DPI awareness. A DPI-aware process with the viewport = framebuffer renders one pixel per screen pixel: sharp lines, full window. Set viewport = window size and at 150% the scene covers 1280×720 of the 1920×1080 image, 44% of it, in the top-left corner, with the clear color around it. Turn awareness off and the app renders 1280×720, which the OS stretches to 1920×1080: the right size, but each rendered pixel smeared over 1.5×1.5 screen pixels, so the thin lines go soft. Unaware, both viewport buttons give the same picture, because the app is told its window and its framebuffer are both 1280×720.

The widget below simulates the Win32 modal move loop. Drag the mock window by its title bar (or turn on the automatic drag). While you hold it, the game's own loop is stuck inside DispatchMessage, and the ball only moves when something draws a frame:
game loop only
WM_TIMER in the drag
render thread
drag automatically
strategy: render only in the game loopnot draggingframes drawn: 60 per second
A mock game window with a bouncing ball. Drag it by the title bar (or press drag automatically), once per strategy, and watch the frames-per-second readout. Game loop only: update and render both run in the stalled loop, so the ball freezes from the moment the drag starts until you let go and the readout drops to 0. On release, the 0.25 s catch-up cap from §5 throws away the rest of the stalled time, so the ball resumes close to where it froze. WM_TIMER in the drag: a timer started on WM_ENTERSIZEMOVE is dispatched by the modal loop, so frames keep coming; the widget draws 64 a second, the rate of the default 15.625 ms tick, while a real timer set to USER_TIMER_MINIMUM (10 ms) can't fire more than 100 times a second and fires less often when the tick is coarser. Render thread: the drawing (and here the simulation with it) never depended on the message loop, so it doesn't notice the drag. With the timer or the thread, the simulation keeps real time through the drag.

The complete program below does everything in this section by hand on Win32: it declares Per-Monitor V2 awareness, sizes a 1280×720 client area (the inside of the window, without the title bar and borders) for the window's DPI with AdjustWindowRectExForDpi, pumps with PeekMessage so the loop never blocks, applies the suggested rectangle on WM_DPICHANGED, and starts a timer on WM_ENTERSIZEMOVE so the game keeps stepping during a drag. That works because the modal loop is an ordinary GetMessage/DispatchMessage loop: it hands any message addressed to a window to that window's procedure[24], and WM_TIMER is a low-priority message that GetMessage generates when the timer expires and nothing more urgent is waiting[28]. DefWindowProc turns WM_CLOSE into DestroyWindow, and the program's WM_DESTROY handler posts the quit message that ends the pump.
The platform layer by hand on Win32 (complete program)
C++
Rust (windows-sys)

// The platform layer by hand on Win32: Per-Monitor V2 DPI awareness, a
// PeekMessage pump that never blocks, a fixed-timestep accumulator on
// QueryPerformanceCounter, and a timer that keeps the game ticking while
// Windows runs its own loop during a window drag. Link with user32.
#define UNICODE
#define WIN32_LEAN_AND_MEAN
#include <windows.h>
#include <cstdio>

struct Game {
    LARGE_INTEGER ticksPerSecond{}, previous{};
    double accumulator = 0.0;
    long long steps = 0, frames = 0, framesDuringDrag = 0;
    int framebufferWidth = 0, framebufferHeight = 0;
    UINT dpi = 96;
    bool dragging = false;
    static constexpr double kDt = 1.0 / 60.0;

    // One iteration of the game loop: measure elapsed time, step the simulation, draw.
    void frame() {
        LARGE_INTEGER now;
        QueryPerformanceCounter(&now);
        double frameTime = double(now.QuadPart - previous.QuadPart) / double(ticksPerSecond.QuadPart);
        previous = now;
        if (frameTime > 0.25) frameTime = 0.25;       // after a long stall, catch up at most 15 steps
        accumulator += frameTime;
        while (accumulator >= kDt) { ++steps; accumulator -= kDt; }   // update(kDt)
        ++frames;                                      // render(accumulator / kDt)
        if (dragging) ++framesDuringDrag;
    }
};

Game game;
constexpr UINT_PTR kDragTimer = 1;

LRESULT CALLBACK windowProc(HWND window, UINT message, WPARAM wParam, LPARAM lParam) {
    switch (message) {
    case WM_SIZE:                 // the client area, in physical pixels under Per-Monitor V2
        game.framebufferWidth = LOWORD(lParam);
        game.framebufferHeight = HIWORD(lParam);
        return 0;                 // a renderer would mark its swapchain for recreation here
    case WM_DPICHANGED: {         // moved to a monitor with another scale: use the suggested rect
        game.dpi = HIWORD(wParam);
        const RECT* suggested = reinterpret_cast<const RECT*>(lParam);
        SetWindowPos(window, nullptr, suggested->left, suggested->top,
                     suggested->right - suggested->left, suggested->bottom - suggested->top,
                     SWP_NOZORDER | SWP_NOACTIVATE);
        return 0;
    }
    case WM_ENTERSIZEMOVE:        // a drag begins and DefWindowProc enters its own message loop,
        game.dragging = true;     // which still dispatches WM_TIMER to this window
        SetTimer(window, kDragTimer, USER_TIMER_MINIMUM, nullptr);
        return 0;
    case WM_EXITSIZEMOVE:
        game.dragging = false;
        KillTimer(window, kDragTimer);
        return 0;
    case WM_TIMER:
        if (wParam == kDragTimer) game.frame();   // keep the game running inside the drag
        return 0;
    case WM_DESTROY:
        PostQuitMessage(0);       // ends the pump in main
        return 0;
    }
    return DefWindowProc(window, message, wParam, lParam);   // WM_CLOSE -> DestroyWindow, etc.
}

int main() {
    // Without this the process counts as DPI-unaware: it renders at 96 DPI and
    // Windows bitmap-stretches the result on a scaled display, which blurs it.
    SetProcessDpiAwarenessContext(DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2);

    WNDCLASSEX windowClass{};
    windowClass.cbSize = sizeof(windowClass);
    windowClass.lpfnWndProc = windowProc;
    windowClass.hInstance = GetModuleHandle(nullptr);
    windowClass.hCursor = LoadCursor(nullptr, IDC_ARROW);
    windowClass.lpszClassName = L"EngineWindow";
    RegisterClassEx(&windowClass);

    HWND window = CreateWindowEx(0, L"EngineWindow", L"Engine", WS_OVERLAPPEDWINDOW,
                                 CW_USEDEFAULT, CW_USEDEFAULT, CW_USEDEFAULT, CW_USEDEFAULT,
                                 nullptr, nullptr, windowClass.hInstance, nullptr);
    if (!window) return 1;
    // A 1280x720 client area at 100% scale, scaled for this monitor; the title bar and
    // borders come on top, at this DPI's sizes.
    game.dpi = GetDpiForWindow(window);
    RECT rect{0, 0, MulDiv(1280, game.dpi, 96), MulDiv(720, game.dpi, 96)};
    AdjustWindowRectExForDpi(&rect, WS_OVERLAPPEDWINDOW, FALSE, 0, game.dpi);
    SetWindowPos(window, nullptr, 0, 0, rect.right - rect.left, rect.bottom - rect.top,
                 SWP_NOMOVE | SWP_NOZORDER);
    ShowWindow(window, SW_SHOW);

    QueryPerformanceFrequency(&game.ticksPerSecond);
    QueryPerformanceCounter(&game.previous);
    for (bool running = true; running;) {
        // Drain everything waiting, then run a frame. PeekMessage returns at once when
        // the queue is empty, so the loop never sleeps here.
        MSG message;
        while (PeekMessage(&message, nullptr, 0, 0, PM_REMOVE)) {
            if (message.message == WM_QUIT) running = false;
            TranslateMessage(&message);   // key presses -> WM_CHAR
            DispatchMessage(&message);    // calls windowProc
        }
        if (running) game.frame();
    }
    std::printf("%lld steps, %lld frames (%lld during drags), framebuffer %dx%d at %u DPI\n",
                game.steps, game.frames, game.framesDuringDrag,
                game.framebufferWidth, game.framebufferHeight, game.dpi);
}

What's intentionally missing

    Input. Keyboard, mouse and raw-input messages arrive through the same windowProc; the Input Systems module covers turning them into actions.
    A renderer. The program never draws or presents, so with nothing to wait on its loop spins one core. The surface (§7), the swapchain and vsync come next.
    Swapchain recreation. WM_SIZE only records the new size. A renderer recreates its swapchain there, and skips drawing while the window is minimized and the size is 0×0.
    A render thread. The drag timer keeps the game ticking at the timer's rate. Moving rendering to its own thread removes the dependence on the message loop entirely, at the cost of synchronizing with it.
    Error handling. Every call can fail; a shipping layer checks RegisterClassEx, CreateWindowEx and the DPI calls, and falls back on older Windows versions that lack the per-monitor functions.

Check Your UnderstandingQ1 of 4

Your render loop freezes while the user drags the window title bar to resize, then resumes when they let go. Why?
On Win32 a resize/move enters a modal loop that runs its own message pump inside DispatchMessage, so your outer loop does not run until the drag ends.
The GPU cannot render while the window is being resized.
Resizing clears the event queue, so there is nothing to render.
0/4 correct

Wrong answers, and why: the resize freeze is the modal message loop, not a GPU limit or a cleared queue; in current winit you render in RedrawRequested via run_app, not in about_to_wait or the deprecated run closure; judder under a frame cap comes from tick-rounded sleeps, not the clock; and viewports take framebuffer pixels.

[Back to Platform & Window: the OS Event Loop](index.md)
