# ClickGUI & Module System - QUICK REFERENCE & CODE SNIPPETS

## COPY-PASTE BOILERPLATE CLASSES

### 1. Module.java (Base Class)

```java
package net.lax1dude.eaglercraft.v1_8.client.module;

import java.util.ArrayList;
import java.util.List;
import net.lax1dude.eaglercraft.v1_8.client.Setting;

public abstract class Module {
    private String name;
    private String description;
    private EnumModuleCategory category;
    private boolean enabled = false;
    private int keyCode = 0;
    protected List<Setting> settings = new ArrayList<>();
    
    public Module(String name, String description, EnumModuleCategory category) {
        this.name = name;
        this.description = description;
        this.category = category;
    }
    
    public void onEnable() {}
    public void onUpdate() {}
    public void onDisable() {}
    public void onRender(int mouseX, int mouseY, float partialTicks) {}
    public void onKeyPress(int keyCode, char character) {}
    public void onPreRender(float partialTicks) {}
    public void onPostRender(float partialTicks) {}
    
    public synchronized void toggle() {
        setEnabled(!enabled);
    }
    
    public synchronized void setEnabled(boolean state) {
        if (enabled == state) return;
        enabled = state;
        if (enabled) {
            onEnable();
        } else {
            onDisable();
        }
    }
    
    public String getName() { return name; }
    public String getDescription() { return description; }
    public EnumModuleCategory getCategory() { return category; }
    public boolean isEnabled() { return enabled; }
    public int getKeyCode() { return keyCode; }
    public List<Setting> getSettings() { return settings; }
    public void setKeyCode(int code) { this.keyCode = code; }
    
    protected void addSetting(Setting setting) {
        settings.add(setting);
    }
    
    public Setting getSetting(String settingName) {
        for (Setting s : settings) {
            if (s.getName().equals(settingName)) return s;
        }
        return null;
    }
}
```

### 2. Setting.java (Configuration Values)

```java
package net.lax1dude.eaglercraft.v1_8.client;

import net.lax1dude.eaglercraft.v1_8.client.module.Module;

public class Setting {
    private String name;
    private Object value;
    private Object defaultValue;
    private Object minValue;
    private Object maxValue;
    private Module parentModule;
    
    public Setting(String name, Module module, boolean defaultValue) {
        this.name = name;
        this.parentModule = module;
        this.defaultValue = defaultValue;
        this.value = defaultValue;
    }
    
    public Setting(String name, Module module, double value, double min, double max) {
        this.name = name;
        this.parentModule = module;
        this.defaultValue = value;
        this.value = value;
        this.minValue = min;
        this.maxValue = max;
    }
    
    public Setting(String name, Module module, String defaultValue) {
        this.name = name;
        this.parentModule = module;
        this.defaultValue = defaultValue;
        this.value = defaultValue;
    }
    
    public String getName() { return name; }
    public Object getValue() { return value; }
    public void setValue(Object val) { this.value = val; }
    
    public boolean getBooleanValue() { return (Boolean) value; }
    public double getNumberValue() { return ((Number) value).doubleValue(); }
    public String getStringValue() { return (String) value; }
    
    public Object getMinValue() { return minValue; }
    public Object getMaxValue() { return maxValue; }
    public Module getParentModule() { return parentModule; }
    
    public Class<?> getType() { return value.getClass(); }
    
    public boolean isNumberSetting() { return value instanceof Number; }
    public boolean isBooleanSetting() { return value instanceof Boolean; }
    public boolean isStringSetting() { return value instanceof String; }
}
```

### 3. EnumModuleCategory.java

