# Performance Optimization Guide

This document outlines performance optimizations implemented in the Dinosaur Exploder game and provides best practices for future development.

## Recent Optimizations (2026-02-07)

### 1. Entity Spawn Capping 🎯 (HIGH IMPACT)

**Problem:** Boss coin drops grew linearly with player level, causing severe performance issues at high levels.

**Solution:** Implemented maximum caps for coin spawning:
- **Orange Dino (Level 5 Boss)**: Capped at 15 coins (was 2×level)
- **Red Dino (Level 10 Boss)**: Capped at 10 coins (was 1×level)

**Impact:**
- Level 10: 50% reduction (20→15 for Orange, 10→10 for Red)
- Level 20: 63% reduction (40→15 for Orange, 20→10 for Red)
- Level 50+: 85%+ reduction (100+→15 for Orange, 50+→10 for Red)

**Files:**
- `src/main/java/com/dinosaur/dinosaurexploder/controller/core/collisions/ProjectileOrangeDinoCollision.java`
- `src/main/java/com/dinosaur/dinosaurexploder/controller/core/collisions/ProjectileRedDinoCollision.java`

**Best Practice:**
```java
// ❌ BAD: Unbounded growth
for (int i = 0; i < level * multiplier; i++) {
    spawn("entity", x, y);
}

// ✅ GOOD: Capped with reasonable maximum
int entityCount = Math.min(level * multiplier, MAX_REASONABLE_COUNT);
for (int i = 0; i < entityCount; i++) {
    spawn("entity", x, y);
}
```

### 2. Console Logging Removal 📝 (MEDIUM IMPACT)

**Problem:** 12 `System.out.println()` calls in collision handlers created I/O overhead on every collision event.

**Solution:** Removed all debug console logging from hot code paths:
- Collision handlers (9 occurrences)
- Game actions (3 occurrences)

**Impact:** Eliminated console I/O overhead from all collision events (potentially hundreds per second during active gameplay).

**Files Modified:**
- All collision handler files in `src/main/java/com/dinosaur/dinosaurexploder/controller/core/collisions/`
- `src/main/java/com/dinosaur/dinosaurexploder/controller/core/GameActions.java`

**Best Practice:**
```java
// ❌ BAD: Console output in hot paths
onCollisionBegin(TYPE_A, TYPE_B, (a, b) -> {
    System.out.println("Collision detected!");  // Called many times per second
    handleCollision(a, b);
});

// ✅ GOOD: Use proper logging only for important events
private static final Logger logger = Logger.getLogger(MyClass.class.getName());

// Only log important, infrequent events
if (bossDefeated) {
    logger.log(Level.INFO, "Boss defeated at level {0}", level);
}
```

### 3. Image Resource Caching 🖼️ (MEDIUM IMPACT)

**Problem:** Image objects were created on every entity spawn, especially inefficient for frequently spawned projectiles.

**Solution:** Implemented static image cache in `GameEntityFactory`:
- Added `imageCache` HashMap for image storage
- Created `getCachedImage()` method for lazy-loaded caching
- Updated entity spawning methods to use cached images

**Impact:** 
- First load: Same as before
- Subsequent spawns: Near-instant image retrieval
- Most impactful for projectiles (spawned many times per second)

**Files:**
- `src/main/java/com/dinosaur/dinosaurexploder/model/GameEntityFactory.java`

**Best Practice:**
```java
// ❌ BAD: Creating new Image every spawn
@Spawns("projectile")
public Entity newProjectile(SpawnData data) {
    Image img = new Image(getClass().getResourceAsStream("/path/to/image.png"));
    return entityBuilder().view(new ImageView(img)).build();
}

// ✅ GOOD: Use cached images
private static final Map<String, Image> imageCache = new HashMap<>();

private Image getCachedImage(String path) {
    return imageCache.computeIfAbsent(path, p -> 
        new Image(getClass().getResourceAsStream("/" + p))
    );
}

@Spawns("projectile")
public Entity newProjectile(SpawnData data) {
    Image img = getCachedImage("path/to/image.png");
    return entityBuilder().view(new ImageView(img)).build();
}
```

## Performance Best Practices

### 🎯 Entity Management

1. **Cap entity counts**: Always use `Math.min()` when spawning entities based on game progression
2. **Use object pooling**: For frequently created/destroyed entities (future enhancement)
3. **Clean up off-screen entities**: Use `OffscreenCleanComponent` for moving entities
4. **Batch operations**: Group entity operations when possible

### 📝 Logging

1. **Avoid console output in hot paths**: No `System.out.println()` in:
   - Collision handlers
   - Per-frame update methods (`onUpdate`)
   - Event callbacks that fire frequently
2. **Use proper logging**: Java's `java.util.logging.Logger` with appropriate levels
3. **Log only important events**: Level ups, game over, boss defeats, errors

### 🖼️ Resource Management

1. **Cache loaded resources**: Images, sounds, and other assets should be cached
2. **Lazy loading**: Load resources only when needed, cache on first use
3. **Static caches**: Use static collections for resources shared across instances

### 🔄 Update Loop Optimization

1. **Minimize per-frame work**: Cache calculations when possible
2. **Use dirty flags**: Only update UI when values actually change
3. **Batch UI updates**: Update multiple text fields in one operation

## Performance Monitoring

### How to Identify Performance Issues

1. **Frame rate drops**: Watch for FPS drops during gameplay
2. **Memory profiling**: Use Java VisualVM or similar tools
3. **Entity count**: Monitor entity count in game (should stay reasonable)
4. **Spawn rate**: Watch for excessive entity spawning

### Testing High-Level Gameplay

When testing performance improvements:
1. Play to level 20+ to see long-term effects
2. Monitor frame rate during boss battles
3. Check for lag when many entities are on screen
4. Test on lower-end hardware if possible

## Future Optimization Opportunities

### Not Yet Implemented

1. **Object Pooling**: Reuse projectile entities instead of creating/destroying
2. **Spatial Partitioning**: Optimize collision detection for many entities
3. **Texture Atlases**: Combine multiple textures into single image
4. **Dirty Flags for UI**: Only update UI components when values change
5. **Entity Culling**: Don't update off-screen entities

### Low Priority

- Entity search optimization in `GameActions.showLevelMessage()` (only called on level up)
- Audio manager cleanup optimization (not a hot path)

## Contributing

When adding new features, please:
1. Follow these performance best practices
2. Test at high levels (20+) to ensure scalability
3. Profile before and after changes if adding intensive operations
4. Run the full test suite: `mvn test`
5. Format code: `mvn spotless:apply`

## Resources

- [FXGL Performance Guide](https://github.com/AlmasB/FXGL/wiki/Performance)
- [Java Performance Best Practices](https://docs.oracle.com/en/java/javase/21/)
- [Game Optimization Patterns](https://gameprogrammingpatterns.com/optimization-patterns.html)

---

Last Updated: 2026-02-07
Contributors: GitHub Copilot, Karthik-Dsa
