# Input Buffering


A fighting-game player swears they did the motion, but the special move didn't come out. The directions may have been right and simply landed a few frames too far apart. Input buffering widens timing windows so the game registers the input the player meant. It's a feel feature that adds no latency: it changes when an input is accepted, and the result still renders as soon as it would have.

    Motion windows. A fighting-game quarter-circle, rolling the stick from down to forward, counts if its directions land within an N-frame window. Community frame-data analysis puts Street Fighter 6's quarter-circle window at 11 frames, one to three frames stricter than Street Fighter V's windows, and reports that SF6 reads inputs three times per frame where SFV read once[15].
    Coyote time. A jump still fires for a few frames after you walk off a ledge. Celeste's designer documents this kind of forgiveness (coyote time, jump buffering, corner correction) as deliberate feel[14].
    Jump buffering. A jump pressed just before landing fires on the landing frame instead of being dropped[14].

Worked example

Take an 11-frame window, SF6's quarter-circle figure, about 183 ms at 60 fps. The player presses Down on frame 100, Forward on frame 106 and Punch on frame 114. The window reaches back 11 frames from the Punch, to frame 103, so the Down on frame 100 falls outside it and the move fails, even though the order was right. Widen the window to 15 frames and it reaches back to frame 99: the same presses now connect.

The widget is that check. The move fires only if the sequence lands inside the buffer window:
▼ Down
▶ Forward
● Punch
Window 11 f
Reset
The timeline scrolls left with "now" at the right edge, and the shaded box is the buffer window. Each press drops a marker. The target is a quarter-circle: Down, then Forward, then Punch. Pressing Punch judges the window that ends at that press (the "judged window" bracket): a green "MOVE! quarter-circle" if Down and Forward both landed inside it in order, a red "no Down in window" or "no Forward in window" if not. Press them too slowly and a correct sequence still fails, because buffering checks timing as well as order. The example's spacing (Forward 6 frames after Down, Punch 8 frames after that) gives "no Down in window" at 11 f and connects at 15 f. With a widget control focused, the keyboard is faster than clicking: S or ↓ for Down, D or → for Forward, J for Punch. At 60 fps, 11 frames is about 183 ms.

[Back to Input Systems (HID)](index.md)
