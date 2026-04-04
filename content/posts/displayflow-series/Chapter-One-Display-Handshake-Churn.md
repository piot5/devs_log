---
title: "Chapter One: Display Handshake Churn"
date: 2026-04-03
draft: false
description: "Neutralizing monitor setup pain points via Win32 API staging."
tags: ["Rust", "Windows", "GDI"]
---

Managing monitor setups, resolutions, positions, and orientations is a common pain point for most multi display setup users. While Windows provides a UI for this, automating it via code involves navigating the deep waters of the **GDI**.

I’ll dissect a core component of my project, **displayflow**, specifically focusing on how to "stage" display configurations to ensure a seamless transition between monitor profiles.

Automating monitor setups on Windows requires precise interaction with the Win32 API. In this post, we analyze the "stage\_config" function, which is responsible for preparing display changes in the registry.

## Detailed Functionality

### 1\. Neutralizing "Display Handshake Churn"

If you've ever changed your resolution, you know the screen goes black for a second. Now imagine doing that for four monitors sequentially. The system would hang, windows would jump around, and it would take ages.

The solution is a **two-phase commit** approach:

1. **Staging**: Write all desired changes to the registry without triggering a hardware reset.
2. **Applying**: Trigger one single global reset to apply all changes at once.

### 2\. Multi-Monitor Synchronization

A core aspect of this function is the use of "CDS\_NORESET". If we omitted this flag, Windows would force a screen flicker and a desktop rearrangement after every single monitor update. By "staging" the changes, all new values are written to the registry first.

### 3\. The Final Step

After calling stage\_config for all monitors, a final call to ChangeDisplaySettingsExW(None, None, None, CDS\_UPDATEREGISTRY, None) is executed. This tells Windows to take all the staged registry entries and apply them in a single hardware handshake.

### 4\. Just a sidequest on the way: "Portrait" Swap

For now, I am staying within the GDI stack and not touching CCD API; a full implementation is on my roadmap.
Win32 Display API handles rotation based on resolution. When a monitor is rotated 90 or 270 degrees, the logical width and height must be swapped in the "DEVMODEW" structure.

```rust
let (w, h) = if rotation == DMDO\\\\\\\_90 || rotation == DMDO\\\\\\\_270 { 
    (task.height, task.width) // Swap for vertical orientation
} else { 
    (task.width, task.height) 
};
```

## The Code

The following code illustrates how the DFEngine handles the configuration staging phase:

```rust
   fn stage_config(&self, name: &str, task: &DisplayTask) -> bool {
    let name_u16: Vec<u16> = name.encode_utf16().chain(iter::once(0)).collect();
    unsafe {
        let mut dm = GdiDevMode::new();
        if EnumDisplaySettingsW(PCWSTR(name_u16.as_ptr()), ENUM_CURRENT_SETTINGS, &mut dm.0).as_bool() {
            // 1. Determine rotation
            let rotation = match task.direction.as_deref() {
                Some("90") | Some("right") => DMDO_90, 
                Some("180") | Some("inverted") => DMDO_180, 
                Some("270") | Some("left") => DMDO_270, 
                _ => DMDO_DEFAULT
            };

            // 2. Adjust dimensions based on rotation
            let (w, h) = if rotation == DMDO_90 || rotation == DMDO_270 { 
                (task.height, task.width) 
            } else { 
                (task.width, task.height) 
            };

            dm.0.dmPelsWidth = w as u32;
            dm.0.dmPelsHeight = h as u32;
            
            // 3. Mark relevant fields
            dm.0.dmFields |= DM_PELSWIDTH | DM_PELSHEIGHT | DM_POSITION | DM_DISPLAYORIENTATION;
            dm.0.Anonymous1.Anonymous2.dmPosition.x = task.x;
            dm.0.Anonymous1.Anonymous2.dmPosition.y = task.y;
            dm.0.Anonymous1.Anonymous2.dmDisplayOrientation = rotation;

            // 4. Optional frequency setting
            if task.freq > 0 {
                dm.0.dmDisplayFrequency = task.freq;
                dm.0.dmFields |= DM_DISPLAYFREQUENCY;
            }

            // 5. Flags for staging
            let mut flags = CDS_UPDATEREGISTRY | CDS_NORESET;
            if task.is_primary { flags |= CDS_SET_PRIMARY; }

            // 6. Disable monitor if resolution is 0/0
            // Note: We must mark dmFields to ensure Windows doesn't ignore the 0 values.
            if task.width == 0 && task.height == 0 {
                dm.0.dmPelsWidth = 0;
                dm.0.dmPelsHeight = 0;
                dm.0.dmFields = DM_PELSWIDTH | DM_PELSHEIGHT | DM_POSITION;
            }

            // 7. Write to registry without reset
            ChangeDisplaySettingsExW(
                PCWSTR(name_u16.as_ptr()), 
                Some(&dm.0), 
                None, 
                flags, 
                None
            ) == DISP_CHANGE_SUCCESSFUL
        } else { 
            false 
        }
```

What features are coming next:

DDC input and powerstate (Brightness and Contrast are working)



Full CCD implementation



As you might have noticed if you checked in my GitHub files, i prepared a pipe to communicate to extern binary frontend.


