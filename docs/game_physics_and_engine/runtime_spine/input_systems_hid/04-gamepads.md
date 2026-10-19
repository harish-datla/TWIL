# Gamepads


Your players own Xbox, PlayStation and Switch controllers plus third-party pads, and each OS exposes them through different APIs. Think of one standard gamepad (two sticks, two triggers, a d-pad or four-way cross, plus face and shoulder buttons) and let a library map every real device onto it. The library matters because of the many backends underneath.

On Windows, XInput is the Xbox-controller API, and it reports the two triggers as independent axes. (Microsoft now recommends its newer GameInput API, a superset of XInput, DirectInput and Raw Input, for new code: GameInput overview.) DirectInput, the older DirectX joystick API, combines both triggers onto one shared axis when it reads an Xbox controller: one trigger moves the axis one way from center and the other moves it the opposite way, so pulling both fully reads the same as pulling neither. Microsoft notes this "does mean it is not possible to see all possible trigger combination values through DirectInput... To test the trigger values separately, you must use XInput"[7]. SDL hides all of it behind a standard model with separate left and right trigger axes, using a community mapping database[8]. Rust's gilrs does the same and, helpfully, names face buttons by position (South/East/North/West) to sidestep the Xbox-vs-Nintendo A/B swap[9].

Worked example

The player holds the left stick fully up and a quarter of the way right. SDL reports LEFTX = 8192 and LEFTY = −32768. Dividing by 32767, as the C++ below does, gives rawX = 0.25 and rawY = −1.00003. So up is negative Y in SDL, and the vector's length comes out at 1.03, slightly past full deflection, which is why §5 clamps the length to 1 before shaping it.
Read a gamepad: poll for connect, sample state per frame
C++ · SDL3
Rust · gilrs

// SDL3: stick axes are int16, -32768 (up/left) to 32767 (down/right): Y grows DOWNWARD.
// Triggers are 0..32767.
SDL_Event event;
while (SDL_PollEvent(&event)) {
    if (event.type == SDL_EVENT_GAMEPAD_ADDED)
        gamepad = SDL_OpenGamepad(event.gdevice.which);   // standard-mapped handle
}
if (gamepad) {
    float rawX = SDL_GetGamepadAxis(gamepad, SDL_GAMEPAD_AXIS_LEFTX) / 32767.0f;
    float rawY = SDL_GetGamepadAxis(gamepad, SDL_GAMEPAD_AXIS_LEFTY) / 32767.0f;
    bool jump = SDL_GetGamepadButton(gamepad, SDL_GAMEPAD_BUTTON_SOUTH);
}


[Back to Input Systems (HID)](index.md)
