---
name: modding-fivem-qbcore
description: Guides the creation of scripts, resources, and server management for FiveM using the QBCore framework. Use when the user asks about GTA V RP servers, FiveM scripts (Lua), QBCore exports, or creating UI (NUI) for FiveM.
---

# Modding FiveM (QBCore Framework)

## When to use this skill

- When the user asks to create or modify a **FiveM resource** (`fxmanifest.lua`).
- When the user mentions **QBCore**, "city scripts", or RP framework logic.
- When working with **Lua** scripts for GTA V.
- When creating NUI (HTML/CSS/JS interfaces) for FiveM.

## Workflow

1. **Resource Creation**:
    - Create folder `[script-name]`.
    - Create `fxmanifest.lua` (The manifest file).
    - Create `client/main.lua` and `server/main.lua`.
2. **QBCore Initialization**:
    - In `client/main.lua` & `server/main.lua`, retrieve the core object:

        ```lua
        local QBCore = exports['qb-core']:GetCoreObject()
        ```

3. **Event Handling**:
    - Listen for QBCore events (e.g., `QBCore:Client:OnPlayerLoaded`).
    - Register network events (`RegisterNetEvent`, `TriggerServerEvent`).
4. **UI Development (NUI)**:
    - Use `SendNUIMessage` (Lua) -> `window.addEventListener('message')` (JS).
    - Use `fetch` (`https://resource-name/callback`) to send data back to Lua.

## Instructions

### 1. `fxmanifest.lua` Standard

```lua
fx_version 'cerulean'
game 'gta5'

description 'My QBCore Script'
version '1.0.0'

shared_scripts { 'config.lua' }
client_scripts { 'client/main.lua' }
server_scripts { 'server/main.lua' }
```

### 2. Client-Side Pattern (Lua)

- **Natives**: Use [FiveM Natives](https://docs.fivem.net/natives/) for game logic (spawning cars, drawing markers).
- **QBCore**: Use `QBCore.Functions.Notify('Message', 'success')` for UI feedback.
- **Threads**: Use `CreateThread(function() ... end)` for loops (like drawing markers every frame).

### 3. Server-Side Pattern (Lua)

- **Database**: Use `oxmysql` (standard for QBCore) for SQL queries.

    ```lua
    MySQL.query('SELECT * FROM players WHERE citizenid = ?', {cid}, function(result) ... end)
    ```

- **Items**: `xPlayer.Functions.AddItem('item_name', 1)`

### 4. NUI (HTML/CSS/React)

- Scripts can open a web browser overlay.
- **Show**: `SetNuiFocus(true, true)`
- **Hide**: `SetNuiFocus(false, false)`

### 5. Modern NUI (React/Vue)

Modern NUI development uses web frameworks like React or Vue 3 for complex, reactive interfaces.

#### Architecture

- **Structure**: `[resource-name]/web/` (contains package.json, source code).
- **Build Process**: Use Vite or Webpack to bundle the app into a single `index.html` + `assets/` folder.
- **fxmanifest**: Point `ui_page` to the *built* index file (e.g., `web/dist/index.html`).

#### Communication Patterns (Two-Way)

**1. Lua to JS (Receiving Data):**
In React/Vue, listen for the `message` event.

*Lua (Sending):*

```lua
SendNUIMessage({
    action = 'OPEN_INVENTORY',
    data = { items = {} }
})
```

*JS/React (Receiving - Hook Example):*

```javascript
import { useEffect, useState } from 'react';

export const useNuiEvent = (action, handler) => {
  useEffect(() => {
    const eventListener = (event) => {
      const { action: eventAction, data } = event.data;
      if (eventAction === action) handler(data);
    };
    window.addEventListener('message', eventListener);
    return () => window.removeEventListener('message', eventListener);
  }, [action, handler]);
};

// Usage
useNuiEvent('OPEN_INVENTORY', (data) => console.log(data.items));
```

**2. JS to Lua (Sending Data / Callbacks):**
Use `fetch()` to post data to the resource URL.

*JS/React (Sending):*

```javascript
// Function wrapper for NUI fetch
const fetchNui = async (eventName, data = {}) => {
  const resourceName = window.GetParentResourceName ? window.GetParentResourceName() : 'nui-frame-app';
  const resp = await fetch(`https://${resourceName}/${eventName}`, {
    method: 'post',
    headers: { 'Content-Type': 'application/json; charset=UTF-8' },
    body: JSON.stringify(data),
  });
  return await resp.json();
};

// Usage
fetchNui('closeUI', { reason: 'user_closed' }).then(result => console.log(result));
```

*Lua (Receiving Callback):*

```lua
RegisterNUICallback('closeUI', function(data, cb)
    SetNuiFocus(false, false)
    print("UI Closed: " .. data.reason)
    cb({ status = 'ok' }) -- Respond to the fetch promise
end)
```

#### Best Practices

- **Development Mode**: Run your React/Vue dev server `npm run dev` in browser for UI testing. Mock the NUI events.
- **Production Build**: Always run `npm run build` before restarting the script in FiveM.
- **Focus Handling**: Don't leave `SetNuiFocus(true, true)` without a way to exit (ESC key handler in JS).
- **State Management**: Use Zustand (React) or Pinia (Vue) for "global" data like player inv, distinct from local UI state.

## Resources

- [FiveM Native Reference](https://docs.fivem.net/natives/)
- [QBCore Documentation](https://docs.qbcore.org/qbcore-documentation)
- [Cfx.re Cookbook](https://docs.fivem.net/docs/cookbook/)
