---
title: "Chapter One: Display Handshake Churn"
date: 2026-01-03
tags: ["Rust", "Windows", "GDI"]
description: "Neutralizing monitor setup pain points via Win32 API staging."
project_url: "https://github.com/piot5/displayflow"
landing_page: "https://piot5.github.io/displayflow_cli/"
youtube_demo: "https://www.youtube.com/watch?v=_ch6X-_A2Ms"
---

Managing monitor setups—resolutions, positions, and orientations—is a notorious pain point. While Windows provides a UI, automating it involves navigating the deep waters of the **GDI (Graphics Device Interface)**.

## 1. Neutralizing "Display Handshake Churn"

If you change your resolution manually, the screen flickers or goes black. Doing this for four monitors sequentially would cause the system to hang and windows to jump randomly across the desktop.

The solution is a **two-phase commit** approach:
1. **Staging**: Write desired changes to the registry using the `CDS_NORESET` flag.
2. **Applying**: Trigger one single global reset to apply all changes at once.

## 2. The Staging Logic

In `display_controller.rs`, the `stage_config` function prepares the `DEVMODEW` structure. Note the bitwise OR operator (`|=`) to preserve existing flags:

```rust
// Snippet from display_controller.rs
pub fn stage_config(&self, device_name: &str, task: &DisplayTask) -> bool {
    unsafe {
        let mut dm = GdiDevMode::new();
        let name_u16: Vec<u16> = device_name.encode_utf16().chain(iter::once(0)).collect();

        // 1. Load current settings to preserve unrelated flags
        if EnumDisplaySettingsExW(PCWSTR(name_u16.as_ptr()), ENUM_CURRENT_SETTINGS, &mut dm.0, 0).as_bool() {
            
            // 2. Set new dimensions and position
            dm.0.dmPelsWidth = task.width as u32;
            dm.0.dmPelsHeight = task.height as u32;
            dm.0.dmFields |= DM_PELSWIDTH | DM_PELSHEIGHT | DM_POSITION | DM_DISPLAYORIENTATION;
            
            dm.0.Anonymous1.Anonymous2.dmPosition.x = task.x;
            dm.0.Anonymous1.Anonymous2.dmPosition.y = task.y;
            dm.0.Anonymous1.Anonymous2.dmDisplayOrientation = task.get_rotation_constant();

            // 3. Apply via Registry Staging
            let mut flags = CDS_UPDATEREGISTRY | CDS_NORESET;
            if task.is_primary { flags |= CDS_SET_PRIMARY; }

            ChangeDisplaySettingsExW(
                PCWSTR(name_u16.as_ptr()), 
                Some(&dm.0), 
                None, 
                flags, 
                None
            ) == DISP_CHANGE_SUCCESSFUL
        } else { false }
    }
}
```



Back to Project: [GitHub]({{< param project_url >}}) | [Official Website]({{< param landing_page >}}) | [Demo Video]({{< param youtube_demo >}})