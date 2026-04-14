---
title: "Chapter Three: Ghost Monitors & The EDID Void"
date: 2026-05-10
tags: ["Windows API", "EDID", "Ghosting", "Win32"]
description: "Why inactive monitors are invisible to hardware calls and how to bridge the gap."
project_url: "https://github.com/piot5/displayflow"
landing_page: "https://piot5.github.io/displayflow_cli/"
youtube_demo: "https://www.youtube.com/watch?v=_ch6X-_A2Ms"
---

In a perfect world, every monitor would always be reachable. In reality, Windows often keeps "Ghost Monitors" in its registry—displays that were once connected but are now powered off or disconnected. These ghosts create a massive problem for automation: they exist in the GDI, but they have no hardware pulse.
1. The EDID Dead End

When a monitor is inactive or "detached" in Windows, the operating system stops polling its hardware bus. If you try to call GetVCPFeatureAndVCPFeatureReply or read the EDID via SetupDi functions on an inactive display, the hardware returns... nothing.

    The Problem: You cannot identify a monitor's serial number if the monitor is currently disabled in the display settings.

This creates a "Chicken and Egg" problem: You need the serial number to know which monitor it is to enable it, but you have to enable it first to see the serial number.
2. Bridging the Gap in scan.rs

DisplayFlow solves this by splitting the scan into two distinct layers:

    Live GDI Layer: Only sees what is currently "active" and has an HMONITOR handle.

    Registry Layer: Scans SYSTEM\CurrentControlSet\Enum\DISPLAY to find every device the PC has ever seen.

In scan.rs, we iterate through the registry even for monitors that don't have a live handle:
Rust

// Snippet from scan.rs: Hunting for ghosts
if let Ok(params) = hw_key.open_subkey(format!(r"{}\Device Parameters", inst_id)) {
    if let Ok(edid) = params.get_raw_value("EDID") {
        let (sn, size, res) = parse_edid(&edid.bytes);
        // We found it! Even if the monitor is 'Off', 
        // Windows stores the last known EDID here.
    }
}

3. The "Synthetic ID" Solution

Since we cannot rely on live hardware IDs for inactive monitors, we use the Instance ID (e.g., MONITOR\GSM5B09\...) as a fallback.

In synth.rs, we map these static registry strings to our own persistent_id. This allows DisplayFlow to:

    Look at a "Ghost" in the registry.

    Match its InstanceID to a saved Profile.

    Send a ChangeDisplaySettingsExW call to "wake it up" by assigning it a resolution and position.

4. Why this matters for the ddc-ci crate

As we move toward the ddc-ci crate, this distinction becomes even more critical. DDC/CI commands strictly require a physical handle. If a monitor is a "Ghost," DDC commands will fail 100% of the time.

Our Logic Flow:

    Step 1: Identify the monitor via Registry/Synthetic ID (Software Identity).

    Step 2: If it’s inactive, apply the GDI configuration first (The "Wake Up").

    Step 3: Once the monitor has a live handle, then send the DDC/CI commands for brightness and contrast.

Back to Project: [GitHub]({{< .Params.project_url >}}) | [Official Website]({{< .Params.landing_page >}}) | [Demo Video]({{< .Params.youtube_demo >}})