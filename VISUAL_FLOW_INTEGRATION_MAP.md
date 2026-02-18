# ClickGUI & Module System - VISUAL INTEGRATION MAP

## COMPLETE EXECUTION FLOW DIAGRAM

```
═══════════════════════════════════════════════════════════════════════════════
                         EAGLERCRAFT CLIENT STARTUP
═══════════════════════════════════════════════════════════════════════════════

1. Minecraft.<init>()
   └─> initializeCustomClient()
       ├─> ModuleManager.getInstance()
       ├─> ModuleManager.registerModule(new ModuleSpeed())
       ├─> ModuleManager.registerModule(new ModuleNoFall())
       ├─> ModuleManager.registerModule(...more modules...)
       └─> ClickGUI.instance = new ClickGUI()
           └─> ClickGUI.initializePanels()
               ├─> Creates panel for COMBAT category
               ├─> Creates panel for MOVEMENT category
               ├─> Creates panel for RENDER category
               └─> ... for all categories ...

═══════════════════════════════════════════════════════════════════════════════
                           MAIN GAME LOOP
                      (This repeats every frame - 60 FPS)
═══════════════════════════════════════════════════════════════════════════════

FRAME START
│
├─ 1. INPUT PROCESSING (Minecraft.processKeyBinds)
│  └─> Check if Y pressed?
│      ├─> YES: ClickGUI.toggle() → GUI open/closed state changes
│      └─> NO: Check module keybinds
│          └─> ModuleManager.checkKeyBinds(keyCode)
│              └─> For each module:
│                  └─> if (module.getKeyCode() == keyCode)
│                      └─> module.toggle() → module onEnable/onDisable called
│
├─ 2. GAME UPDATES (physics, player movement, etc)
│  └─> EntityRenderer.orientCamera()
│  └─> WorldRenderer updates
│  └─> Entity updates
│
├─ 3. RENDER WORLD (3D rendering)
│  └─> EntityRenderer.renderWorld()
│      ├─> Renders terrain
│      ├─> Renders entities
│      ├─> Renders particles
│      └─> Renders 3D overlays (modules can hook here)
│
├─ 4. RENDER GAME OVERLAY ← ⭐ MAIN HOOK FOR MODULES & GUI
│  │   (GuiIngame.renderGameOverlay)
│  │
│  ├─> MODULE UPDATE PHASE (game logic)
│  │   └─> ModuleManager.updateModules()
│  │       └─> For each enabled module:
│  │           ├─> Try
│  │           │   └─> module.onUpdate()  ← Your module logic runs here!
│  │           │       ├─> Can modify world state
│  │           │       ├─> Can modify player
│  │           │       ├─> Can modify entities
│  │           │       └─> NO rendering here!
│  │           └─> Catch: disable module if it crashes
│  │
│  ├─> RENDER VANILLA HUD (health bar, hunger, etc)
│  │
│  ├─> MODULE RENDER PHASE (HUD elements only)
│  │   └─> ModuleManager.renderModules(mouseX, mouseY, partialTicks)
│  │       └─> For each enabled module:
│  │           ├─> Try
│  │           │   └─> module.onRender(mouseX, mouseY, partialTicks)
│  │           │       ├─> Draw text on screen
│  │           │       ├─> Draw 2D shapes
│  │           │       ├─> Draw HUD elements
│  │           │       └─> NO world state changes!
│  │           └─> Catch: exception handled
│  │
│  └─> CLICKGUI RENDER PHASE (Always on top)
│      └─> ClickGUI.render(mouseX, mouseY, partialTicks)
│          ├─> Draw semi-transparent background
│          ├─> For each panel:
│          │   ├─> Draw panel header with category name
│          │   ├─> For each button in panel:
│          │   │   ├─> Draw button background (green if enabled, gray if disabled)
│          │   │   ├─> Draw module name
│          │   │   ├─> Draw [ON]/[OFF] indicator
│          │   │   └─> For each setting slider:
│          │   │       └─> Draw slider with current value
│          │   └─> Draw panel border
│          └─> Wait for user interaction (mouse click/drag)
│
├─ 5. HANDLE CLICKGUI MOUSE INTERACTION (if GUI is open)
│  ├─> If Mouse.isClicked()
│  │   └─> ClickGUI.mouseClicked(mouseX, mouseY, button)
│  │       ├─> Check if dragging panel header → update panel position
│  │       └─> For each panel:
│  │           └─> For each button:
│  │               └─> If click is on button
│  │                   └─> module.toggle()
│  │                       ├─> Set module enabled = !enabled
│  │                       ├─> Call module.onEnable() OR module.onDisable()
│  │                       └─> Update button visual state
│  │
│  ├─> If Mouse.isDragged()
│  │   └─> ClickGUI.mouseDragged(X, Y)
│  │       ├─> Update dragging panel position
│  │       └─> For each slider:
│  │           └─> Update slider value based on mouse X movement
│  │
│  └─> If Mouse.isReleased()
│      └─> ClickGUI.mouseReleased(mouseX, mouseY, button)
│
├─ 6. RENDER CURRENT SCREEN (pause menu, inventory, etc)
│  └─> If (currentScreen != null)
│      └─> currentScreen.drawScreen()
│
└─ 7. SWAP BUFFERS & DISPLAY
    └─> Display.update()
       └─> Frame displayed on screen

═══════════════════════════════════════════════════════════════════════════════
                              MODULE LIFECYCLE
═══════════════════════════════════════════════════════════════════════════════

User presses Module's Keybind (or clicks button in ClickGUI):
│
├─> module.toggle()
│   └─> setEnabled(!enabled)
│       ├─> if (enabled == state) return  // Already in this state
│       ├─> enabled = state  // Change state
│       │
│       ├─> IF ENABLING:
│       │   ├─> Call module.onEnable()
│       │   │   └─> Your setup code runs here
│       │   │       ├─> Initialize variables
│       │   │       ├─> Set initial player state
│       │   │       ├─> Start timers
│       │   │       └─> Setup hooks
│       │   │
│       │   └─> NOW every frame:
│       │       ├─> module.onUpdate() is called  (game logic)
│       │       └─> module.onRender() is called  (HUD rendering)
│       │
│       └─> IF DISABLING:
│           ├─> Call module.onDisable()
│           │   └─> Your cleanup code runs here
│           │       ├─> Restore player state
│           │       ├─> Reset values
│           │       ├─> Stop timers
│           │       └─> Cleanup objects
│           │
│           └─> NOW:
│               ├─> module.onUpdate() NO LONGER CALLED
│               └─> module.onRender() NO LONGER CALLED

═══════════════════════════════════════════════════════════════════════════════
```

