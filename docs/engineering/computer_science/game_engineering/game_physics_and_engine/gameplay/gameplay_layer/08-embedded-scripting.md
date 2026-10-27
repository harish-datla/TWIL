# Embedded scripting

Tuning a boss fight means changing its behavior dozens of times an hour, and a C++ rebuild per change kills that loop. Embed a scripting VM for fast iteration without a recompile, hot-reload, modding, and designer access. Lua is the canonical embedded choice: small, fast, and designed to be embedded (LuaJIT is usually much faster and ships in engines such as Defold). The C boundary is a virtual stack: you push args, call, and read results by index[16].

The program below is a complete host. It registers one engine function, spawn, that scripts can call; runs the script's on_update every tick in protected mode, so a script error is logged instead of crashing the game; drives a door sequence that spans several ticks as a coroutine; and hot-reloads the script partway through, once with a good edit and once with a broken one. The Rust version uses the mlua crate[17], which wraps the same C API.
Embedding Lua: an engine function, a protected call each tick, a coroutine, a hot reload (complete program)
C
Rust (mlua)

// Embedding Lua 5.4: expose an engine function, call the script every tick in
// protected mode, drive a multi-tick door sequence as a coroutine, and hot-reload.
#include <lauxlib.h>
#include <lua.h>
#include <lualib.h>
#include <stdio.h>
#include <string.h>

static int nextEntityId = 1;

// Callable from Lua as spawn(prefab, x, y). Arguments arrive on the stack; the
// return value is pushed, and the C function returns how many it pushed.
static int luaSpawn(lua_State* L) {
    const char* prefab = luaL_checkstring(L, 1);   // raises a Lua error on a bad argument
    double x = luaL_checknumber(L, 2);
    double y = luaL_checknumber(L, 3);
    printf("  engine: spawn %s at (%g, %g) -> id %d\n", prefab, x, y, nextEntityId);
    lua_pushinteger(L, nextEntityId++);
    return 1;
}

// Compile and run a chunk. On a syntax or runtime error, report it and keep the
// definitions already loaded, so a bad edit doesn't take the game down.
static void loadScript(lua_State* L, const char* source, const char* chunkName) {
    if (luaL_loadbuffer(L, source, strlen(source), chunkName) != LUA_OK ||
        lua_pcall(L, 0, 0, 0) != LUA_OK) {
        printf("  script error, keeping old code: %s\n", lua_tostring(L, -1));
        lua_pop(L, 1);                               // keep the stack balanced
    }
}

// Call the global on_update(dt) in protected mode: a script error is logged, not fatal.
static void callOnUpdate(lua_State* L, double dt) {
    lua_getglobal(L, "on_update");
    lua_pushnumber(L, dt);
    if (lua_pcall(L, 1, 0, 0) != LUA_OK) {
        printf("  on_update failed: %s\n", lua_tostring(L, -1));
        lua_pop(L, 1);
    }
}

static const char* kScriptV1 =
    "state = state or { ticks = 0 }   -- survives a reload: the table outlives the chunk\n"
    "local calls = 0                  -- does not: a reload runs this line again\n"
    "function wait(seconds)           -- yields once per tick until the time has passed\n"
    "  local elapsed = 0\n"
    "  while elapsed < seconds do elapsed = elapsed + coroutine.yield() end\n"
    "end\n"
    "function door_sequence()\n"
    "  print('  door: opening'); wait(1.0)\n"
    "  print('  door: open');    spawn('goblin_grunt', 4, 2); wait(1.5)\n"
    "  print('  door: closing'); wait(1.0)\n"
    "  print('  door: closed')\n"
    "end\n"
    "function on_update(dt)\n"
    "  state.ticks = state.ticks + 1; calls = calls + 1\n"
    "end\n"
    "function report() print(('  v1: state.ticks=%d calls=%d'):format(state.ticks, calls)) end\n";

static const char* kScriptV2 =   // the designer's edit, loaded while the game runs
    "state = state or { ticks = 0 }\n"
    "local calls = 0\n"
    "function on_update(dt) state.ticks = state.ticks + 1; calls = calls + 1 end\n"
    "function report() print(('  v2: state.ticks=%d calls=%d'):format(state.ticks, calls)) end\n";

static const char* kScriptBroken = "function on_update(dt) state.ticks = end\n";