```java
package net.lax1dude.eaglercraft.v1_8.client.module;

public enum EnumModuleCategory {
    COMBAT("Combat", 0xFFAA0000),
    MOVEMENT("Movement", 0xFF00AA00),
    RENDER("Render", 0xFF0000AA),
    UTILITY("Utility", 0xFFAAAA00),
    PLAYER("Player", 0xFFAA00AA),
    WORLD("World", 0xFF00AAAA),
    MISC("Miscellaneous", 0xFF888888);
    
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

### 4. ModuleManager.java

```java
package net.lax1dude.eaglercraft.v1_8.client;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import net.lax1dude.eaglercraft.v1_8.client.module.EnumModuleCategory;
import net.lax1dude.eaglercraft.v1_8.client.module.Module;

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
        if (instance == null) instance = new ModuleManager();
        return instance;
    }
    
    public void registerModule(Module module) {
        modules.add(module);
        modulesByCategory.get(module.getCategory()).add(module);
    }
    
    public List<Module> getModules() {
        return new ArrayList<>(modules);
    }
    
    public List<Module> getModulesByCategory(EnumModuleCategory category) {
        return new ArrayList<>(modulesByCategory.get(category));
    }
    
    public Module getModule(String name) {
        for (Module m : modules) {
            if (m.getName().equalsIgnoreCase(name)) return m;
        }
        return null;
    }
    
    public void updateModules() {
        for (Module module : modules) {
            if (module.isEnabled()) {
                try {
                    module.onUpdate();
                } catch (Exception e) {
                    e.printStackTrace();
                    module.setEnabled(false);
                }
            }
        }
    }
    
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
    
    public void checkKeyBinds(int keyCode) {
        for (Module module : modules) {
            if (module.getKeyCode() == keyCode) {
                module.toggle();
            }
        }
    }
}
```

---

## CREATING SIMPLE MODULES

### Speed Module

```java
package net.lax1dude.eaglercraft.v1_8.client.module.impl;

import net.minecraft.client.Minecraft;
import net.minecraft.entity.player.EntityPlayerSP;
import net.lax1dude.eaglercraft.v1_8.client.Setting;
import net.lax1dude.eaglercraft.v1_8.client.module.EnumModuleCategory;
import net.lax1dude.eaglercraft.v1_8.client.module.Module;

public class ModuleSpeed extends Module {
    private Setting speedAmount;
    
    public ModuleSpeed() {
        super("Speed", "Move faster", EnumModuleCategory.MOVEMENT);
        speedAmount = new Setting("Speed", this, 1.5, 1.0, 3.0);
        addSetting(speedAmount);
    }
    
    @Override
    public void onUpdate() {
        EntityPlayerSP player = Minecraft.getMinecraft().thePlayer;
        if (player == null) return;
        
        double multiplier = speedAmount.getNumberValue();
        
        if (player.movementInput.moveForward != 0 || player.movementInput.moveStrafe != 0) {
            double yaw = player.rotationYaw * Math.PI / 180.0;
            double speed = multiplier * 0.05;
            double sin = Math.sin(yaw);
            double cos = Math.cos(yaw);
            
            double forward = player.movementInput.moveForward;
            double strafe = player.movementInput.moveStrafe;
            
            player.motionX += (forward * cos - strafe * sin) * speed;
            player.motionZ += (strafe * cos + forward * sin) * speed;
        }
    }
}
```

### No-Fall Module

```java
package net.lax1dude.eaglercraft.v1_8.client.module.impl;

import net.minecraft.client.Minecraft;
import net.minecraft.entity.player.EntityPlayerSP;
import net.lax1dude.eaglercraft.v1_8.client.module.EnumModuleCategory;
import net.lax1dude.eaglercraft.v1_8.client.module.Module;

public class ModuleNoFall extends Module {
    public ModuleNoFall() {
        super("NoFall", "Prevent fall damage", EnumModuleCategory.PLAYER);
    }
    
    @Override
    public void onUpdate() {
        EntityPlayerSP player = Minecraft.getMinecraft().thePlayer;
        if (player == null) return;
        
        if (player.fallDistance > 3.0F) {
            player.sendQueue.addToSendQueue(
                new C03PacketPlayer(true)  // "on ground" packet
            );
            player.fallDistance = 0.0F;
        }
    }
}
```

### Step Module (Walk up blocks without jumping)

```java
package net.lax1dude.eaglercraft.v1_8.client.module.impl;