---

## FILE MODIFICATION CHECKLIST

### ✅ Files You Need to MODIFY

#### 1. `src/game/java/net/minecraft/client/gui/GuiIngame.java`

**Find method:** `renderGameOverlay(float partialTicks)`

**Add at the BEGINNING:**
```java
// At the very start of renderGameOverlay
ModuleManager.getInstance().updateModules();
```

**Add at the END (before final statements):**
```java
// Before function returns
ModuleManager.getInstance().renderModules(mouseX, mouseY, partialTicks);
if (ClickGUI.instance != null) {
    ClickGUI.instance.render(mouseX, mouseY, partialTicks);
}
```

**Add methods (anywhere in the class):**
```java
// Mouse interaction for ClickGUI
public void handleMouseClick(int mouseX, int mouseY, int button) {
    if (ClickGUI.instance != null && ClickGUI.instance.isOpen()) {
        ClickGUI.instance.mouseClicked(mouseX, mouseY, button);
    }
}

public void handleMouseRelease(int mouseX, int mouseY, int button) {
    if (ClickGUI.instance != null) {
        ClickGUI.instance.mouseReleased(mouseX, mouseY, button);
    }
}

public void handleMouseMove(int mouseX, int mouseY) {
    if (ClickGUI.instance != null) {
        ClickGUI.instance.mouseDragged(mouseX, mouseY);
    }
}
```