int main(void) {
    lua_State* L = luaL_newstate();
    luaL_openlibs(L);             // every standard library: fine for your own scripts, not for mods
    lua_register(L, "spawn", luaSpawn);
    loadScript(L, kScriptV1, "=level.lua");

    // A coroutine is a Lua thread with its own stack. The registry reference
    // keeps the garbage collector from freeing it while the host still resumes it.
    lua_State* door = lua_newthread(L);
    int doorRef = luaL_ref(L, LUA_REGISTRYINDEX);
    lua_getglobal(door, "door_sequence");
    int doorStatus = LUA_YIELD;

    const double dt = 0.5;
    for (int tick = 1; tick <= 8; ++tick) {
        printf("tick %d\n", tick);
        callOnUpdate(L, dt);
        if (doorStatus == LUA_YIELD) {             // resume where the script left off
            int resultCount = 0;
            lua_pushnumber(door, dt);              // becomes coroutine.yield()'s return value
            doorStatus = lua_resume(door, L, 1, &resultCount);
            lua_pop(door, resultCount);
            if (doorStatus != LUA_OK && doorStatus != LUA_YIELD)
                printf("  door script failed: %s\n", lua_tostring(door, -1));
        }
        if (tick == 3) {
            lua_getglobal(L, "report"); lua_pcall(L, 0, 0, 0);
            loadScript(L, kScriptV2, "=level.lua");     // hot reload
            loadScript(L, kScriptBroken, "=level.lua"); // a bad save in the editor
        }
    }
    lua_getglobal(L, "report"); lua_pcall(L, 0, 0, 0);
    luaL_unref(L, LUA_REGISTRYINDEX, doorRef);
    lua_close(L);
    return 0;
}

Run either one at half a second per tick and the door starts opening on tick 1, the goblin spawns on tick 3 when the 1 s wait runs out, and the door shuts on tick 8. After tick 3 the good edit loads and the broken one fails to compile, so the good edit keeps running. At the end state.ticks reads 8 and calls reads 5. The next two subsections explain why.
What's intentionally missing

    A sandbox. Both hosts load io and os. Mod scripts need a restricted environment (the callout at the end of this section).
    Per-entity scripts. One global script drives everything. Engines usually give each entity its own table or environment and pass it the entity's handle, never a raw pointer.
    Runaway scripts. Nothing stops an infinite loop or unbounded allocation. The C API can install a hook with lua_sethook that fires every N instructions, and a custom allocator can cap memory[19].
    File watching. The reload is triggered by a tick number. An editor build watches the script files and reloads on save.
    Coroutine recovery. A coroutine that errors is reported and then dead; a real host restarts it or removes the behavior that owned it.

Behavior that spans ticks: coroutines

The door has to open, wait and close over several seconds, but update() must return every tick. Written as an update method, the code has to record where it is (a state and a timer) and pick up from there next tick, because when it returns, "the execution position is lost"[7]. A coroutine keeps the position for you: it's a function that can pause partway, called yielding, and later continue from the same line with its local variables intact[18]. In Lua, coroutine.yield returns whatever the next resume passes in, so the listing's wait() yields once per tick and adds that tick's dt each time it resumes. From C, lua_newthread creates the coroutine's own stack and lua_resume runs it until it yields or finishes[19]. Unity coroutines are the same idea in C#: yield return null resumes on the next frame[18].

Worked example

In the widget a tick is 0.125 s, a step that binary floating point adds up exactly (ten steps of 0.1 sum to slightly less than 1.0, so a 1 s wait would take an eleventh tick). After the switch the door opens for 8 ticks, holds for 12 and closes for 8, 28 ticks in all. The state-machine version stores a state (one of five) and a timer between those ticks, and needs a case per state. The coroutine stores neither: its local elapsed and the line it's paused on live on its suspended stack.
press the switch
auto-play
step one tick
reset
Coroutine (Lua)

function door_sequence()
  wait_until(switch_pressed)
  open_door();  wait(1.0)
  spawn_goblin()
  wait(1.5)
  close_door(); wait(1.0)
end

Stored between ticks: a suspended Lua thread, paused on line 2
State machine (C++)

switch (state) {
case Idle:
  if (switchPressed) { openDoor(); enter(Opening); }
  break;
case Opening:
  timer += dt;
  if (timer >= 1.0) { spawnGoblin(); enter(Holding); }
  break;
case Holding:
  timer += dt;
  if (timer >= 1.5) { closeDoor(); enter(Closing); }
  break;
case Closing:
  timer += dt;
  if (timer >= 1.0) enter(Done);
  break;
case Done:
  break;
}
// enter(next) sets state = next and timer = 0

