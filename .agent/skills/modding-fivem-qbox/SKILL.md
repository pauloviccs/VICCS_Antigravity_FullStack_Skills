---
name: modding-fivem-qbox
description: Guides the creation of scripts, resources, and server management for FiveM using the Qbox framework. Use when the user asks about Qbox, qbx_core, migrating from QBCore, or creating modern FiveM RP scripts with the Ox ecosystem.
---

# Modding FiveM (Qbox Framework)

## What is Qbox?

Qbox is a **modern FiveM roleplay framework** created on September 27, 2022, as a fork of QBCore. It improves upon QBCore with:

- **~99% backwards compatibility** with QBCore scripts (bridge layer)
- **Better performance**: ~0.42ms CPU, ~30 MiB memory
- **Enhanced security**: Written with anti-cheat in mind
- **Higher code quality**: Refactored to modern standards
- **Ox ecosystem integration**: Uses `ox_lib`, `oxmysql`, `ox_inventory`

## When to Use This Skill

- Creating or modifying **Qbox/qbx_core resources**
- Migrating scripts from **QBCore to Qbox**
- Working with **Ox ecosystem** (`ox_lib`, `ox_inventory`, `oxmysql`)
- Building **modern FiveM RP servers**
- Using **exports and modules** instead of the legacy core object

---

## Core Differences: Qbox vs QBCore

| Aspect | QBCore | Qbox |
|--------|--------|------|
| Core Object | `exports['qb-core']:GetCoreObject()` | Uses exports + imported modules |
| Inventory | `qb-inventory` | `ox_inventory` |
| UI Library | Custom | `ox_lib` |
| Database | `oxmysql` | `oxmysql` |
| Player Data | `QBCore.Functions.GetPlayerData()` | `QBX.PlayerData` (via module import) |
| Notifications | `QBCore.Functions.Notify()` | `lib.notify()` from ox_lib |
| Text UI | `exports['qb-core']:DrawText()` | `lib.showTextUI()` |

---

## Developer's Guide: Best Practices

### ❌ DO NOT

1. **Access database tables owned by qbx_core directly**
   - If the schema changes, your script breaks
   - Request new exports via GitHub issues if data isn't accessible

2. **Modify qbx_core internal code**
   - Makes updates difficult and confuses debugging
   - File a GitHub issue if you need flexibility

3. **Use deprecated functions/events**
   - These will be removed in future updates

4. **Rely on unversioned/unreleased resources**
   - No guarantees of stability; breaking changes likely

### ✅ DO

1. **Use exports and imported modules**
2. **Set statebag when spawning owned vehicles**
   - Set `vehicleid` statebag to the ID from `player_vehicles` table
3. **Pass vehicle properties into spawnVehicle function**
   - Don't set properties after the vehicle exists (anti-pattern)

---

## Resource Structure (Qbox)

```
[my-resource]/
├── fxmanifest.lua
├── config.lua
├── client/
│   └── main.lua
├── server/
│   └── main.lua
└── shared/
    └── locale.lua
```

### fxmanifest.lua

```lua
fx_version 'cerulean'
game 'gta5'

description 'My Qbox Resource'
version '1.0.0'

-- Import ox_lib modules
shared_scripts {
    '@ox_lib/init.lua',  -- Required for ox_lib
    'config.lua'
}

client_scripts { 'client/main.lua' }
server_scripts { 'server/main.lua' }
```

---

## API: Common Exports & Modules

### Importing Modules (Client)

```lua
-- Import playerdata module for QBX.PlayerData
local QBX = exports.qbx_core:GetPlayerData and {} or {}
if GetResourceState('qbx_core') == 'started' then
    QBX.PlayerData = exports.qbx_core:GetPlayerData()
end

-- Or use the module system
local playerData = require '@qbx_core.modules.playerdata'
local qbx = require '@qbx_core.modules.lib'
```

### Key Exports

```lua
-- Jobs & Gangs
local jobs = exports.qbx_core:GetJobs()
local gangs = exports.qbx_core:GetGangs()

-- Vehicles & Weapons
local vehicles = exports.qbx_core:GetVehiclesByName()
local weapons = exports.qbx_core:GetWeapons()

-- Locations
local locations = exports.qbx_core:GetLocations()

-- Items (via ox_inventory)
local items = exports.ox_inventory:Items()

-- Player Search (Server)
local players = exports.qbx_core:SearchPlayers({
    metadata = { ['licences.driver'] = true }
})

-- Delete Vehicle (preferred method, handles persistence)
exports.qbx_core:DeleteVehicle(entity)

-- Get Vehicle Plate
local plate = qbx.getVehiclePlate(vehicle) -- requires lib module
```

---

## Converting QBCore to Qbox

### Common Replacements Table

| QBCore | Qbox |
|--------|------|
| `QBCore.Functions.GetPlayerData()` | `QBX.PlayerData` (import playerdata module) |
| `QBCore.Functions.GetPlate(vehicle)` | `qbx.getVehiclePlate(vehicle)` (import lib module) |
| `QBCore.Shared.Jobs` | `exports.qbx_core:GetJobs()` |
| `QBCore.Shared.Gangs` | `exports.qbx_core:GetGangs()` |
| `QBCore.Shared.Vehicles` | `exports.qbx_core:GetVehiclesByName()` |
| `QBCore.Shared.Weapons` | `exports.qbx_core:GetWeapons()` |
| `QBCore.Shared.Locations` | `exports.qbx_core:GetLocations()` |
| `QBCore.Shared.Items` | `exports.ox_inventory:Items()` |
| `exports['qb-core']:DrawText(text, pos)` | `lib.showTextUI(text, { position = pos })` |
| `exports['qb-core']:HideText()` | `lib.hideTextUI()` |
| `exports['qb-core']:KeyPressed()` | `lib.hideTextUI()` |

