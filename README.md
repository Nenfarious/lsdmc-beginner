### Paper API 1.21.4+ – A hands-on starter kit for Java devs

Since you already write Java. These notes fill the Minecraft-specific gaps so you can compile something useful **today** and avoid newbie pitfalls.

---

## 1. One-time workstation setup

```bash
# JDK 21 (any vendor) – Paper is built for it
sdk install java 21-tem       # or use your package manager

# Gradle wrapper (recommended)
gradle wrapper --gradle-version 8.8
```

**`settings.gradle.kts`**

```kotlin
plugins {
    id("io.papermc.paperweight.userdev") version "2.0.0-SNAPSHOT" // latest userdev ([github.com](https://github.com/PaperMC/paperweight/releases))
}
```

**`build.gradle.kts`**

```kotlin
repositories {
    maven("https://repo.papermc.io/repository/maven-public/")
}
dependencies {
    paperweightDevelopmentBundle("io.papermc.paper:paper-dev-bundle:1.21.4-R0.1-SNAPSHOT")
    implementation("net.kyori:adventure-api:4.17.0")
}
```

Run a throw-away server while you code:

```bash
./gradlew runServer   # launches a clean Paper test instance
```

---

## 2. Plugin anatomy in five files

| File               | Purpose                                                  |
| ------------------ | -------------------------------------------------------- |
| `paper-plugin.yml` | Metadata, permissions, soft-depend list                  |
| `Main.java`        | Extends `JavaPlugin` – entry/exit hooks                  |
| `commands/…`       | Brigadier trees registered via the LifecycleEventManager |
| `listeners/…`      | EventSubscriber classes                                  |
| `services/…`       | Logic classes injected into commands & listeners         |

`paper-plugin.yml`

```yaml
name: WarpPads
version: 0.1.0
main: dev.nenf.warppads.WarpPads
api-version: "1.21"
permissions:
  warppads.use:
    description: Allow /pad use
    default: true
```

---

## 3. Events – the “hello world” of Paper

```java
public final class JoinListener implements Listener {
    @EventHandler
    public void onJoin(PlayerJoinEvent e) {
        Component msg = Component.text("✦ Welcome ✦", NamedTextColor.AQUA);
        e.getPlayer().sendMessage(msg);
    }
}
```

* Register once in `onEnable` with `getServer().getPluginManager().registerEvents(this, this);`.
* Avoid heavy work here – delegate to async tasks instead.

---

## 4. Brigadier commands the Paper way

```java
public class PadCommand {
    public static void register(JavaPlugin plugin) {
        LifecycleEventManager.registerCommand("pad", builder -> {
            builder.executes(ctx -> {
                Player p = ctx.getSource().getPlayer();
                plugin.getComponent(PadService.class).openGui(p);
                return 1;
            });
        });
    }
}
```