Stored between ticks: state = Idle, timer = 0.000 s
tick 310 (38.750 s)goblins spawned: 0 / 0ticks where the two versions disagree: 0
Two copies of the same door behavior, each driving its own door at 8 ticks a second; the highlighted line is where each one stands. Press the switch, then pause auto-play and step one tick at a time. The coroutine sits on a wait line while time passes and remembers its position on its own stack; the state machine re-enters the switch every tick and finds its place from the state and timer it stored. Both doors move identically and the disagreement count stays at 0. The widget's coroutine is a JavaScript generator. A generator can pause only in its own body, so it needs yield* to pause inside a helper like wait; a Lua coroutine can yield from any function it calls.
What coroutines cost

The paused position lives on a Lua stack, not in fields you can inspect or write to disk. Lua's standard library has no way to save a suspended coroutine, while a state and a timer are two numbers a save game can store. The host also has to keep the coroutine reachable: threads are garbage-collected like any Lua object[19], which is why the C listing stores it in the registry with luaL_ref. In Unity, each coroutine is an allocated enumerator that holds references to its owner and captured variables until it ends or is stopped[18].
Hot reload

Hot reload means running the edited script again inside the live VM, so a designer sees a change without restarting the level. Lua compiles each chunk (one file or string of source, loaded in one go) as the body of an anonymous function[19], so running it again re-executes every top-level statement: each function on_update definition replaces the global, and each top-level local starts over. That's the listing's calls = 5: the reload re-ran local calls = 0, and the new on_update counted only ticks 4 to 8. State that has to survive lives in a table the new chunk reuses, state = state or { ticks = 0 }, which is why state.ticks reached 8.

    A running coroutine can mix old and new code. It keeps running the function it started with, so the door finishes the old door_sequence even after a reload defines a new one. But a global it calls, such as wait, is looked up at the call, so from the reload on it gets whatever that global holds: the new chunk's version if the chunk redefined it, the old one if not (the listing's v2 leaves wait alone).
    A syntax error changes nothing. luaL_loadbuffer compiles the whole chunk before any of it runs, so the broken edit is rejected and the old definitions stay. A chunk that fails halfway through running has already replaced everything it defined before the error.
    Data in tables, behavior in functions. The reload swaps functions safely when the state they act on lives somewhere the reload doesn't reset.

Don't script the inner loop; hot-reload's hard part is state; sandbox mods

The boundary has per-call overhead (stack marshaling, and for VM languages, interpretation). Epic's own guidance on Blueprint: the gap to C++ is usually insignificant and matters most in tight loops, systems that process large data sets, and tick-driven classes with many instances[20], so keep hot per-entity math native and script decisions and events. Don't claim Lua is "as fast as C" (LuaJIT narrows the gap, doesn't erase it); and Lua is the dominant embedded choice, not universally best (C# in Unity and Blueprint in Unreal ship more by install base). Hot-reload is easy to swap code but hard to migrate live state. Sandbox untrusted mods by whitelisting safe functions (not blacklisting), stripping io/os/package; beware load/setfenv/require escapes (and setfenv is gone in Lua 5.2+)[21].

Worked example

Give 10,000 entities a scripted on_update and the engine crosses the C-to-Lua boundary 10,000 times a frame, 600,000 times a second at 60 fps, each crossing pushing arguments and reading results through the stack. Script the boss's decisions instead (a handful of calls a frame) and keep the 10,000 entities' movement in one native loop.
Check Your UnderstandingQ1 of 4

A teammate says: deep inheritance is obsolete, so the only correct modern architecture is a pure ECS. What is the accurate correction?
Composition beat deep inheritance for cross-cutting behavior, but there are three live models (Unity GameObject+component, Unreal Actor+component with shallow inheritance, pure ECS), each shipping AAA; ECS is one option, not the only correct one.
Inheritance is always wrong and ECS is always fastest, so every engine should be pure ECS.
Composition and inheritance are the same thing in practice.
0/4 correct

Wrong answers, and why: ECS is one of three live models (don't crown it; inheritance isn't always wrong); a dangling reference is fixed by deferred destruction + generational handles (a null check can't catch a reused slot); a double buffer makes lag consistent, not zero; and a variant delegates to its base instead of copying it.