---

#### 2. `src/game/java/net/minecraft/client/Minecraft.java`

**Find method:** `processKeyBinds()` or similar key handling

**Add in the key event loop:**
```java
// When processing key events
if (Keyboard.getEventKeyState()) {
    int keyCode = Keyboard.getEventKey();
    
    // Toggle ClickGUI
    if (keyCode == Keyboard.KEY_Y) {
        if (ClickGUI.instance != null) {
            ClickGUI.instance.toggle();
        }
    }
    
    // Only check module keybinds if GUI is NOT open
    if (ClickGUI.instance == null || !ClickGUI.instance.isOpen()) {
        ModuleManager.getInstance().checkKeyBinds(keyCode);
    }
}
```

**Find method:** `run()` or constructor

**Add initialization:**
```java
// In Minecraft() constructor, after all initialization:
private void initializeCustomClient() {
    ModuleManager manager = ModuleManager.getInstance();
    
    // Register all modules
    manager.registerModule(new ModuleSpeed());
    manager.registerModule(new ModuleNoFall());
    manager.registerModule(new ModuleStep());
    manager.registerModule(new ModuleAimbot());
    // Add more...
    
    // Initialize GUI
    ClickGUI.instance = new ClickGUI();
}

// Call this early: initializeCustomClient();
```

---

### ✅ Files You Need to CREATE

Create all these files in the specified directories. Use the code snippets from QUICK_REFERENCE_SNIPPETS.md:

**Core System:**
- `src/game/java/net/lax1dude/eaglercraft/v1_8/client/Module.java`
- `src/game/java/net/lax1dude/eaglercraft/v1_8/client/Setting.java`
- `src/game/java/net/lax1dude/eaglercraft/v1_8/client/ModuleManager.java`
- `src/game/java/net/lax1dude/eaglercraft/v1_8/client/module/EnumModuleCategory.java`

**UI System:**
- `src/game/java/net/lax1dude/eaglercraft/v1_8/client/gui/ClickGUI.java`
- `src/game/java/net/lax1dude/eaglercraft/v1_8/client/gui/ClickGUIPanel.java`
- `src/game/java/net/lax1dude/eaglercraft/v1_8/client/gui/ClickGUIButton.java`
- `src/game/java/net/lax1dude/eaglercraft/v1_8/client/gui/ClickGUISlider.java` (optional)

**Example Modules:**
- `src/game/java/net/lax1dude/eaglercraft/v1_8/client/module/impl/ModuleSpeed.java`
- `src/game/java/net/lax1dude/eaglercraft/v1_8/client/module/impl/ModuleNoFall.java`
- `src/game/java/net/lax1dude/eaglercraft/v1_8/client/module/impl/ModuleStep.java`
- `src/game/java/net/lax1dude/eaglercraft/v1_8/client/module/impl/ModuleAimbot.java`

---

## CODE FLOW EXAMPLE: User Clicks Speed Module Button

```
User clicks "Speed" button in ClickGUI
│
└─> ClickGUIButton.mouseClicked(X, Y, LEFT_BUTTON)
    └─> Is X,Y inside button bounds? YES
        └─> module.toggle()  ← Get the Module instance (Speed module)
            └─> speedModule.setEnabled(!speedModule.isEnabled())
                └─> speedModule was FALSE, becomes TRUE
                    ├─> speedModule.onEnable()
                    │   └─> Initialize speed modifier
                    │
                    └─> Next frame:
                        ├─> ModuleManager.updateModules()
                        │   └─> for (Module m : modules)
                        │       └─> if (m.isEnabled())  ← Speed is now enabled!
                        │           └─> m.onUpdate()
                        │               └─> speedModule.onUpdate()  ← RUNS EVERY FRAME NOW
                        │                   └─> Apply speed boost to player
                        │
                        └─> ModuleManager.renderModules()
                            └─> speedModule.onRender()  ← Optionally draw HUD info
```

---

## CODE FLOW EXAMPLE: Module Processes Game Logic Update