The `LifecycleEventManager` re-registers commands after `/reload` so you never touch the old Bukkit `CommandExecutor`. ([docs.papermc.io](https://docs.papermc.io/paper/dev/command-api/basics/registration/))

---

## 5. Async work without killing the main thread

```java
// Example: expensive SQL query
CompletableFuture.runAsync(() -> padRepo.loadPads(uuid))
                 .thenAcceptAsync(pads -> gui.open(p, pads), Bukkit.getScheduler());
```

Rules of thumb:

* Never access Bukkit API from a non-main thread unless the method says “Async” in Javadocs.
* For CPU loops, run async and push results back to main with `runTask()`.

---

## 6. Persistent data & custom items

### 6.1 PersistentDataContainer (attach data to entities/blocks)

```java
NamespacedKey KEY = new NamespacedKey(plugin, "lastVisited");
player.getPersistentDataContainer()
      .set(KEY, PersistentDataType.LONG, System.currentTimeMillis());
```

### 6.2 Data components (ItemStack#components)

```java
ItemStack ticket = new ItemStack(Material.PAPER);
ticket.set(Component.key("display_name"),
           Component.text("Dungeon Ticket", NamedTextColor.GOLD));
```

Components survive version jumps, unlike old NBT tricks. Guides: ([docs.papermc.io](https://docs.papermc.io/paper/dev/api/), [docs.papermc.io](https://docs.papermc.io/paper/dev/))

---

## 7. GUI basics in three lines

```java
Inventory inv = Bukkit.createInventory(new PadGuiHolder(), 27, "Warp Pads");
player.openInventory(inv);
```

Implement `PadGuiHolder` to tag your inventory so clicks route to the right handler. Full guide in “Custom InventoryHolder”. ([docs.papermc.io](https://docs.papermc.io/paper/dev/))

---

## 8. Teleports and chunk ops – use the async flavor

```java
player.teleportAsync(targetLocation);   // returns CompletableFuture<Boolean>
```

Loads the chunk off-thread, then warps. No stutter. ([jd.papermc.io](https://jd.papermc.io/paper/1.21.7/org/bukkit/entity/Entity.html))

---

## 9. Configs that scale

* `config.yml` – human settings, reloadable.
* `paper-world-defaults.yml` – world performance knobs (use git to diff your tweaks). ([docs.papermc.io](https://docs.papermc.io/paper/reference/world-configuration/))
* For big datasets use SQLite or MySQL with an ORM (Exposed, JDBI) and cache in memory.

---

## 10. Debugging & profiling

| Tool           | Run command                    | Use case                |
| -------------- | ------------------------------ | ----------------------- |
| Timings v2     | `/timings report`              | Quick TPS hot-spots     |
| Spark profiler | `/spark profiler --timeout 60` | Flamegraphs, CPU/memory |
| `paper.log`    | grep for `[WARN]`              | Detect unsafe sync work |

---

## 11. Integration snippets

### Vault economy

```java
Economy econ = Bukkit.getServicesManager().load(Economy.class);
econ.depositPlayer(player, 50.0);
```

### WorldGuard event hook

```java
@EventHandler
public void onBreak(BlockBreakEvent e){
    if (!WorldGuardUtils.isInRegion(e.getBlock().getLocation(), "mine")) {
        e.setCancelled(true);
    }
}
```

---

## 12. Common pitfalls checklist

* Synchronous chunk loads (`world.getChunkAt(x,z)`).
* `ChatColor` instead of Adventure components.
* Registering commands in `onEnable` with Bukkit’s `getCommand("x")` – outdated.
* Forgetting to add `provides: restart` in your systemd service; Paper needs full restart for JDK upgrades.

---

## 13. Practice projects

### 3 Simple Mini-Plugins (Pick 1 or All 3)  

#### **1. BlockLogger**  
**Purpose**: Log when players break specific blocks (e.g., diamonds, ancient debris).  
**Code snippets**:  
```java
// Listener
@EventHandler
public void onBreak(BlockBreakEvent e) {
    if (e.getBlock().getType() == Material.DIAMOND_ORE) {
        plugin.getLogger().info(e.getPlayer().getName() + " mined diamonds!");
    }
}

// Config.yml
logged_blocks: ["DIAMOND_ORE", "ANCIENT_DEBRIS"]
```  
**Skills**: Event handling, config parsing, basic logging.  

---

#### **2. PlayerTrails**  
**Purpose**: Leave a particle trail behind players when they walk. Toggle with `/trail on/off`.  
**Code snippets**:  
```java
// Command
if (args[0].equals("on")) {
    player.sendActionBar(Component.text("Trails enabled", NamedTextColor.GREEN));
    plugin.getComponent(TrailService.class).enableTrail(player);
}

// TrailService
public void enableTrail(Player p) {
    Bukkit.getScheduler().runTaskTimer(plugin, () -> {
        p.getWorld().spawnParticles(Particle.FIREWORKS_SPARK, p.getLocation(), 10);
    }, 0, 20L);
}
```  
**Skills**: Command registration, particle API, task scheduling.  

---

#### **3. WeatherAlert**  
**Purpose**: Warn players when a storm starts in their world.  
**Code snippets**:  
```java
// Event
@EventHandler
public void onStorm(WeatherChangeEvent e) {
    if (e.toWeatherState() == WeatherState.STORM) {
        e.getWorld().getPlayers().forEach(p -> 
            p.sendMessage(MiniMessage.miniMessage().deserialize(
                "<red>Storm incoming! Take cover.</red>"
            ))
        );
    }
}

// Config
alert_delay_seconds: 30
```  
**Skills**: World events, message formatting, config tuning.  

---

### Those too easy? Try: **MazeRunner**  
**Still simple, but more advanced**:  
1. Command `/maze` → Generates a random 10x10 maze at a safe location.  
2. Player must solve it within 60 seconds to win rewards.  
3. Use `teleportAsync()` to send them to the maze.  
4. Track progress via a `Map<Player, Location>` cache.  
5. Send a GUI menu to choose maze difficulty (easy/medium/hard).  

**Skills**: Async teleports, GUI handling, timer logic, config-based rewards.  

--- 

These projects teach **event hooks**, **command flow**, **particle effects**, and **async safety** without requiring ORM or complex data systems. Ship one, and you’ll already know more than most plugin devs.

---

### Next stops

* Study the API pages for Events and Components. ([docs.papermc.io](https://docs.papermc.io/paper/dev/api/))
* Skim open-source plugins on GitHub for project structure ideas.
* Hang in **#plugin-dev** on Discord for live code reviews.
* Of course, hmu if you need anything!

**Write code, profile early, stay async, and you will look like a Paper veteran within a week.**
