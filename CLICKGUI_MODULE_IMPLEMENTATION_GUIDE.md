# Eaglercraft 1.8 - ClickGUI & Module System - COMPREHENSIVE IMPLEMENTATION GUIDE

## TABLE OF CONTENTS
1. [Architecture Overview](#architecture-overview)
2. [Core Module System Architecture](#core-module-system-architecture)  
3. [The Event System](#the-event-system)
4. [ClickGUI Architecture](#clickgui-architecture)
5. [Step-by-Step Implementation](#step-by-step-implementation)
6. [Complete Code Examples](#complete-code-examples)
7. [Integration Points](#integration-points)
8. [Best Practices](#best-practices)

---

## ARCHITECTURE OVERVIEW

### Game Rendering Pipeline (What You Need to Know)

The Eaglercraft client uses this rendering order:

```
┌─────────────────────────────────────────────────────────────┐
│  Minecraft Game Loop (EntityRenderer.func_181560_a)         │
├─────────────────────────────────────────────────────────────┤
│  1. Update (updateRenderer)                                 │
│     ├─ FOV calculations                                     │
│     ├─ Torch flicker                                        │
│     └─ Smooth camera                                        │
│                                                              │
│  2. Input Handling (processKeyBinds, input events)          │
│     ├─ Player movement                                      │
│     ├─ Module activation checks                             │
│     └─ ClickGUI interaction                                 │
│                                                              │
│  3. World Rendering (renderWorld)                           │
│     ├─ Terrain                                              │
│     ├─ Entities                                             │
│     └─ Particles                                            │
│                                                              │
│  4. GUI Rendering (renderGameOverlay in GuiIngame)          │
│     ├─ HUD elements (health, hunger, etc)                  │
│     ├─ MODULE RENDERING (Your modules here)                │
│     └─ ClickGUI OVERLAY RENDERING (Your GUI here)          │
│                                                              │
│  5. Screen Rendering (if currentScreen != null)             │
│     └─ Menu screens, inventory, pause menu                  │
└─────────────────────────────────────────────────────────────┘
```

### Key Files You'll Be Modifying

```
src/game/java/net/minecraft/client/
├─ Minecraft.java                    (Main game loop reference)
├─ gui/
│  ├─ GuiIngame.java                 (In-game GUI rendering - MODULE RENDER HOOK)
│  ├─ GuiScreen.java                 (Base for all GUI screens)
│  └─ ScaledResolution.java           (Screen coordinate scaling)
└─ renderer/
    └─ EntityRenderer.java           (Rendering pipeline)
    
YOUR CUSTOM CODE LOCATION (recommended):
src/game/java/net/lax1dude/eaglercraft/v1_8/client/
├─ ModuleManager.java                (Module registry and management)
├─ module/
│  ├─ Module.java                    (Base module interface)
│  ├─ EnumModuleCategory.java         (Module categories)
│  └─ [Your modules here]
├─ gui/
│  ├─ ClickGUI.java                  (Main GUI manager)
│  ├─ ClickGUIPanel.java              (Individual GUI panels)
│  ├─ ClickGUIButton.java             (Toggle buttons)
│  └─ ClickGUISlider.java             (Value sliders)
└─ event/
    ├─ Event.java                    (Event base class)
    ├─ EventManager.java              (Event dispatcher)
    └─ [Event types]
```

---

## CORE MODULE SYSTEM ARCHITECTURE

### 1. The Module Base Class

A module is a feature/hack that can be toggled on/off. Every module extends from a base class:

```java
// src/game/java/net/lax1dude/eaglercraft/v1_8/client/module/Module.java

public abstract class Module {
    // BASIC PROPERTIES
    private String name;                    // Display name ("Fly", "Killaura")
    private String description;             // Short description
    private EnumModuleCategory category;    // Category (COMBAT, MOVEMENT, RENDER, etc)
    private boolean enabled = false;        // Is module active?
    private int keyCode = 0;                // Keybind to toggle (0 = none)
    
    // SETTINGS STORAGE
    protected List<Setting> settings = new ArrayList<>();  // Module-specific settings
    
    // Constructor
    public Module(String name, String description, EnumModuleCategory category) {
        this.name = name;
        this.description = description;
        this.category = category;
    }
    
    // ============================================
    // LIFECYCLE METHODS (called by ModuleManager)
    // ============================================
    
    /**
     * Called once when module is enabled
     * Used for initialization, hook setup, etc
     */
    public void onEnable() {
        // Override in your module
        // Example: register event listeners, start timers
    }
    
    /**
     * Called every frame when module is ENABLED
     * This is where your module logic runs
     * 
     * CALLED FROM: GuiIngame.renderGameOverlay (game thread)
     * CALLED EVERY TICK (20 ticks per second)
     */
    public void onUpdate() {
        // Override in your module
        // Example: apply velocity, check players, etc
    }
    
    /**
     * Called once when module is disabled
     * Used for cleanup
     */
    public void onDisable() {
        // Override in your module
        // Example: restore settings, cleanup threads
    }
    
    /**
     * Called every frame for rendering HUD elements
     * Only add GUI renderings here, never gameplay logic
     * 
     * CALLED FROM: GuiIngame.renderGameOverlay (after 3D rendering)
     * GUI rendering happens in 2D (screen coordinates)
     */
    public void onRender(int mouseX, int mouseY, float partialTicks) {
        // Override in your module
        // Example: draw text, draw shapes on HUD
    }
    
    // ============================================
    // EVENT HANDLERS (optional, if using event system)
    // ============================================
    
    /**
     * Called when a key is pressed
     * Only if module is enabled and EventManager is implemented
     */
    public void onKeyPress(int keyCode, char character) {
        // Override if needed
    }
    
    /**
     * Called before the world is rendered
     * Useful for manipulating render state
     */
    public void onPreRender(float partialTicks) {
        // Override if needed
    }
    
    /**
     * Called after the world is rendered
     * Useful for drawing 3D shapes in the world
     */
    public void onPostRender(float partialTicks) {
        // Override if needed
    }
    
    // ============================================
    // TOGGLE & CONTROL METHODS
    // ============================================
    
    /**
     * Toggle the module on/off
     */
    public synchronized void toggle() {
        setEnabled(!enabled);
    }
    
    /**
     * Set enabled state with proper lifecycle
     */
    public synchronized void setEnabled(boolean state) {
        if (enabled == state) return;  // Already in this state
        
        enabled = state;
        
        if (enabled) {
            onEnable();
        } else {
            onDisable();
        }
        
        // Notify ClickGUI of state change
        if (ClickGUI.instance != null) {
            ClickGUI.instance.onModuleStateChanged(this);
        }
    }
    
    // ============================================
    // GETTERS
    // ============================================
    
    public String getName() { return name; }
    public String getDescription() { return description; }
    public EnumModuleCategory getCategory() { return category; }
    public boolean isEnabled() { return enabled; }
    public int getKeyCode() { return keyCode; }
    public List<Setting> getSettings() { return settings; }
    
    public synchronized void setKeyCode(int code) { 
        this.keyCode = code; 
    }
    
    // ============================================
    // SETTINGS MANAGEMENT
    // ============================================
    
    /**
     * Add a setting to this module
     * This creates a configurable value that appears in the ClickGUI
     */
    protected void addSetting(Setting setting) {
        settings.add(setting);
    }
    
    /**
     * Get a setting by name
     */
    public Setting getSetting(String settingName) {
        for (Setting s : settings) {
            if (s.getName().equals(settingName)) return s;
        }
        return null;
    }
}
```

### 2. Module Categories

Organize modules into categories:

```java
// src/game/java/net/lax1dude/eaglercraft/v1_8/client/module/EnumModuleCategory.java

public enum EnumModuleCategory {
    COMBAT("Combat", 0xFF0000),           // Red
    MOVEMENT("Movement", 0x00FF00),       // Green
    RENDER("Render", 0x0000FF),           // Blue
    UTILITY("Utility", 0xFFFF00),         // Yellow
    PLAYER("Player", 0xFF00FF),           // Magenta
    WORLD("World", 0x00FFFF),             // Cyan
    MISC("Miscellaneous", 0x888888);      // Gray
    
    private String displayName;
    private int color;
    
    EnumModuleCategory(String displayName, int color) {
        this.displayName = displayName;
        this.color = color;
    }
    
    public String getDisplayName() { return displayName; }
    public int getColor() { return color; }
}
```

### 3. The ModuleManager

Central registry that manages all modules:

```java
// src/game/java/net/lax1dude/eaglercraft/v1_8/client/ModuleManager.java

public class ModuleManager {
    private static ModuleManager instance;
    
    private List<Module> modules = new ArrayList<>();
    private Map<EnumModuleCategory, List<Module>> modulesByCategory = new HashMap<>();
    
    private ModuleManager() {
        for (EnumModuleCategory cat : EnumModuleCategory.values()) {
            modulesByCategory.put(cat, new ArrayList<>());
        }
    }
    
    public static ModuleManager getInstance() {
        if (instance == null) {
            instance = new ModuleManager();
        }
        return instance;
    }
    
    /**
     * Register a module
     * Call this during client startup
     */
    public void registerModule(Module module) {
        modules.add(module);
        modulesByCategory.get(module.getCategory()).add(module);
    }
    
    /**
     * Get all modules
     */
    public List<Module> getModules() {
        return new ArrayList<>(modules);
    }
    
    /**
     * Get modules by category
     */
    public List<Module> getModulesByCategory(EnumModuleCategory category) {
        return new ArrayList<>(modulesByCategory.get(category));
    }
    
    /**
     * Get a module by name
     */
    public Module getModule(String name) {
        for (Module m : modules) {
            if (m.getName().equalsIgnoreCase(name)) {
                return m;
            }
        }
        return null;
    }
    
    /**
     * Called every frame - updates all ENABLED modules
     * HOOK LOCATION: GuiIngame.renderGameOverlay (in main update section)
     */
    public void updateModules() {
        for (Module module : modules) {
            if (module.isEnabled()) {
                try {
                    module.onUpdate();
                } catch (Exception e) {
                    e.printStackTrace();
                    // Disable broken module
                    module.setEnabled(false);
                }
            }
        }
    }
    
    /**
     * Called every frame for rendering
     * HOOK LOCATION: GuiIngame.renderGameOverlay (in GUI render section)
     */
    public void renderModules(int mouseX, int mouseY, float partialTicks) {
        for (Module module : modules) {
            if (module.isEnabled()) {
                try {
                    module.onRender(mouseX, mouseY, partialTicks);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
    }
    
    /**
     * Check key presses for module toggles
     * HOOK LOCATION: Minecraft.processKeyBinds
     */
    public void checkKeyBinds(int keyCode) {
        for (Module module : modules) {
            if (module.getKeyCode() == keyCode) {
                module.toggle();
            }
        }
    }
}
```

### 4. Settings System

Allow modules to have configurable values:

```java
// src/game/java/net/lax1dude/eaglercraft/v1_8/client/Setting.java

public class Setting {
    private String name;
    private Object value;
    private Object defaultValue;
    private Object minValue;
    private Object maxValue;
    private Module parentModule;
    
    // Constructor for boolean
    public Setting(String name, Module module, boolean defaultValue) {
        this.name = name;
        this.parentModule = module;
        this.defaultValue = defaultValue;
        this.value = defaultValue;
    }
    
    // Constructor for number (with min/max)
    public Setting(String name, Module module, double value, double min, double max) {
        this.name = name;
        this.parentModule = module;
        this.defaultValue = value;
        this.value = value;
        this.minValue = min;
        this.maxValue = max;
    }
    
    // Constructor for string
    public Setting(String name, Module module, String defaultValue) {
        this.name = name;
        this.parentModule = module;
        this.defaultValue = defaultValue;
        this.value = defaultValue;
    }
    
    // ============================================
    // GETTERS & SETTERS
    // ============================================
    
    public String getName() { return name; }
    public Object getValue() { return value; }
    public void setValue(Object val) { this.value = val; }
    
    public boolean getBooleanValue() { 
        return (Boolean) value; 
    }
    
    public double getNumberValue() { 
        return ((Number) value).doubleValue(); 
    }
    
    public String getStringValue() { 
        return (String) value; 
    }
    
    public Object getMinValue() { return minValue; }
    public Object getMaxValue() { return maxValue; }
    public Module getParentModule() { return parentModule; }
    
    /**
     * Return the type of this setting
     */
    public Class<?> getType() {
        return value.getClass();
    }
    
    public boolean isNumberSetting() {
        return value instanceof Number;
    }
    
    public boolean isBooleanSetting() {
        return value instanceof Boolean;
    }
    
    public boolean isStringSetting() {
        return value instanceof String;
    }
}
```

---

## THE EVENT SYSTEM

If you want a more advanced module system with events, you can use this:

```java
// src/game/java/net/lax1dude/eaglercraft/v1_8/client/event/Event.java

public class Event {
    public enum Type {
        PRE_RENDER,     // Before world render
        POST_RENDER,    // After world render
        KEY,            // Key pressed
        MOUSE,          // Mouse moved/clicked
        CHAT,           // Chat message
        PLAYER_UPDATE,  // Player tick update
        ENTITY_RENDER,  // Entity being rendered
        ATTACK,         // Right before attack
    }
    
    private Type type;
    private boolean cancelled = false;
    
    public Event(Type type) {
        this.type = type;
    }
    
    public Type getType() { return type; }
    public boolean isCancelled() { return cancelled; }
    public void setCancelled(boolean state) { this.cancelled = state; }
}

// Example implementations:
public class KeyEvent extends Event {
    private int keyCode;
    private char character;
    
    public KeyEvent(int keyCode, char character) {
        super(Type.KEY);
        this.keyCode = keyCode;
        this.character = character;
    }
    
    public int getKeyCode() { return keyCode; }
    public char getCharacter() { return character; }
}

// EventManager to dispatch events
public class EventManager {
    private static EventManager instance;
    private Map<Event.Type, List<Module>> listeners = new HashMap<>();
    
    public static EventManager getInstance() {
        if (instance == null) instance = new EventManager();
        return instance;
    }
    
    /**
     * Fire an event to all listening modules
     */
    public void fireEvent(Event event) {
        List<Module> listeners = this.listeners.get(event.getType());
        if (listeners != null) {
            for (Module module : listeners) {
                if (module.isEnabled()) {
                    // Module responds to event
                    // Implementation depends on event type
                }
            }
        }
    }
    
    /**
     * Register a module to listen to events
     */
    public void addEventListener(Module module, Event.Type type) {
        listeners.computeIfAbsent(type, k -> new ArrayList<>()).add(module);
    }
}
```

---

## CLICKGUI ARCHITECTURE

### 1. ClickGUI Manager

```java
// src/game/java/net/lax1dude/eaglercraft/v1_8/client/gui/ClickGUI.java

public class ClickGUI {
    public static ClickGUI instance;
    
    private Minecraft mc = Minecraft.getMinecraft();
    private boolean open = false;
    private int guiKey = Keyboard.KEY_Y;  // Press Y to toggle GUI
    
    private List<ClickGUIPanel> panels = new ArrayList<>();
    private ClickGUIPanel draggingPanel = null;
    private int dragOffsetX = 0;
    private int dragOffsetY = 0;
    
    // Rendering variables
    private ScaledResolution scaledResolution;
    private int screenWidth;
    private int screenHeight;
    private int mouseX = 0;
    private int mouseY = 0;
    
    // Color scheme
    private static final int COLOR_DARK_BG = 0xFF1a1a1a;   // Dark gray background
    private static final int COLOR_PANEL_HEADER = 0xFF2d2d2d; // Slightly lighter gray
    private static final int COLOR_BUTTON_ENABLED = 0xFF00aa00;  // Green
    private static final int COLOR_BUTTON_DISABLED = 0xFF4d4d4d; // Medium gray
    private static final int COLOR_TEXT = 0xFFFFFFFF;       // White
    private static final int COLOR_TEXT_SECONDARY = 0xFFB0B0B0; // Light gray
    
    public ClickGUI() {
        instance = this;
        initializePanels();
    }
    
    /**
     * Initialize GUI panels for each module category
     */
    private void initializePanels() {
        ModuleManager manager = ModuleManager.getInstance();
        int panelX = 10;
        
        for (EnumModuleCategory category : EnumModuleCategory.values()) {
            ClickGUIPanel panel = new ClickGUIPanel(
                category.getDisplayName(),
                panelX, 10,         // x, y position
                150, 300,           // width, height
                category
            );
            
            // Add buttons for each module in this category
            for (Module module : manager.getModulesByCategory(category)) {
                panel.addModule(module);
            }
            
            panels.add(panel);
            panelX += 160;  // Offset each panel
        }
    }
    
    /**
     * Called from GuiIngame.renderGameOverlay at the END of rendering
     * This is where we render the GUI elements on top of everything
     */
    public void render(int mouseX, int mouseY, float partialTicks) {
        if (!open) return;
        
        this.mouseX = mouseX;
        this.mouseY = mouseY;
        this.scaledResolution = mc.scaledResolution;
        this.screenWidth = scaledResolution.getScaledWidth();
        this.screenHeight = scaledResolution.getScaledHeight();
        
        // Draw semi-transparent background
        drawRect(0, 0, screenWidth, screenHeight, 0x60000000);
        
        // Render all panels
        for (ClickGUIPanel panel : panels) {
            panel.render(mouseX, mouseY, partialTicks);
        }
    }
    
    /**
     * Called when mouse is clicked
     */
    public void mouseClicked(int mouseX, int mouseY, int button) {
        if (!open) return;
        
        for (ClickGUIPanel panel : panels) {
            // Check if panel header is clicked (for dragging)
            if (panel.isMouseOverHeader(mouseX, mouseY)) {
                if (button == 0) {  // Left click
                    draggingPanel = panel;
                    dragOffsetX = mouseX - panel.getX();
                    dragOffsetY = mouseY - panel.getY();
                    return;
                }
            }
            
            // Let panel handle clicks
            panel.mouseClicked(mouseX, mouseY, button);
        }
    }
    
    /**
     * Called when mouse is released
     */
    public void mouseReleased(int mouseX, int mouseY, int button) {
        if (button == 0) {
            draggingPanel = null;
        }
        
        for (ClickGUIPanel panel : panels) {
            panel.mouseReleased(mouseX, mouseY, button);
        }
    }
    
    /**
     * Called every frame while mouse button is held
     */
    public void mouseDragged(int mouseX, int mouseY) {
        if (draggingPanel != null) {
            draggingPanel.setPosition(
                mouseX - dragOffsetX,
                mouseY - dragOffsetY
            );
        }
        
        for (ClickGUIPanel panel : panels) {
            panel.mouseDragged(mouseX, mouseY);
        }
    }
    
    /**
     * Called when a module's enabled state changes
     */
    public void onModuleStateChanged(Module module) {
        // Update all panels
        for (ClickGUIPanel panel : panels) {
            panel.onModuleStateChanged(module);
        }
    }
    
    /**
     * Toggle GUI visibility
     */
    public void toggle() {
        open = !open;
        if (!open) {
            draggingPanel = null;  // Stop dragging
        }
    }
    
    // Getters
    public boolean isOpen() { return open; }
    public void setOpen(boolean state) { open = state; }
    
    // Static utility to draw rectangles (exists in Gui class)
    public static void drawRect(int x1, int y1, int x2, int y2, int color) {
        int xMin = Math.min(x1, x2);
        int yMin = Math.min(y1, y2);
        int xMax = Math.max(x1, x2);
        int yMax = Math.max(y1, y2);
        
        GlStateManager.enableBlend();
        GlStateManager.tryBlendFuncSeparate(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA, 1, 0);
        
        Tessellator tessellator = Tessellator.getInstance();
        WorldRenderer wr = tessellator.getWorldRenderer();
        
        wr.begin(7, DefaultVertexFormats.POSITION_COLOR);
        wr.pos(xMin, yMax, 0).color((color >> 16) & 0xFF, (color >> 8) & 0xFF, color & 0xFF, (color >> 24) & 0xFF).endVertex();
        wr.pos(xMax, yMax, 0).color((color >> 16) & 0xFF, (color >> 8) & 0xFF, color & 0xFF, (color >> 24) & 0xFF).endVertex();
        wr.pos(xMax, yMin, 0).color((color >> 16) & 0xFF, (color >> 8) & 0xFF, color & 0xFF, (color >> 24) & 0xFF).endVertex();
        wr.pos(xMin, yMin, 0).color((color >> 16) & 0xFF, (color >> 8) & 0xFF, color & 0xFF, (color >> 24) & 0xFF).endVertex();
        tessellator.draw();
        
        GlStateManager.disableBlend();
    }
    
    /**
     * Draw a string with shadow
     */
    public static void drawString(FontRenderer fontRenderer, String text, int x, int y, int color) {
        fontRenderer.drawStringWithShadow(text, x, y, color);
    }
}
```

### 2. ClickGUI Panel (Category Container)

```java
// src/game/java/net/lax1dude/eaglercraft/v1_8/client/gui/ClickGUIPanel.java

public class ClickGUIPanel {
    private String title;
    private int x, y, width, height;
    private EnumModuleCategory category;
    private List<ClickGUIButton> buttons = new ArrayList<>();
    private boolean expanded = true;
    private int headerHeight = 15;
    
    public ClickGUIPanel(String title, int x, int y, int width, int height, EnumModuleCategory category) {
        this.title = title;
        this.x = x;
        this.y = y;
        this.width = width;
        this.height = height;
        this.category = category;
    }
    
    /**
     * Add a module to this panel
     */
    public void addModule(Module module) {
        ClickGUIButton button = new ClickGUIButton(module, 5, headerHeight + 5 + (buttons.size() * 20));
        buttons.add(button);
    }
    
    /**
     * Called to render this panel and all its buttons
     */
    public void render(int mouseX, int mouseY, float partialTicks) {
        // Draw panel background
        ClickGUI.drawRect(x, y, x + width, y + height, 0xFF2d2d2d);
        
        // Draw panel border
        drawBorder(x, y, x + width, y + height, 0xFF888888);
        
        // Draw header
        ClickGUI.drawRect(x, y, x + width, y + headerHeight, category.getColor());
        
        // Draw title text
        ClickGUI.drawString(
            Minecraft.getMinecraft().fontRendererObj,
            title + (expanded ? "" : " [+]"),
            x + 3, y + 3,
            0xFFFFFFFF
        );
        
        if (expanded) {
            int currentY = y + headerHeight;
            for (ClickGUIButton button : buttons) {
                button.setY(currentY);
                button.render(mouseX, mouseY);
                currentY += 20;
            }
        }
    }
    
    /**
     * Called when mouse is clicked
     */
    public void mouseClicked(int mouseX, int mouseY, int button) {
        // Check if header is clicked (toggle expand)
        if (isMouseOverHeader(mouseX, mouseY) && button == 1) {  // Right click
            expanded = !expanded;
            return;
        }
        
        if (expanded) {
            for (ClickGUIButton btn : buttons) {
                btn.mouseClicked(mouseX, mouseY, button);
            }
        }
    }
    
    /**
     * Check if mouse is over header
     */
    public boolean isMouseOverHeader(int mouseX, int mouseY) {
        return mouseX >= x && mouseX <= x + width && 
               mouseY >= y && mouseY <= y + headerHeight;
    }
    
    // Other methods...
    public void mouseReleased(int mouseX, int mouseY, int button) {
        for (ClickGUIButton btn : buttons) {
            btn.mouseReleased(mouseX, mouseY, button);
        }
    }
    
    public void mouseDragged(int mouseX, int mouseY) {
        for (ClickGUIButton btn : buttons) {
            btn.mouseDragged(mouseX, mouseY);
        }
    }
    
    public void onModuleStateChanged(Module module) {
        for (ClickGUIButton btn : buttons) {
            if (btn.getModule() == module) {
                btn.onModuleStateChanged();
            }
        }
    }
    
    public void setPosition(int x, int y) {
        this.x = x;
        this.y = y;
    }
    
    public int getX() { return x; }
    public int getY() { return y; }
    
    private void drawBorder(int x1, int y1, int x2, int y2, int color) {
        // Draw 1px border
        ClickGUI.drawRect(x1, y1, x2, y1 + 1, color);      // Top
        ClickGUI.drawRect(x1, y2 - 1, x2, y2, color);      // Bottom
        ClickGUI.drawRect(x1, y1, x1 + 1, y2, color);      // Left
        ClickGUI.drawRect(x2 - 1, y1, x2, y2, color);      // Right
    }
}
```

### 3. ClickGUI Button (Module Toggle)

```java
// src/game/java/net/lax1dude/eaglercraft/v1_8/client/gui/ClickGUIButton.java

public class ClickGUIButton {
    private Module module;
    private int x, y;
    private int width = 150;
    private int height = 20;
    
    // Sub-components
    private ClickGUIToggle toggleButton;
    private ClickGUISlider[] sliders;
    
    public ClickGUIButton(Module module, int x, int y) {
        this.module = module;
        this.x = x;
        this.y = y;
        
        // Create toggle button (left side)
        this.toggleButton = new ClickGUIToggle(x + 5, y + 2, 100, 16, module);
        
        // Create sliders for settings (right side)
        List<Setting> settings = module.getSettings();
        sliders = new ClickGUISlider[settings.size()];
        for (int i = 0; i < settings.size(); i++) {
            Setting setting = settings.get(i);
            if (setting.isNumberSetting()) {
                sliders[i] = new ClickGUISlider(x + 5, y + 25 + (i * 20), 140, 15, setting);
            }
        }
    }
    
    /**
     * Render this button (and its settings if expanded)
     */
    public void render(int mouseX, int mouseY) {
        // Draw button background
        int bgColor = module.isEnabled() ? 0xFF2d5d2d : 0xFF4d4d4d;
        ClickGUI.drawRect(x, y, x + width, y + height, bgColor);
        
        // Draw module name
        int textColor = module.isEnabled() ? 0xFF00FF00 : 0xFFAAAAAA;
        ClickGUI.drawString(
            Minecraft.getMinecraft().fontRendererObj,
            module.getName(),
            x + 5, y + 5,
            textColor
        );
        
        // Draw enabled indicator
        String statusText = module.isEnabled() ? "[ON]" : "[OFF]";
        ClickGUI.drawString(
            Minecraft.getMinecraft().fontRendererObj,
            statusText,
            x + width - 30, y + 5,
            textColor
        );
        
        // Render toggle button (not typically needed if clicking the button toggles)
        // toggleButton.render(mouseX, mouseY);
        
        // Render sliders if module has settings
        if (module.isEnabled()) {
            for (ClickGUISlider slider : sliders) {
                if (slider != null) {
                    slider.render(mouseX, mouseY);
                }
            }
        }
    }
    
    /**
     * Called when mouse clicks on this button
     */
    public void mouseClicked(int mouseX, int mouseY, int button) {
        // Check if click is on the button itself
        if (mouseX >= x && mouseX <= x + width && 
            mouseY >= y && mouseY <= y + height) {
            if (button == 0) {  // Left click
                module.toggle();  // Toggle the module
                return;
            }
        }
        
        // Check sliders
        for (ClickGUISlider slider : sliders) {
            if (slider != null) {
                slider.mouseClicked(mouseX, mouseY, button);
            }
        }
    }
    
    public void mouseReleased(int mouseX, int mouseY, int button) {
        for (ClickGUISlider slider : sliders) {
            if (slider != null) {
                slider.mouseReleased(mouseX, mouseY, button);
            }
        }
    }
    
    public void mouseDragged(int mouseX, int mouseY) {
        for (ClickGUISlider slider : sliders) {
            if (slider != null) {
                slider.mouseDragged(mouseX, mouseY);
            }
        }
    }
    
    public void onModuleStateChanged() {
        // Update visual state
    }
    
    public Module getModule() { return module; }
    public void setY(int y) { this.y = y; }
}
```

---

## STEP-BY-STEP IMPLEMENTATION

### STEP 1: Create the directory structure

```bash
# Create custom client code directories
mkdir -p src/game/java/net/lax1dude/eaglercraft/v1_8/client
mkdir -p src/game/java/net/lax1dude/eaglercraft/v1_8/client/module
mkdir -p src/game/java/net/lax1dude/eaglercraft/v1_8/client/gui
mkdir -p src/game/java/net/lax1dude/eaglercraft/v1_8/client/event
```

### STEP 2: Create all the base classes

Create files in: `src/game/java/net/lax1dude/eaglercraft/v1_8/client/`

1. `Module.java` - Module base class
2. `Setting.java` - Settings system
3. `ModuleManager.java` - Module registry
4. `EnumModuleCategory.java` - Categories

Create files in: `src/game/java/net/lax1dude/eaglercraft/v1_8/client/gui/`

1. `ClickGUI.java` - Main GUI manager
2. `ClickGUIPanel.java` - Panel for each category
3. `ClickGUIButton.java` - Button for each module
4. `ClickGUISlider.java` - Slider for number settings

### STEP 3: Hook into the game loop

**In `GuiIngame.java` - `renderGameOverlay` method:**

Find the method and locate where `drawHotbar` or similar is called. Add these lines:

```java
// At the BEGINNING of renderGameOverlay (for updates)
ModuleManager.getInstance().updateModules();

// At the END of renderGameOverlay (for rendering)
int i = scaledresolution.getScaledWidth();
int j = scaledresolution.getScaledHeight();
ModuleManager.getInstance().renderModules(
    Mouse.getX() * i / mc.displayWidth,
    j - Mouse.getY() * j / mc.displayHeight - 1,
    partialTicks
);
if (ClickGUI.instance != null) {
    ClickGUI.instance.render(
        Mouse.getX() * i / mc.displayWidth,
        j - Mouse.getY() * j / mc.displayHeight - 1,
        partialTicks
    );
}
```

**In `Minecraft.java` - `processKeyBinds` method:**

Add key handling for ClickGUI and modules:

```java
// In processKeyBinds loop, after other key checks:

// Check for ClickGUI toggle key
if (Keyboard.getEventKeyState()) {
    if (Keyboard.getEventKey() == Keyboard.KEY_Y) {  // Press Y to open GUI
        if (ClickGUI.instance != null) {
            ClickGUI.instance.toggle();
        }
    }
    
    // Check module key binds
    ModuleManager.getInstance().checkKeyBinds(Keyboard.getEventKey());
}
```

**In `GuiIngame.java` - add mouse handlers:**

```java
// Add these methods to GuiIngame
public void mouseClicked(int mouseX, int mouseY, int button) throws IOException {
    if (ClickGUI.instance != null && ClickGUI.instance.isOpen()) {
        ClickGUI.instance.mouseClicked(mouseX, mouseY, button);
    }
}

public void mouseReleased(int mouseX, int mouseY, int button) {
    if (ClickGUI.instance != null) {
        ClickGUI.instance.mouseReleased(mouseX, mouseY, button);
    }
}

public void mouseDragged(int mouseX, int mouseY) {
    if (ClickGUI.instance != null) {
        ClickGUI.instance.mouseDragged(mouseX, mouseY);
    }
}
```

### STEP 4: Create your first module

Example: **Speed Module** (Movement category)

```java
// src/game/java/net/lax1dude/eaglercraft/v1_8/client/module/impl/ModuleSpeed.java

public class ModuleSpeed extends Module {
    private EntityPlayerSP player;
    private Setting speedMultiplier;
    
    public ModuleSpeed() {
        super("Speed", "Move faster", EnumModuleCategory.MOVEMENT);
        
        // Add settings
        speedMultiplier = new Setting("Multiplier", this, 1.5, 1.0, 3.0);
        addSetting(speedMultiplier);
    }
    
    @Override
    public void onEnable() {
        this.player = Minecraft.getMinecraft().thePlayer;
        if (player != null) {
            player.motionX *= 1.2;
            player.motionZ *= 1.2;
        }
    }
    
    @Override
    public void onUpdate() {
        EntityPlayerSP player = Minecraft.getMinecraft().thePlayer;
        if (player == null) return;
        
        // Get movement direction
        double multiplier = speedMultiplier.getNumberValue();
        
        if (player.movementInput.moveForward != 0 || player.movementInput.moveStrafe != 0) {
            // Apply speed boost
            double yaw = player.rotationYaw * Math.PI / 180.0;
            
            double speed = multiplier * 0.05;  // Base speed
            double sin = Math.sin(yaw);
            double cos = Math.cos(yaw);
            
            double forward = player.movementInput.moveForward;
            double strafe = player.movementInput.moveStrafe;
            
            player.motionX += (forward * cos - strafe * sin) * speed;
            player.motionZ += (strafe * cos + forward * sin) * speed;
        }
    }
    
    @Override
    public void onRender(int mouseX, int mouseY, float partialTicks) {
        // Optional: draw HUD text showing speed is active
        FontRenderer font = Minecraft.getMinecraft().fontRendererObj;
        font.drawStringWithShadow("Speed: " + 
            String.format("%.1f", speedMultiplier.getNumberValue()) + "x",
            10, 10, 0xFF00FF00);
    }
    
    @Override
    public void onDisable() {
        // Restore normal speed
    }
}
```

### STEP 5: Register modules in ModuleManager

In your client initialization code:

```java
// Call this on client startup (like in Minecraft constructor or first frame)
public void initializeClientMods() {
    ModuleManager manager = ModuleManager.getInstance();
    
    // Register all modules
    manager.registerModule(new ModuleSpeed());
    manager.registerModule(new ModuleXray());
    manager.registerModule(new ModuleKillaura());
    // ... etc
    
    // Initialize GUI
    ClickGUI.instance = new ClickGUI();
}
```

---

## COMPLETE CODE EXAMPLES

### Example 1: Xray Module (Render category)

```java
public class ModuleXray extends Module {
    private boolean blocksHidden = false;
    private List<Block> hiddenBlocks = new ArrayList<>();
    
    public ModuleXray() {
        super("Xray", "See through blocks", EnumModuleCategory.RENDER);
        
        Setting oreOnly = new Setting("OreOnly", this, true);
        addSetting(oreOnly);
    }
    
    @Override
    public void onEnable() {
        blocksHidden = false;
        // Hide blocks
        if (getSetting("OreOnly").getBooleanValue()) {
            // Only hide non-ore blocks
            hideNonOreBlocks();
        } else {
            // Hide most blocks but leave ores visible
            hideAllButOres();
        }
        blocksHidden = true;
    }
    
    @Override
    public void onDisable() {
        // Restore block rendering
        if (blocksHidden) {
            restoreBlocks();
            blocksHidden = false;
        }
    }
    
    private void hideNonOreBlocks() {
        // This is pseudo-code - actual implementation would need to 
        // modify block models or use a shader
        Minecraft mc = Minecraft.getMinecraft();
        if (mc.theWorld == null) return;
        
        // In reality, you'd modify the texture or model rendering
        // OR create an overlay shader that renders only ores
    }
    
    private void hideAllButOres() {
        // Similar implementation
    }
    
    private void restoreBlocks() {
        // Restore default rendering
    }
}
```

### Example 2: Killaura Module (Combat category)

```java
public class ModuleKillaura extends Module {
    private EntityPlayer targetEntity = null;
    private Setting range;
    private Setting attackCooldown;
    private long lastAttackTime = 0;
    
    public ModuleKillaura() {
        super("Killaura", "Auto-attack nearby players", EnumModuleCategory.COMBAT);
        
        range = new Setting("Range", this, 6.0, 3.0, 15.0);
        attackCooldown = new Setting("Cooldown", this, 10, 1, 20);
        
        addSetting(range);
        addSetting(attackCooldown);
    }
    
    @Override
    public void onUpdate() {
        Minecraft mc = Minecraft.getMinecraft();
        EntityPlayerSP player = mc.thePlayer;
        WorldClient world = mc.theWorld;
        
        if (player == null || world == null) return;
        
        // Find nearby players
        double searchRange = range.getNumberValue();
        EntityPlayer closestPlayer = null;
        double closestDistance = searchRange;
        
        List<EntityPlayer> players = world.playerEntities;
        for (EntityPlayer p : players) {
            if (p == player) continue;  // Don't target self
            if (p.getHealth() <= 0) continue;  // Don't target dead players
            
            double dist = player.getDistanceToEntity(p);
            if (dist < closestDistance) {
                closestDistance = dist;
                closestPlayer = p;
            }
        }
        
        targetEntity = closestPlayer;
        
        // Attack if we have a target and cooldown is ready
        if (targetEntity != null) {
            long now = System.currentTimeMillis();
            long cooldown = (long) attackCooldown.getNumberValue() * 50;  // Convert to ms
            
            if (now - lastAttackTime >= cooldown) {
                // Face the target
                faceEntity(targetEntity);
                
                // Attack
                mc.playerController.attackEntity(player, targetEntity);
                player.swingItem();
                
                lastAttackTime = now;
            }
        }
    }
    
    @Override
    public void onRender(int mouseX, int mouseY, float partialTicks) {
        if (targetEntity != null) {
            FontRenderer font = Minecraft.getMinecraft().fontRendererObj;
            font.drawStringWithShadow("Target: " + targetEntity.getName(),
                10, 25, 0xFFFF0000);
        }
    }
    
    private void faceEntity(Entity entity) {
        Minecraft mc = Minecraft.getMinecraft();
        EntityPlayerSP player = mc.thePlayer;
        
        // Get target position
        double dx = entity.posX - player.posX;
        double dy = entity.posY + entity.getEyeHeight() - (player.posY + player.getEyeHeight());
        double dz = entity.posZ - player.posZ;
        
        double dist = Math.sqrt(dx*dx + dz*dz);
        
        // Calculate angles
        float yaw = (float) Math.atan2(dz, dx) * 180 / (float)Math.PI - 90;
        float pitch = (float) -Math.atan2(dy, dist) * 180 / (float)Math.PI;
        
        // Apply with some smoothing to avoid suspicion
        player.rotationYaw += (yaw - player.rotationYaw) * 0.5f;
        player.rotationPitch = pitch;
    }
}
```

---

## INTEGRATION POINTS

### Where to Hook Into the Game Loop

#### 1. **Module Updates** (game logic)
- **Location**: `GuiIngame.renderGameOverlay()` - beginning
- **Frequency**: Every frame (60 FPS typically)
- **Purpose**: Game logic updates happen here
- **Safe to modify world, player, entities**: YES

```java
// In GuiIngame.renderGameOverlay, near the beginning:
ModuleManager.getInstance().updateModules();
```

#### 2. **Module Rendering** (HUD overlay)
- **Location**: `GuiIngame.renderGameOverlay()` - end
- **Frequency**: Every frame
- **Purpose**: Render 2D HUD elements only
- **Render state**: Screen coordinates, 2D

```java
// In GuiIngame.renderGameOverlay, at the end:
ModuleManager.getInstance().renderModules(x, y, partialTicks);
```

#### 3. **Key Input Handling**
- **Location**: `Minecraft.processKeyBinds()`
- **Frequency**: When key is pressed
- **Purpose**: Handle module toggling via keybinds

```java
// In Minecraft.processKeyBinds loop:
if (Keyboard.getEventKeyState()) {
    ModuleManager.getInstance().checkKeyBinds(Keyboard.getEventKey());
}
```

#### 4. **Mouse Input Handling**
- **Location**: Add methods to `GuiIngame`
- **Frequency**: When mouse clicks
- **Purpose**: Handle GUI clicks

```java
// Add to GuiIngame:
public void handleMouseClick(int mouseX, int mouseY, int button) {
    ClickGUI.instance.mouseClicked(mouseX, mouseY, button);
}
```

#### 5. **Pre-Render Hook** (optional, for advanced modules)
- **Location**: `EntityRenderer.func_181560_a()` - before `renderWorld`
- **Frequency**: Every frame
- **Purpose**: Modifications to rendering state before world render
- **Example**: ESP boxes drawn in 3D space

```java
// In EntityRenderer before renderWorld call:
EventManager.getInstance().fireEvent(new PreRenderEvent());
```

#### 6. **Post-Render Hook** (optional)
- **Location**: `EntityRenderer.func_181560_a()` - after `renderWorld`
- **Frequency**: Every frame
- **Purpose**: Draw 3D shapes in world space

```java
// In EntityRenderer after renderWorld call:
EventManager.getInstance().fireEvent(new PostRenderEvent());
```

---

## BEST PRACTICES

### 1. **Module Safety**
- Always check if Minecraft.getMinecraft() components are null
- Use try-catch in onUpdate/onRender to prevent crashes
- Disable module automatically if it errors

```java
@Override
public void onUpdate() {
    try {
        // Code here
    } catch (Exception e) {
        e.printStackTrace();
        this.setEnabled(false);  // Auto-disable on error
    }
}
```

### 2. **Performance**
- Don't create new objects every frame
- Cache important objects
- Use int division instead of float when possible
- Avoid nested loops in onUpdate

```java
// BAD - creates list every frame
public void onUpdate() {
    List<EntityPlayer> players = new ArrayList<>(
        Minecraft.getMinecraft().theWorld.playerEntities
    );
}

// GOOD - cache it
private List<EntityPlayer> playerListCache;

public void onEnable() {
    playerListCache = Minecraft.getMinecraft().theWorld.playerEntities;
}

public void onUpdate() {
    // Use cached list
}
```

### 3. **Thread Safety**
- Minecraft.thePlayer can be null mid-transition
- Entity lists can change during iteration (use ArrayList copy)
- Use synchronized blocks if threading

```java
public synchronized void toggle() {
    setEnabled(!enabled);
}
```

### 4. **Settings Management**
- Use Settings for all configurable values
- Don't hard-code values
- Store settings to file for persistence

```java
// Create setting in constructor
Setting knockback = new Setting("Knockback", this, 1.0, 0.0, 2.0);

// Use in onUpdate
double kb = knockback.getNumberValue();
```

### 5. **Rendering**
- Only render in onRender() method, never in onUpdate()
- Use FontRenderer from Minecraft for text
- Remember Y coordinate is flipped on GUI rendering

```java
// CORRECT
public void onRender(int mouseX, int mouseY, float partialTicks) {
    Minecraft mc = Minecraft.getMinecraft();
    mc.fontRendererObj.drawStringWithShadow("Text", 10, 10, 0xFFFFFFFF);
}

// WRONG - don't do in onUpdate
public void onUpdate() {
    // rendering code - WRONG!
}
```

### 6. **Module Dependencies**
If a module needs data from another module, use ModuleManager to get it:

```java
public void onUpdate() {
    Module speedModule = ModuleManager.getInstance().getModule("Speed");
    if (speedModule != null && speedModule.isEnabled()) {
        // Do something different if Speed is enabled
    }
}
```

---

## SUMMARY OF HOW EVERYTHING WORKS

**The complete flow:**

1. **Game starts** → ModuleManager is created → ClickGUI is created with panels
2. **Every frame**:
   - Input processing: Check if Y is pressed → toggle ClickGUI, check module keybinds
   - `GuiIngame.renderGameOverlay()` is called
     - Calls `ModuleManager.updateModules()` 
       - Which calls `module.onUpdate()` for each enabled module
     - Calls `ModuleManager.renderModules()` for HUD rendering
       - Which calls `module.onRender()` for each enabled module
     - Calls `ClickGUI.render()` to draw GUI panels and buttons
3. **User clicks on ClickGUI button**:
   - `ClickGUIButton.mouseClicked()` is called
   - Calls `module.toggle()`
   - Module's `onEnable()` or `onDisable()` is called
   - Button visually updates
4. **User drags GUI panel**:
   - `ClickGUIPanel` position updates
   - GUI redraws at new position next frame
5. **User changes slider value**:
   - `ClickGUISlider` updates its `Setting` value
   - Module uses new value next `onUpdate()` call

**That's the complete system in operation!**

---

## NEXT STEPS

1. Create all base classes (Module, ModuleManager, Setting, ClickGUI, etc)
2. Hook into GuiIngame.renderGameOverlay  
3. Hook into Minecraft key handling
4. Create 3-5 simple modules to test (Speed, NoFall, Xray, etc)
5. Test the ClickGUI rendering and interaction
6. Expand with more complex modules using the same pattern
7. Add settings persistence (JSON/file saving)
8. Add module profiles (save/load sets of enabled modules with settings)

---

## COMMON ISSUES & SOLUTIONS

### Issue: GUI doesn't render
**Solution**: Verify hook is in GuiIngame.renderGameOverlay, check isOpen() is being called

### Issue: Modules don't update
**Solution**: Verify ModuleManager.updateModules() is being called every frame

### Issue: Module crashes the game
**Solution**: Add try-catch in onUpdate(), check for null pointers

### Issue: ClickGUI doesn't respond to clicks
**Solution**: Verify mouse input handlers are hooked, check if GUI is marked as open

---

**This guide should give you everything you need to implement a full ClickGUI + Module system!**