### Conversion Example

**Before (QBCore):**

```lua
local QBCore = exports['qb-core']:GetCoreObject()

RegisterCommand('getjob', function()
    local PlayerData = QBCore.Functions.GetPlayerData()
    local job = PlayerData.job.name
    QBCore.Functions.Notify('Your job: ' .. job, 'success')
end)
```

**After (Qbox):**

```lua
-- In fxmanifest.lua, ensure '@ox_lib/init.lua' is in shared_scripts

RegisterCommand('getjob', function()
    local PlayerData = exports.qbx_core:GetPlayerData()
    local job = PlayerData.job.name
    lib.notify({
        title = 'Job Info',
        description = 'Your job: ' .. job,
        type = 'success'
    })
end)
```

---

## Server Migration Steps

### 1. Update Configuration Files

Update `server.cfg`, `ox.cfg`, `permissions.cfg`, and `voice.cfg`.

### 2. Update Job Grades

In Qbox, grades are **numbers without quotations** (not strings like QBCore).

**qbx_core/shared/jobs.lua:**

```lua
['police'] = {
    label = 'Police',
    grades = {
        [0] = { name = 'Recruit', payment = 500 },  -- Note: numeric keys
        [1] = { name = 'Officer', payment = 750 },
        [2] = { name = 'Sergeant', payment = 1000 },
    }
}
```

### 3. Configure ox_inventory

Run the database conversion tool provided by ox_inventory.

### 4. Run SQL Migrations

Execute `qbx_core.sql` to create necessary tables.

Ensure `player.citizenid` column has correct collation:

```sql
ALTER TABLE players MODIFY citizenid VARCHAR(11) 
CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### 5. Built-in Features

Qbox includes built-in:

- **Multicharacter** system
- **Queue** system  
- **Multi-job/gang** support

Configure these in `qbx_core/config.lua`.

---

## Using ox_lib (Essential)

Qbox heavily relies on `ox_lib`. Common patterns:

### Notifications

```lua
lib.notify({
    title = 'Title',
    description = 'Message here',
    type = 'success', -- 'success', 'error', 'info', 'warning'
    duration = 5000
})
```

### Text UI

```lua
-- Show
lib.showTextUI('[E] Interact', {
    position = 'right-center',
    icon = 'hand'
})

-- Hide
lib.hideTextUI()
```

### Context Menus

```lua
lib.registerContext({
    id = 'my_menu',
    title = 'My Menu',
    options = {
        {
            title = 'Option 1',
            description = 'Does something',
            icon = 'gear',
            onSelect = function()
                print('Selected!')
            end
        }
    }
})

lib.showContext('my_menu')
```

### Input Dialogs

```lua
local input = lib.inputDialog('Enter Details', {
    { type = 'input', label = 'Name', required = true },
    { type = 'number', label = 'Amount', min = 1, max = 100 },
    { type = 'select', label = 'Type', options = {
        { value = 'a', label = 'Option A' },
        { value = 'b', label = 'Option B' }
    }}
})

if input then
    print(input[1], input[2], input[3]) -- Name, Amount, Type
end
```

### Callbacks (Replacing QBCore Callbacks)

```lua
-- Server
lib.callback.register('myResource:getServerData', function(source, arg1)
    -- Returns data to client
    return { success = true, data = arg1 }
end)

-- Client
local result = lib.callback.await('myResource:getServerData', false, 'test')
print(result.success, result.data)
```

---

## Creating a Complete Qbox Resource

### Example: Simple Job Check Script

**fxmanifest.lua:**

```lua
fx_version 'cerulean'
game 'gta5'

description 'Job Info Script'
version '1.0.0'

shared_scripts { '@ox_lib/init.lua' }
client_scripts { 'client/main.lua' }
```

**client/main.lua:**

```lua
local isOnDuty = false

RegisterCommand('duty', function()
    local PlayerData = exports.qbx_core:GetPlayerData()
    
    if not PlayerData.job then
        return lib.notify({
            title = 'Error',
            description = 'No job data found',
            type = 'error'
        })
    end
    
    isOnDuty = not isOnDuty
    
    -- Trigger server event to update duty status
    TriggerServerEvent('qbx_core:server:SetDuty', isOnDuty)
    
    lib.notify({
        title = 'Duty Status',
        description = isOnDuty and 'You are now on duty' or 'You are now off duty',
        type = 'success'
    })
end, false)

-- Register keybind with ox_lib
lib.addKeybind({
    name = 'toggle_duty',
    description = 'Toggle Duty Status',
    defaultKey = 'F6',
    onPressed = function()
        ExecuteCommand('duty')
    end
})
```

---

## Key Qbox Resources

| Resource | Purpose |
|----------|---------|
| `qbx_core` | Core framework |
| `ox_lib` | UI library, callbacks, utilities |
| `ox_inventory` | Inventory system |
| `oxmysql` | Database driver |
| `qbx_vehicles` | Vehicle management |
| `qbx_garages` | Garage system |
| `qbx_banking` | Banking system |
| `qbx_policejob` | Police job |
| `qbx_ambulancejob` | EMS job |

---

## Resources

- [Qbox Documentation](https://docs.qbox.re/)
- [Qbox GitHub](https://github.com/Qbox-project)
- [Qbox Discord](https://discord.gg/qbox)
- [ox_lib Documentation](https://overextended.dev/ox_lib)
- [ox_inventory Documentation](https://overextended.dev/ox_inventory)
- [FiveM Native Reference](https://docs.fivem.net/natives/)
- [Qbox Installation Guide](https://docs.qbox.re/installation)
- [Converting from QBCore](https://docs.qbox.re/converting)
- [Developer's Guide](https://docs.qbox.re/developers)