import net.minecraft.client.Minecraft;
import net.minecraft.entity.player.EntityPlayerSP;
import net.minecraft.util.BlockPos;
import net.minecraft.util.MovingObjectPosition;
import net.lax1dude.eaglercraft.v1_8.client.Setting;
import net.lax1dude.eaglercraft.v1_8.client.module.EnumModuleCategory;
import net.lax1dude.eaglercraft.v1_8.client.module.Module;

public class ModuleStep extends Module {
    private Setting height;
    
    public ModuleStep() {
        super("Step", "Step up blocks automatically", EnumModuleCategory.MOVEMENT);
        height = new Setting("Height", this, 1.0, 0.5, 3.0);
        addSetting(height);
    }
    
    @Override
    public void onUpdate() {
        EntityPlayerSP player = Minecraft.getMinecraft().thePlayer;
        if (player == null) return;
        
        MovingObjectPosition rayTrace = player.rayTrace(5.0, 1.0F);
        if (rayTrace == null) return;
        
        BlockPos blockPos = rayTrace.getBlockPos();
        if (blockPos == null) return;
        
        double stepHeight = height.getNumberValue();
        boolean isMoving = player.movementInput.moveForward != 0 || 
                          player.movementInput.moveStrafe != 0;
        
        if (isMoving && player.onGround) {
            player.motionY = 0.42;  // Step up
        }
    }
}
```

### Aimbot Module (Combat - Lock onto nearest player)

```java
package net.lax1dude.eaglercraft.v1_8.client.module.impl;

import net.minecraft.client.Minecraft;
import net.minecraft.entity.EntityLivingBase;
import net.minecraft.entity.player.EntityPlayer;
import net.minecraft.entity.player.EntityPlayerSP;
import net.minecraft.world.WorldClient;
import net.lax1dude.eaglercraft.v1_8.client.Setting;
import net.lax1dude.eaglercraft.v1_8.client.module.EnumModuleCategory;
import net.lax1dude.eaglercraft.v1_8.client.module.Module;

public class ModuleAimbot extends Module {
    private Setting range;
    private Setting smoothness;
    
    public ModuleAimbot() {
        super("Aimbot", "Lock onto nearest player", EnumModuleCategory.COMBAT);
        range = new Setting("Range", this, 8.0, 3.0, 15.0);
        smoothness = new Setting("Smoothness", this, 0.8, 0.1, 1.0);
        addSetting(range);
        addSetting(smoothness);
    }
    
    @Override
    public void onUpdate() {
        Minecraft mc = Minecraft.getMinecraft();
        EntityPlayerSP player = mc.thePlayer;
        WorldClient world = mc.theWorld;
        
        if (player == null || world == null) return;
        
        double searchRange = range.getNumberValue();
        EntityPlayer closestPlayer = null;
        double closestDistance = searchRange;
        
        for (EntityPlayer p : world.playerEntities) {
            if (p == player) continue;
            if (p.getHealth() <= 0) continue;
            
            double distance = player.getDistanceToEntity(p);
            if (distance < closestDistance) {
                closestDistance = distance;
                closestPlayer = p;
            }
        }
        
        if (closestPlayer != null) {
            double smooth = smoothness.getNumberValue();
            lookAtEntity(player, closestPlayer, (float) smooth);
        }
    }
    