```
Frame 1: Speed module is ENABLED
│
├─ GuiIngame.renderGameOverlay() called
│  │
│  ├─ ModuleManager.updateModules()  ← MODULE UPDATE PHASE
│  │  │
│  │  └─> for (Module module : modules)
│  │      └─> if (module.isEnabled())
│  │          └─> module.onUpdate()
│  │
│  │              ↓↓↓ SPEED MODULE RUNS HERE ↓↓↓
│  │
│  │              speedModule.onUpdate() {
│  │                  EntityPlayerSP player = Minecraft.getMinecraft().thePlayer;
│  │                  
│  │                  // Check if player is moving
│  │                  if (player.movementInput.moveForward != 0 || 
│  │                      player.movementInput.moveStrafe != 0) {
│  │                      
│  │                      // Get movement direction
│  │                      double yaw = player.rotationYaw * Math.PI / 180.0;
│  │                      double speed = 1.5 * 0.05;  // From speedMultiplier setting
│  │                      double sin = Math.sin(yaw);
│  │                      double cos = Math.cos(yaw);
│  │                      
│  │                      double forward = player.movementInput.moveForward;
│  │                      double strafe = player.movementInput.moveStrafe;
│  │                      
│  │                      // MODIFY PLAYER MOTION (Y axis not affected)
│  │                      player.motionX += (forward * cos - strafe * sin) * speed;
│  │                      player.motionZ += (strafe * cos + forward * sin) * speed;
│  │                      // Player is now moving faster!
│  │                  }
│  │              }
│  │
│  │              ↑↑↑ END SPEED MODULE ↑↑↑
│  │
│  └─> [Rest of game rendering continues...]
│
└─ Frame displays on screen, player moved faster!
```

---

## CODE FLOW EXAMPLE: Module Renders HUD Info

```
Frame 1: Speed module is ENABLED
│
├─ GuiIngame.renderGameOverlay() called [continues from game logic]
│  │
│  ├─ ModuleManager.renderModules(mouseX, mouseY, partialTicks)
│  │  │        ↓↓↓ MODULE RENDER PHASE ↓↓↓
│  │  │
│  │  └─> for (Module module : modules)
│  │      └─> if (module.isEnabled())
│  │          └─> module.onRender(mouseX, mouseY, partialTicks)
│  │
│  │              speedModule.onRender(X, Y, partialTicks) {
│  │                  FontRenderer font = Minecraft.getMinecraft().fontRendererObj;
│  │                  
│  │                  // Draw some text on the screen
│  │                  font.drawStringWithShadow(
│  │                      "Speed: 1.5x",  // Text telling user it's active
│  │                      10, 10,          // Screen coordinates
│  │                      0xFF00FF00       // Green color
│  │                  );
│  │              }
│  │
│  │              ↑↑↑ END SPEED MODULE RENDER ↑↑↑
│  │
│  └─> [Other modules render their stuff...]
│
└─ Frame displays with "Speed: 1.5x" text on screen
```

---

## COMPLETE EXAMPLE: Full User Interaction Cycle

