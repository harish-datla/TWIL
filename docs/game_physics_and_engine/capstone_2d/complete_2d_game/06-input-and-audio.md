# Input and Audio

Read input inside `simulate(dt)` through an action map such as `move` and `fire`, not raw scancodes. When an event occurs, push a small sound command into the SPSC ring. The simulation is the sole producer; the audio callback is the sole consumer. A null-audio implementation keeps headless tests independent of a device.

```cpp
if (input.action(Action::Fire) && ship.cooldown <= 0) {
    spawnBullet(world, ship);
    audioRing.tryPush({ SoundId::Shot, 1.0f, panFor(ship.x) });
    ship.cooldown = FIRE_INTERVAL;
}
```

The ring must never block the simulation or audio callback. If full, drop and count the sound. A 1024-slot ring can absorb a large burst, but a stopped consumer can fill any finite ring.

For an SPSC ring, each side reads its own index relaxed, reads the other index with acquire, writes its slot, and publishes its own index with release. Release ensures the slot write becomes visible before the index; acquire ensures the consumer sees the slot after observing that index. Align indices to separate cache lines and avoid false sharing.
