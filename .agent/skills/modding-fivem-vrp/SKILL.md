---
name: modding-fivem-vrp
description: Comprehensive guide to modding FiveM servers using the VRP (vRP 1/2), VRPex, and Creative frameworks. Includes architecture, API reference, common patterns, and migration guides.
---

# VRP / VRPex / Creative Framework Modding

This skill provides deep technical knowledge for developing resources on strings of the **VRP (Virtual RolePlay)** framework ecosystem, including **vRP 1**, **vRP 2**, **VRPex**, and **Creative** (a popular Brazilian fork).

## 1. Architecture Overview

### The Proxy & Tunnel System

VRP uses a unique `Proxy` and `Tunnel` system to bridge Server and Client execution context.

- **Proxy**: Allows calling functions from one resource to another *on the same side* (Server-to-Server or Client-to-Client).
- **Tunnel**: Allows calling functions across the network (Server-to-Client or Client-to-Server).

**Standard Boilerplate:**

```lua
-- server.lua
local Tunnel = module("vrp", "lib/Tunnel")
local Proxy = module("vrp", "lib/Proxy")

vRP = Proxy.getInterface("vRP")
vRPclient = Tunnel.getInterface("vRP")

-- Create a Tunnel for Client to access Server functions
src = {}
Tunnel.bindInterface("my_resource", src)
Proxy.addInterface("my_resource", src)

-- client.lua
local Tunnel = module("vrp", "lib/Tunnel")
local Proxy = module("vrp", "lib/Proxy")

vRP = Proxy.getInterface("vRP")
vRPserver = Tunnel.getInterface("vRP", "my_resource") -- Access Server Tunnel

src = {}
Tunnel.bindInterface("my_resource", src)
Proxy.addInterface("my_resource", src)
```

### The ID System (User_ID)

Unlike ESX/QBCore which often use CitizenIDs strings, VRP uses an auto-incrementing **Integer** `user_id`.

- `user_id` 1 is the first player.
- `user_id` is tied to a specific license/steam identified in the database.
- **Source vs UserID**: You typically convert `source` (session id) to `user_id` (database id) for almost all logic.

## 2. Server-Side API Reference (Common vRP/VRPex)

Most functions are accessed via `vRP` interface.

### Player Management

```lua
local user_id = vRP.getUserId(source) -- Get DB ID from Source
local source = vRP.getUserSource(user_id) -- Get Source from DB ID (returns nil if offline)
local ply = vRP.getPlayerEndpoint(source) -- Get IP
vRP.isBanned(user_id) -- Check ban status
vRP.setBanned(user_id, true) -- Ban user
vRP.kick(source, "Reason") -- Kick player
```

### Economy (Money & Bank)

```lua
vRP.getMoney(user_id)
vRP.tryPayment(user_id, amount) -- Returns true/false
vRP.giveMoney(user_id, amount)
vRP.getBankMoney(user_id)
vRP.setBankMoney(user_id, amount)
vRP.tryWithdraw(user_id, amount)
vRP.tryDeposit(user_id, amount)
```

### Inventory

```lua
vRP.getInventory(user_id) -- Returns table: { ["item_id"] = { amount = 10, ... } }
vRP.getInventoryWeight(user_id)
vRP.getInventoryMaxWeight(user_id)
vRP.giveInventoryItem(user_id, "id_name", amount, notify_bool)
vRP.tryGetInventoryItem(user_id, "id_name", amount, notify_bool) -- Removes item, returns true if successful
vRP.clearInventory(user_id)
```

### Groups & Permissions

```lua
vRP.hasPermission(user_id, "police.service") -- Check permission node
vRP.hasGroup(user_id, "Police") -- Check if in group
vRP.addUserGroup(user_id, "Police")
vRP.removeUserGroup(user_id, "Police")
```

### Saving Data (UData, SData, VData)

VRP uses a Key-Value store in the database (usually `vrp_user_data`, `vrp_srv_data`).