```
INITIAL STATE:
- Speed module is OFF (disabled)
- Player is moving normally
- No "Speed" text appears on screen

USER ACTION #1: Press Y key
│
├─ Minecraft.processKeyBinds() detects Y key
└─> if (keyCode == Keyboard.KEY_Y)
    └─> ClickGUI.instance.toggle()
        └─> open = !open  (was FALSE, now TRUE)
        └─> ClickGUI panel are now VISIBLE on screen

FRAME RENDER:
├─ GuiIngame.renderGameOverlay() renders normally
└─> ClickGUI.render() draws all the panels, buttons, etc
    └─> Speed button shows "[OFF]" in gray

USER ACTION #2: User clicks Speed button in ClickGUI
│
├─ Mouse click event detected at button position
└─> ClickGUIButton.mouseClicked(X, Y, 0)  [LEFT_BUTTON = 0]
    └─> Is click in button bounds? YES
        └─> module.toggle()  [Gets speedModule]
            └─> speedModule.setEnabled(true)  [Was false, now true]
                ├─> speedModule.onEnable()
                │   └─> (any initialization needed)
                │
                └─> speedModule is now ENABLED ✓

NEXT FRAMES: Player movement is boosted
│
├─ FRAME 1:
│  ├─ GuiIngame.renderGameOverlay()
│  │  ├─ ModuleManager.updateModules()
│  │  │  └─> speedModule.onUpdate()
│  │  │      └─> player.motionX += ...  [Speed boost applied]
│  │  └─> ModuleManager.renderModules()
│  │      └─> speedModule.onRender()
│  │          └─> Draw "Speed: 1.5x" text on screen (green)
│  │
│  └─> ClickGUI.render()
│      └─> Speed button now shows "[ON]" in green
│
├─ FRAME 2: (same as frame 1, every frame)
│  └─> Player continues boosted movement
│
└─ FRAME N: (same pattern...)

USER ACTION #3: Press Y key again
│
├─ Minecraft.processKeyBinds() detects Y key
└─> ClickGUI.instance.toggle()
    └─> open = !open  (was TRUE, now FALSE)
    └─> ClickGUI panels are now HIDDEN from screen

FRAME RENDERS: ClickGUI hidden, modules still running
│
├─ GuiIngame.renderGameOverlay()
│  ├─ ModuleManager.updateModules()
│  │  └─> speedModule.onUpdate()  [Still enabled!]
│  │      └─> player.motionX += ...  [Speed boost still applied]
│  └─> speedModule.onRender() still draws "Speed: 1.5x"
│
└─> ClickGUI.render() does NOT run (GUI closed)

USER ACTION #4: Click Speed module's keybind (or open GUI and click button again)
│
├─> speedModule.toggle()
│   └─> speedModule.setEnabled(false)  [Was true, now false]
│       ├─> speedModule.onDisable()  [Cleanup]
│       └─> speedModule is now DISABLED ✓

NEXT FRAMES: Normal movement restored
│
└─ ModuleManager.updateModules()
   └─> for (Module m : modules)
       └─> if (m.isEnabled())  ← speedModule is FALSE, so NOT called
           └─> speedModule.onUpdate() NOT RUN
```

---

## DEBUGGING CHECKLIST

**Module not appearing in GUI?**
- [ ] Did you call `manager.registerModule(new MyModule())`?
- [ ] Is the module's category enum correct?
- [ ] Check ModuleManager.getModulesByCategory() returns your module

**Module not updating?**
- [ ] Is the module enabled? Check `module.isEnabled()`
- [ ] Is `ModuleManager.updateModules()` being called from GuiIngame?
- [ ] Does `onUpdate()` have a null check for player/world?
- [ ] Did you add try-catch in `onUpdate()`?

**GUI doesn't appear?**
- [ ] Press Y key - does GUI toggle?
- [ ] Check ClickGUI.instance is not null
- [ ] Is `ClickGUI.render()` being called from GuiIngame?
- [ ] Check screen width/height are correct

**Module doesn't toggle?**
- [ ] Check keybind is registered: `module.setKeyCode(keyCode)`
- [ ] Is `ModuleManager.checkKeyBinds()` being called?
- [ ] If using GUI, did you click the button correctly?
- [ ] Check `onEnable()` method is being called

**Performance issues?**
- [ ] Are you creating lists/objects every frame? (Cache them)
- [ ] Are you iterating through huge lists? (Optimize loops)
- [ ] Are you doing heavy calculations every frame? (Throttle them)
- [ ] Remove old modules you're not using

---

## SUMMARY

1️⃣ **Startup**: ModuleManager and ClickGUI initialized
2️⃣ **Every Frame**: GuiIngame.renderGameOverlay() calls module updates → renders
3️⃣ **User Input**: Key presses toggle modules or open GUI
4️⃣ **Module Lifecycle**: onEnable → onUpdate/onRender loop → onDisable
5️⃣ **GUI Logic**: Click button → module.toggle() → onEnable/Disable called

That's the complete flow!
