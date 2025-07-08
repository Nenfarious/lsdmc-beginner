### Paper API 1.21.4+ – A hands-on PaperAPI starter kit for Java devs

If you already write Java. These notes fill the Minecraft-specific gaps so you can compile something useful **today** and avoid newbie pitfalls.

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
    id("io.papermc.paperweight.userdev") version "2.0.0-SNAPSHOT" // latest userdev ([github.com](https://github.com/PaperMC/paperweight/releases/))
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

Components survive version jumps, unlike old NBT tricks. Guides: ([docs.papermc.io](https://docs.papermc.io/paper/dev/api/?utm_source=chatgpt.com), [docs.papermc.io](https://docs.papermc.io/paper/dev/))

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

## 13. Practice mini-project

Build a **/home** system:

1. Command tree: `/home set`, `/home tp`, `/home list`.
2. Store homes in a SQL table keyed by `UUID`.
3. Async load on join, cache in a map.
4. Use `teleportAsync` with a small cooldown.
5. Show feedback with MiniMessage: `<green>Teleported <yellow>{name}</yellow>`.

Ship that and you will have touched 80 % of what real plugins need.

---

### Next stops

* Study the API pages for Events and Components. ([docs.papermc.io](https://docs.papermc.io/paper/dev/api/))
* Skim open-source plugins on GitHub for project structure ideas.
* Hang in **#plugin-dev** on Discord for live code reviews.
* Of course, hmu if you need anything!

**Write code, profile early, stay async, and you will look like a Paper veteran within a week.**