```lua
-- Set Data
vRP.setUData(user_id, "key", json.encode(data_table))

-- Get Data (Async usually)
local data_str = vRP.getUData(user_id, "key")
local data = json.decode(data_str) or {}
```

## 3. Client-Side API Reference

Accessed via `vRP` (Proxy from client) or specific handlers.

```lua
vRP.notify({"Message"}) -- Standard notification
vRP.playAnim(bool_upper, { {"dict", "anim", loops} }, bool_looping)
vRP.teleport(x, y, z)
vRP.isInComa() -- Check if dead/coma state
```

## 4. Creative Framework Differences (Brazilian VRP Fork)

"Creative" (often `Summerz`, `Network`, `VRPex`) makes significant changes to standard VRP 1/2.

- **Global `vRP`**: Often defines `vRP` globally in `lib/utils` so you don't always need `Proxy.getInterface`.
- **Passport System**: Renames `user_id` to `Passport` in UI/NUI contexts.
- **Inventory Rewrite**: Creative inventories are often robust, NUI-based, with metadata.
- **Event-Driven**: Heavily relies on `RegisterServerEvent` wrapper functions.
- **PolyZone Integration**: Creative usually has built-in zone management.

**Creative Specific Functions:**

```lua
vRP.passport(source) -- Equivalent to getUserId
vRP.identities(source) -- Returns identity table
vRP.request(source, "Question?", time) -- Yes/No prompt
vRP.prompt(source, "Title", "Default") -- Text Input
```

## 5. Directory Structure & Resources

A typical VRP legacy server structure:

```
resources/
  [vrp]/
    vrp/            -- Core Framework
    vrp_mysql/      -- Database Driver (ghmattimysql or oxmysql often used now)
    vrp_admin/      -- Admin Tools
    vrp_barbershop/ -- Character Customization
    vrp_bank/       -- Banking UI
    vrp_garages/    -- Garage System
    vrp_inventory/  -- Inventory HUD
```

## 6. Creating a Module (VRP Class System)

VRP 2 uses Luaoop. Modules are classes.

```lua
local MyModule = class("MyModule", vRP.Extension)

function MyModule:__construct()
  vRP.Extension.__construct(self)
end

vRP:registerExtension(MyModule)
```

## 7. Common Pitfalls & Debugging

- **Proxy/Tunnel Mismatch**: If you change a function signature on Server but not Client (or vice versa) in a Tunnel call, it can fail silently or desync arguments.
- **User Source Loss**: `source` can be lost in async calls (Wait). Always grab `user_id` early or re-verify source if using extensive delays.
- **Database Locks**: Old `vrp_mysql` can deadlock. Migrate to `oxmysql` standard in modern VRPex.
- **Item Weight Limits**: `vRP.tryGetInventoryItem` respects weight. Always check return value. Never assume it worked.
- **Config Files**: `cfg/` folders rule everything. `groups.lua`, `items.lua`, `inventory.lua` are the heart of balancing.

## 8. Migration Guide (ESX/QBCore to VRP)

- **ESX `xPlayer`** -> **vRP `user_id`**: You don't pass an object object around. You pass the integer ID.
- **Events**: VRP uses Tunnel instead of `TriggerServerEvent` for many internal calls, but `TriggerServerEvent` works fine for custom logic.
- **Database**: VRP tables are strictly relational (`vrp_users`, `vrp_user_ids`, `vrp_user_data`). ESX/QBCore are often JSON blobs in `users` table.

## 9. Code Snippets

**Custom Command:**

```lua
RegisterCommand("checkbal", function(source, args, raw)
    local user_id = vRP.getUserId(source)
    if user_id then
        local bank = vRP.getBankMoney(user_id)
        vRPclient.notify(source, {"Bank Balance: $"..bank})
    end
end)
```

**Thread Example:**

```lua
Citizen.CreateThread(function()
    while true do
        Citizen.Wait(1000)
        for user_id, source in pairs(vRP.getUsers()) do
            -- Periodic task for all online players
        end
    end
end)
```
