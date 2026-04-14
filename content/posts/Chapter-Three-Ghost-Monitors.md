---
title: "Chapter Three: Ghost Monitors & The EDID Void"
date: 2026-04-13
tags: ["Windows API", "EDID", "Ghosting", "Win32"]
description: "Why inactive monitors are invisible to hardware calls and how to bridge the gap."
project_url: "https://github.com/piot5/displayflow"
landing_page: "https://piot5.github.io/displayflow_cli/"
youtube_demo: "https://www.youtube.com/watch?v=_ch6X-_A2Ms"
---
<svg viewBox="0 0 800 500" xmlns="http://www.w3.org/2000/svg">
  <rect width="800" height="500" fill="#1a1b26" rx="8"/>
  <style>
    .text { font-family: 'Segoe UI', monospace; font-size: 14px; fill: #a9b1d6; }
    .header { font-weight: bold; font-size: 16px; fill: #7aa2f7; }
    .box { fill: #24283b; stroke: #414868; stroke-width: 2; }
    .arrow { stroke: #7aa2f7; stroke-width: 2; fill: none; marker-end: url(#arrowhead); }
    .ghost { fill: #444b6a; stroke-dasharray: 4; }
    .active { fill: #2ac3de; fill-opacity: 0.1; stroke: #2ac3de; }
    .warning { fill: #f7768e; font-weight: bold; }
  </style>
  
  <defs>
    <marker id="arrowhead" markerWidth="10" markerHeight="7" refX="0" refY="3.5" orient="auto">
      <polygon points="0 0, 10 3.5, 0 7" fill="#7aa2f7" />
    </marker>
  </defs>

  <rect x="50" y="50" width="300" height="150" class="box" />
  <text x="65" y="75" class="header">REGISTRY LAYER (Persistent)</text>
  <text x="65" y="105" class="text">SYSTEM\...\DISPLAY</text>
  <text x="65" y="125" class="text">↳ EDID (Cached Serial)</text>
  <text x="65" y="145" class="text">↳ InstanceID (GSM5B09...)</text>
  <text x="65" y="175" class="text" fill="#9ece6a">Result: Identity Found</text>

  <rect x="450" y="50" width="300" height="150" class="box ghost" />
  <text x="465" y="75" class="header">GDI LAYER (Hardware)</text>
  <text x="465" y="105" class="text">EnumDisplayMonitors()</text>
  <text x="465" y="135" class="warning">❌ Detached: No HMONITOR</text>
  <text x="465" y="155" class="warning">❌ DDC/CI: Fail (No Pulse)</text>

  <rect x="250" y="260" width="300" height="180" class="box active" />
  <text x="265" y="285" class="header">DISPLAYFLOW BRIDGE</text>
  <text x="265" y="315" class="text">1. scan.rs → Fetch Serial from Registry</text>
  <text x="265" y="340" class="text">2. synth.rs → Map InstanceID to Profile</text>
  <text x="265" y="365" class="text">3. Wake Up → ChangeDisplaySettingsExW</text>
  <text x="265" y="410" class="text" fill="#bb9af7" font-weight="bold">Action: Create Live Handle</text>

  <path d="M 200 200 L 250 260" class="arrow" />
  <path d="M 600 200 L 550 260" class="arrow" />
  <path d="M 400 440 L 400 480 L 700 480 L 700 200" class="arrow" />
  <text x="450" y="470" class="text" font-size="12">After Wake: DDC/CI commands enabled</text>

  <text x="350" y="30" class="warning" text-anchor="middle">THE EDID VOID: Identity vs. Accessibility</text>
</svg>
In a perfect world, every monitor would always be reachable. In reality, Windows often keeps "Ghost Monitors" in its registry—displays that were once connected but are now powered off or disconnected. These ghosts create a massive problem for automation: they exist in the GDI, but they have no hardware pulse.
## 1. The EDID Dead End

When a monitor is inactive or "detached" in Windows, the operating system stops polling its hardware bus. If you try to call GetVCPFeatureAndVCPFeatureReply or read the EDID via SetupDi functions on an inactive display, the hardware returns... nothing.

The Problem: You cannot identify a monitor's serial number if the monitor is currently disabled in the display settings.

This creates a "Chicken and Egg" problem: You need the serial number to know which monitor it is to enable it, but you have to enable it first to see the serial number.

## 2. Bridging the Gap in scan.rs

DisplayFlow solves this by splitting the scan into two distinct layers:

Live GDI Layer: Only sees what is currently "active" and has an HMONITOR handle.

Registry Layer: Scans SYSTEM\CurrentControlSet\Enum\DISPLAY to find every device the PC has ever seen.

In scan.rs, we iterate through the registry even for monitors that don't have a live handle:
Rust

// Snippet from scan.rs: Hunting for ghosts

```rust
if let Ok(params) = hw_key.open_subkey(format!(r"{}\Device Parameters", inst_id)) {
    if let Ok(edid) = params.get_raw_value("EDID") {
        let (sn, size, res) = parse_edid(&edid.bytes);
        // We found it! Even if the monitor is 'Off', 
        // Windows stores the last known EDID here.
    }
}
```

## 3. The "Synthetic ID" Solution

Since we cannot rely on live hardware IDs for inactive monitors, we use the Instance ID (e.g., MONITOR\GSM5B09\...) as a fallback.

In synth.rs, we map these static registry strings to our own persistent_id. This allows DisplayFlow to:

  Look at a "Ghost" in the registry.

  Match its InstanceID to a saved Profile.

  Send a ChangeDisplaySettingsExW call to "wake it up" by assigning it a resolution and position.

##4. Why this matters for the ddc-ci crate

As we move toward the ddc-ci crate, this distinction becomes even more critical. DDC/CI commands strictly require a physical handle. If a monitor is a "Ghost," DDC commands will fail 100% of the time.

Our Logic Flow:

	Step 1: Identify the monitor via Registry/Synthetic ID (Software Identity).

	Step 2: If it’s inactive, apply the GDI configuration first (The "Wake Up").

	Step 3: Once the monitor has a live handle, then send the DDC/CI commands for brightness and contrast.

Back to Project: [GitHub]({{< param project_url >}}) | [Official Website]({{< param landing_page >}}) | [Demo Video]({{< param youtube_demo >}})