    private void lookAtEntity(EntityPlayerSP player, EntityLivingBase entity, float smooth) {
        double dx = entity.posX - player.posX;
        double dy = entity.posY + entity.getEyeHeight() - (player.posY + player.getEyeHeight());
        double dz = entity.posZ - player.posZ;
        
        double dist = Math.sqrt(dx * dx + dz * dz);
        
        float targetYaw = (float) (Math.atan2(dz, dx) * 180 / Math.PI) - 90;
        float targetPitch = (float) (-Math.atan2(dy, dist) * 180 / Math.PI);
        
        // Smoothly interpolate angles
        player.rotationYaw += (targetYaw - player.rotationYaw) * smooth;
        player.rotationPitch += (targetPitch - player.rotationPitch) * smooth;
    }
}
```

---

## INTEGRATION HOOKS

### Hook Point 1: Add to GuiIngame.renderGameOverlay()

Find this method and add near the beginning:

```java
public void renderGameOverlay(float partialTicks) {
    ScaledResolution scaledresolution = this.mc.scaledResolution;
    int i = scaledresolution.getScaledWidth();
    int j = scaledresolution.getScaledHeight();
    
    // YOUR HOOK - Call module updates
    ModuleManager.getInstance().updateModules();
    
    // ... rest of the original method ...
    
    // Near the end, before drawScreen checks:
    
    // YOUR HOOK - Call module rendering and show ClickGUI
    ModuleManager.getInstance().renderModules(
        Mouse.getX() * i / this.mc.displayWidth,
        j - Mouse.getY() * j / this.mc.displayHeight - 1,
        partialTicks
    );
    if (ClickGUI.instance != null) {
        ClickGUI.instance.render(
            Mouse.getX() * i / this.mc.displayWidth,
            j - Mouse.getY() * j / this.mc.displayHeight - 1,
            partialTicks
        );
    }
}
```

### Hook Point 2: Add to Minecraft.processKeyBinds()

Find the key processing loop and add:

```java
private void processKeyBinds() {
    // ... existing code ...
    
    while (Keyboard.next()) {
        if (Keyboard.getEventKeyState()) {
            int key = Keyboard.getEventKey();
            
            // YOUR HOOK - GUI toggle and module keybinds
            if (key == Keyboard.KEY_Y) {  // Y key to toggle GUI
                if (ClickGUI.instance != null) {
                    ClickGUI.instance.toggle();
                }
                if (ClickGUI.instance != null && !ClickGUI.instance.isOpen()) {
                    ModuleManager.getInstance().checkKeyBinds(key);
                }
            } else if (ClickGUI.instance == null || !ClickGUI.instance.isOpen()) {
                // Only check module keybinds if GUI is closed
                ModuleManager.getInstance().checkKeyBinds(key);
            }
            
            // ... continue with other key handling ...
        }
    }
}
```

### Hook Point 3: Initialize on Startup

In Minecraft.java constructor or early startup, add:

```java
// Initialize custom client features
private void initializeCustomClient() {
    // Initialize ModuleManager and register all modules
    ModuleManager manager = ModuleManager.getInstance();
    
    manager.registerModule(new ModuleSpeed());
    manager.registerModule(new ModuleNoFall());
    manager.registerModule(new ModuleStep());
    manager.registerModule(new ModuleAimbot());
    // Add more modules as you create them
    
    // Initialize GUI
    ClickGUI.instance = new ClickGUI();
}
```

---

## RENDERING UTILITIES

### Draw 2D Rectangle

```java
public static void drawRect(int x1, int y1, int x2, int y2, int color) {
    if (x1 < x2) {
        int i = x1;
        x1 = x2;
        x2 = i;
    }
    if (y1 < y2) {
        int j = y1;
        y1 = y2;
        y2 = j;
    }
    
    float f = (float)(color >> 24 & 255) / 255.0F;
    float f1 = (float)(color >> 16 & 255) / 255.0F;
    float f2 = (float)(color >> 8 & 255) / 255.0F;
    float f3 = (float)(color & 255) / 255.0F;
    
    Tessellator tessellator = Tessellator.getInstance();
    GlStateManager.enableBlend();
    GlStateManager.disableTexture2D();
    GlStateManager.tryBlendFuncSeparate(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA, 1, 0);
    GlStateManager.color(f1, f2, f3, f);
    
    WorldRenderer worldrenderer = tessellator.getWorldRenderer();
    worldrenderer.begin(7, DefaultVertexFormats.POSITION);
    worldrenderer.pos((double)x1, (double)y2, 0.0D).endVertex();
    worldrenderer.pos((double)x2, (double)y2, 0.0D).endVertex();
    worldrenderer.pos((double)x2, (double)y1, 0.0D).endVertex();
    worldrenderer.pos((double)x1, (double)y1, 0.0D).endVertex();
    tessellator.draw();
    
    GlStateManager.enableTexture2D();
    GlStateManager.disableBlend();
}
```

### Draw 2D Text

```java
public static void drawString(String text, int x, int y, int color) {
    Minecraft.getMinecraft().fontRendererObj.drawStringWithShadow(text, x, y, color);
}
```

### Draw 3D Box (ESP)

```java
public static void drawBox(Entity entity, float partialTicks) {
    double x = entity.lastTickPosX + (entity.posX - entity.lastTickPosX) * partialTicks;
    double y = entity.lastTickPosY + (entity.posY - entity.lastTickPosY) * partialTicks;
    double z = entity.lastTickPosZ + (entity.posZ - entity.lastTickPosZ) * partialTicks;
    
    double width = entity.width / 2;
    double height = entity.height;
    
    GlStateManager.pushMatrix();
    GlStateManager.translate(x, y, z);
    
    // Draw wireframe box
    drawWireframeBox(-width, 0, -width, width, height, width);
    
    GlStateManager.popMatrix();
    GlStateManager.color(1.0F, 1.0F, 1.0F, 1.0F);
}

