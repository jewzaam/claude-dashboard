# Title Bar Click Flow Analysis

How left-click (shade) and middle-click (ghost toggle) produce geometry
changes through a shared `_apply_geometry` path.

## Left-Click: Shade Toggle

No controller involvement. Entirely within `MainWindow`.

```mermaid
flowchart TD
    E(("ButtonRelease-1")) --> A

    subgraph _on_shade_toggle
        A["_shaded = !_shaded"]
    end

    A --> B

    subgraph _apply_shade_state
        B["_apply_title_bar_style()"]
        B --> C{_shaded?}
        C -->|yes| D["pack_forget() all rows"]
        C -->|no| F["re-pack rows in _row_order"]
    end

    D --> G
    F --> G

    subgraph _apply_geometry
        G["compute new_height\nfrom row_count + _shaded"]
        G --> H["x = _tracked_x\ny = _tracked_y"]
        H --> I["_set_geometry(WxH+x+y)"]
        I --> J["_tracked_x/y updated"]
    end
```

## Middle-Click: Ghost Toggle

Crosses into `AppController`, returns to `MainWindow` via
`update_sessions`, which calls the same `_apply_geometry`.

```mermaid
flowchart TD
    E(("Button-2")) --> A

    subgraph MainWindow
        A["_on_ghost_toggle_click()"]
    end

    A --> B

    subgraph AppController
        B["_on_ghost_toggle()"]
        B --> C["toggle hidden flags\n_save_session_state()"]
        C --> D["_refresh_ui()"]
        D --> R{high-priority\nstate visible?}
        R -->|yes| IMM["_do_refresh_ui() immediate"]
        R -->|no| DEB["after(300ms)\n_do_refresh_ui_deferred()"]
        IMM --> UPD
        DEB --> UPD
        UPD["_do_refresh_ui()"]
    end

    UPD --> US

    subgraph update_sessions
        US["add/remove/reorder row frames"]
        US --> CH{rows changed OR\n_force_resize?}
        CH -->|no| SKIP["return"]
        CH -->|yes| G
    end

    subgraph _apply_geometry
        G["compute new_height\nfrom row_count + _shaded"]
        G --> H["x = _tracked_x\ny = _tracked_y"]
        H --> I["_set_geometry(WxH+x+y)"]
        I --> J["_tracked_x/y updated"]
    end
```

## Comparison

| Aspect | Left-click (shade) | Middle-click (ghost toggle) |
|--------|--------------------|-----------------------------|
| Entry point | `_on_shade_toggle` | `_on_ghost_toggle_click` |
| State change | `_shaded` flag (MainWindow) | ghost `hidden` flags (Controller) |
| Row packing | `_apply_shade_state` | `update_sessions` |
| Geometry | `_apply_geometry` | `_apply_geometry` |
| Position source | `_tracked_x/y` | `_tracked_x/y` |
| Synchronous? | Yes | No (300ms debounce) |
