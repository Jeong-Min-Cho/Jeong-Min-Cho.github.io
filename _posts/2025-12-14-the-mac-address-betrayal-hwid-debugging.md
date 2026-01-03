---
layout: post
title: "The MAC Address Betrayal: When Hardware IDs Go Rogue"
date: 2025-12-14 10:00 +0900
categories: [Programming, Debugging]
tags: [ShieldMod, C++, Windows, Hardware, HWID, Debugging]
image:
  path: /assets/img/posts/shieldmod/post3-3.jpg
  alt: Debugging HWID issues meme
---

## Monday Morning Massacre

It was a Monday morning. I opened my support dashboard expecting the usual trickle of questions. Instead, I found **47 HWID reset requests**.

Forty. Seven.

All with the same complaint: "My hardware ID changed and I can't log in anymore."

![Disaster Girl](https://i.kym-cdn.com/photos/images/newsfeed/000/000/130/disaster-girl.jpg)
*Me, the architect of my own Monday morning disaster*

## The Crime Scene

### Impact Report
- **Affected users:** ~30% of user base
- **Support tickets:** 47 HWID reset requests in one week
- **Time to resolve:** 2 days
- **Pattern noticed:** Most reports came Monday morning (after weekend reboots)

The pattern was suspicious. Users weren't getting new computers - they were rebooting them. Something in my HWID generation was changing between reboots.

## The Investigation

First, let me show you what my original HWID generator looked like:

```cpp
// The ORIGINAL code (BROKEN)
std::string GenerateHWID() {
    std::string components;
    components += GetCPUID();           // Stable ✓
    components += GetMotherboardSerial(); // Stable ✓
    components += GetMACAddress();       // UNSTABLE ✗ ← THE TRAITOR
    components += GetWindowsProductID(); // Stable ✓
    return SHA256(components);
}
```

Can you spot the problem? The MAC address.

"But wait," I thought, "MAC addresses are burned into the network card. They're supposed to be permanent!"

![Sweet Summer Child](https://i.imgflip.com/2gnnjh.jpg)
*Oh, my sweet summer child... thinking MAC addresses are reliable*

## Why MAC Addresses Are Liars

Here's what I learned the hard way:

| Scenario | What Happens to MAC Address |
|----------|-------------------|
| VPN software installed | Adds 1-2 virtual adapters |
| Docker running | Adds a bridge adapter |
| Network adapter disabled | Order of adapters changes |
| Windows Update | Adapter re-enumeration |
| USB WiFi unplugged | Missing from list entirely |

My code was getting the "first" MAC address. But the order of network adapters isn't guaranteed, and new adapters appear and disappear like ghosts.

### The Damning Evidence

I added logging and found this gem:

```
User: john_doe
HWID at login (Monday):      a]
HWID at login (Friday):      a3b5c7d9...  (same - he didn't reboot all week)
HWID at login (next Monday): f8e6d4c2...  (DIFFERENT!)

Difference traced to: MAC address changed from
  00:1A:2B:3C:4D:5E to 00:1A:2B:3C:4D:5F

Cause: VPN adapter order changed after weekend reboot
```

The user didn't change anything. They just... rebooted. With a VPN client installed. That's it. That's the crime.

![They're The Same Picture](/assets/img/posts/shieldmod/post3-3.jpg)
*Corporate: Find the difference between these two MAC addresses. Me: They're... wait, they're different now?!*

## The Fix

### Step 1: Remove the Traitor

The first thing was obvious - stop using MAC addresses:

```cpp
// FIXED: Only use stable hardware identifiers
std::string GenerateHWID() {
    std::string components;

    // Component 1: CPU ID (stable, unique per processor)
    int cpuInfo[4];
    __cpuid(cpuInfo, 0);
    components += std::to_string(cpuInfo[1]);
    components += std::to_string(cpuInfo[3]);

    // Component 2: Motherboard serial (stable, unique per board)
    components += GetWMIProperty("Win32_BaseBoard", "SerialNumber");

    // Component 3: Windows Product ID (stable after activation)
    components += GetRegistryValue(
        "SOFTWARE\\Microsoft\\Windows NT\\CurrentVersion",
        "ProductId"
    );

    // Component 4: BIOS serial (stable)
    components += GetWMIProperty("Win32_BIOS", "SerialNumber");

    // REMOVED: MAC address - too unstable
    // components += GetMACAddress(); // NOPE. NEVER AGAIN.

    return SHA256(components);
}
```

### Step 2: Add Fuzzy Matching

What if someone replaces their motherboard? Or reinstalls Windows? We need some flexibility:

```cpp
float CalculateHWIDSimilarity(const std::string& stored, const std::string& current) {
    // Parse into individual component hashes
    auto storedParts = ParseHWIDComponents(stored);   // 4 components
    auto currentParts = ParseHWIDComponents(current); // 4 components

    int matches = 0;
    for (size_t i = 0; i < storedParts.size(); i++) {
        if (storedParts[i] == currentParts[i]) {
            matches++;
        }
    }

    return static_cast<float>(matches) / storedParts.size();
}

bool IsValidHWID(const std::string& stored, const std::string& current) {
    // Exact match - perfect
    if (stored == current) return true;

    // Allow if 3 of 4 components match (75%+)
    // Handles: single component change (e.g., Windows reinstall)
    return CalculateHWIDSimilarity(stored, current) >= 0.75f;
}
```

This means if a user changes ONE thing (reinstalls Windows, replaces a hard drive), they're still recognized. But if they try to use someone else's license, the CPUID and motherboard serial won't match.

## The Hardware Stability Tier List

After this incident, I created this chart for myself:

```
S-TIER (Never changes):
├── CPU ID (literally etched in silicon)
├── Motherboard Serial
└── BIOS Serial

A-TIER (Rarely changes):
├── Windows Product ID (only on reinstall)
└── Hard Drive Serial (only on replacement)

B-TIER (Sometimes changes):
├── GPU ID (if user upgrades)
└── RAM configuration (if user adds more)

F-TIER (DON'T TRUST):
├── MAC Address ← LIES
├── IP Address ← LIES
├── Hostname ← USER CAN CHANGE
└── Computer Name ← USER CAN CHANGE
```

![Trust Nobody](https://i.kym-cdn.com/entries/icons/original/000/017/046/BptVE1JIEAAA3dT.jpg)
*Trust nobody, not even your network adapters*

## The Results

| Metric | Before Fix | After Fix |
|--------|------------|-----------|
| HWID changes after reboot | 30% of users | 0.1% of users |
| Weekly reset requests | 47 | 2 |
| False positive rate | ~28% | <0.5% |
| Component stability | 3/4 stable | 4/4 stable |

The 0.1% that still have issues? Usually VM users or people with really exotic hardware configurations. Those I handle manually.

## Lessons Learned

1. **Test with real-world conditions** - I tested HWID generation on my dev machine. I didn't test after installing VPN software, running Docker, or rebooting with different network states.

2. **Trust only what's physically soldered** - The CPU, motherboard, and BIOS are the only truly stable identifiers. Everything else can change.

3. **Build in tolerance** - The 75% match threshold means users don't lose access from minor changes.

4. **Add logging before you need it** - The only way I found the bug was by logging the individual components. Without that, I'd still be guessing.

## The Final Meme

**Gru's Plan:**

1. Use MAC address for stable HWID
2. Ship to production
3. 47 support tickets Monday morning
4. 47 support tickets Monday morning 😱

---

*Next time: How 5 simultaneous API requests all decided to refresh the auth token at the same time, and how a simple mutex pattern saved the day.*

*This is part 3 of my "Building ShieldMod" series.*