private static void drawWireframeBox(double minX, double minY, double minZ, 
                                     double maxX, double maxY, double maxZ) {
    Tessellator tessellator = Tessellator.getInstance();
    WorldRenderer wr = tessellator.getWorldRenderer();
    
    GlStateManager.disableTexture2D();
    GlStateManager.glLineWidth(2.0F);
    
    wr.begin(1, DefaultVertexFormats.POSITION_COLOR);
    // ... draw 12 edges of box ...
    tessellator.draw();
    
    GlStateManager.enableTexture2D();
    GlStateManager.glLineWidth(1.0F);
}
```

---

## COMMON PATTERNS

### Pattern 1: Get Player and Check Null

```java
@Override
public void onUpdate() {
    Minecraft mc = Minecraft.getMinecraft();
    EntityPlayerSP player = mc.thePlayer;
    WorldClient world = mc.theWorld;
    
    if (player == null || world == null) return;  // Safety check
    
    // Your code here
}
```

### Pattern 2: Find Nearby Entities

```java
List<EntityLivingBase> nearby = new ArrayList<>();
double range = 20.0;

for (Object o : Minecraft.getMinecraft().theWorld.loadedEntityList) {
    Entity entity = (Entity) o;
    if (!(entity instanceof EntityLivingBase)) continue;
    if (entity == Minecraft.getMinecraft().thePlayer) continue;
    
    double dist = Minecraft.getMinecraft().thePlayer.getDistanceToEntity(entity);
    if (dist < range) {
        nearby.add((EntityLivingBase) entity);
    }
}
```

### Pattern 3: Use Setting Value

```java
// In constructor:
Setting mySetting = new Setting("MyValue", this, 5.0, 0.0, 10.0);
addSetting(mySetting);

// In onUpdate:
double value = mySetting.getNumberValue();
```

### Pattern 4: Listen for Specific Key

```java
public void setKeyCode(int code) {
    this.keyCode = code;
}

// Module will auto-toggle when key is pressed (via ModuleManager)
```

### Pattern 5: Module State Check in Another Module

```java
@Override
public void onUpdate() {
    Module speedModule = ModuleManager.getInstance().getModule("Speed");
    
    if (speedModule != null && speedModule.isEnabled()) {
        // Do something special when Speed is enabled
    }
}
```

---

## IMPORTS REFERENCE

```java
// Module system
import net.lax1dude.eaglercraft.v1_8.client.module.Module;
import net.lax1dude.eaglercraft.v1_8.client.module.EnumModuleCategory;
import net.lax1dude.eaglercraft.v1_8.client.ModuleManager;
import net.lax1dude.eaglercraft.v1_8.client.Setting;

// Minecraft API
import net.minecraft.client.Minecraft;
import net.minecraft.entity.player.EntityPlayerSP;
import net.minecraft.entity.Entity;
import net.minecraft.world.WorldClient;
import net.minecraft.util.BlockPos;
import net.minecraft.client.gui.ScaledResolution;

// Rendering
import net.lax1dude.eaglercraft.v1_8.opengl.GlStateManager;
import net.minecraft.client.renderer.Tessellator;
import net.minecraft.client.renderer.vertex.DefaultVertexFormats;
import net.minecraft.client.renderer.WorldRenderer;

// Utilities
import java.util.ArrayList;
import java.util.List;
import java.util.HashMap;
import java.util.Map;
```

---

## DEBUG WINDOW MODULE (For Testing)

```java
package net.lax1dude.eaglercraft.v1_8.client.module.impl;

import net.minecraft.client.Minecraft;
import net.minecraft.entity.player.EntityPlayerSP;
import net.lax1dude.eaglercraft.v1_8.client.module.EnumModuleCategory;
import net.lax1dude.eaglercraft.v1_8.client.module.Module;

public class ModuleDebug extends Module {
    public ModuleDebug() {
        super("Debug Info", "Show debug information", EnumModuleCategory.MISC);
    }
    
    @Override
    public void onRender(int mouseX, int mouseY, float partialTicks) {
        EntityPlayerSP player = Minecraft.getMinecraft().thePlayer;
        if (player == null) return;
        
        int y = 10;
        int x = 10;
        int color = 0xFFFFFF00;  // Yellow
        
        Minecraft mc = Minecraft.getMinecraft();
        
        drawString(String.format("XYZ: %.1f %.1f %.1f", 
            player.posX, player.posY, player.posZ), x, y, color);
        
        drawString(String.format("Motion: %.3f %.3f %.3f",
            player.motionX, player.motionY, player.motionZ), x, y + 10, color);
        
        drawString(String.format("Yaw: %.1f Pitch: %.1f",
            player.rotationYaw, player.rotationPitch), x, y + 20, color);
        
        drawString(String.format("FPS: %d", Minecraft.getDebugFPS()),
            x, y + 30, color);
    }
    
    private void drawString(String text, int x, int y, int color) {
        Minecraft.getMinecraft().fontRendererObj.drawStringWithShadow(text, x, y, color);
    }
}
```

---

## FILE STRUCTURE

```
EaglercraftX-1.8-workspace-master/
└─ src/game/java/net/
    └─ lax1dude/eaglercraft/v1_8/client/
        ├─ Module.java                          (BASE CLASS)
        ├─ Setting.java                         (CONFIG VALUE)
        ├─ ModuleManager.java                   (REGISTRY & MANAGER)
        ├─ module/
        │   ├─ EnumModuleCategory.java          (CATEGORIES)
        │   └─ impl/
        │       ├─ ModuleSpeed.java
        │       ├─ ModuleNoFall.java
        │       ├─ ModuleStep.java
        │       ├─ ModuleAimbot.java
        │       ├─ ModuleDebug.java
        │       └─ [Your modules...]
        └─ gui/
            ├─ ClickGUI.java                    (MAIN GUI)
            ├─ ClickGUIPanel.java               (CATEGORY PANEL)
            ├─ ClickGUIButton.java              (MODULE BUTTON)
            └─ ClickGUISlider.java              (SETTING SLIDER)
```

---

**That's your complete reference guide! Copy and paste as needed, and customize for your needs.**
