# Comprehensive RDP Optimization Guide (High-Fidelity)

This guide combines the Windows 10 host optimizations, the PowerShell implementation, and the macOS client configuration for the MacBook Pro.

## 1. Windows 10 Host Setup (The "Why")

To ensure "no compromise" image quality, the Windows host must be forced to use high-quality encoding (H.264/AVC 4:4:4) and hardware acceleration. This overrides the default "bandwidth-efficient" settings.

### Key Policies (Registry)
The following are applied to `HKLM\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services`:
*   `UserAVC = 1`: Enables AVC/H.264 encoding.
*   `AllowHigherFourFourFourEncoding = 1`: Forces 4:4:4 chroma subsampling (removes color artifacts in text/fine detail).
*   `VisualQuality = 2`: Sets RemoteFX quality to **Lossless**.
*   `HardwareGraphicsAdapter = 1`: Forces the use of a hardware GPU for encoding.
*   `SelectNetworkDetect = 1`: Optimizes network transport (disables legacy connect-time detection which can cause freezes in 24H2).

---

## 2. Implementation Script (`rdp_optimize.ps1`)

Run this script as **Administrator** on the Windows host to apply the settings.

```powershell
# RDP Optimization for "No Compromise" Image Viewing
$registryPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services"

# Ensure path exists
if (-not (Test-Path $registryPath)) { New-Item -Path $registryPath -Force | Out-Null }

# 1. Enable AVC/H.264 hardware encoding
Set-ItemProperty -Path $registryPath -Name "UserAVC" -Value 1 -Type DWord

# 2. Prioritize H.264/AVC 4:4:4 Graphics Mode
Set-ItemProperty -Path $registryPath -Name "AllowHigherFourFourFourEncoding" -Value 1 -Type DWord

# 3. Set Visual Quality to Lossless
Set-ItemProperty -Path $registryPath -Name "VisualQuality" -Value 2 -Type DWord

# 4. Enable Hardware Graphics Adapter for all sessions
Set-ItemProperty -Path $registryPath -Name "HardwareGraphicsAdapter" -Value 1 -Type DWord

# 5. Optimized Transport (Avoids freezes in Win11/Win10 updates)
Set-ItemProperty -Path $registryPath -Name "SelectNetworkDetect" -Value 1 -Type DWord

Write-Host "RDP Optimizations applied. Please reboot the host."
```

---

## 3. macOS Client Setup (Native Retina Detail)

For the MacBook Pro, the goal is to leverage the full display resolution for maximum image fidelity.

### Recommended Configuration: "Native Retina Mode"
This provides the highest possible detail, ensuring images are perfectly sharp.

1.  **In Microsoft Remote Desktop Connection Settings**:
    *   **Display Tab**: ✅ **Enable** "Optimize for Retina displays".
    *   **Color depth**: High Color (32 bit).
    *   **Graphics interpolation**: Set to **None**. (This prevents pixel smoothing, keeping the image sharp and accurate for medical viewing).
2.  **User Experience**:
    *   The Windows host will recognize the native high resolution of the MacBook.
    *   **Result**: No compression artifacts, maximum pixel density, and perfect color precision (via AVC 4:4:4).

### 🛠️ Fixing Tiny Text in IE / Legacy Apps
Older apps like Internet Explorer do not always scale correctly at native resolution. Use these solutions:

*   **Quick Fix (IE Zoom)**: Inside the Internet Explorer window, press **`Ctrl` + `+`** (plus) or **`Ctrl` + `Mouse Wheel Up`** to set the zoom to **150%** or **200%**. This enlarges text while keeping the Retina sharpness.
*   **System Fix (Recommended)**: Right-click the Windows Desktop > **Display Settings** > **Scale and Layout** > Set to **200%**. This is the standard "HiDPI" way to run a Mac display on Windows—it keeps images perfectly sharp but makes text and buttons readable.

> [!NOTE]
> If you use the **System Fix (200% Scaling)**, you must **Sign Out** and back in for all apps to display correctly.

---

## 4. Final Verification
Click the **Signal Strength (Bars)** icon in the top RDP bar:
*   It should say **"Hardware acceleration is in use"**.
*   Images should be artifact-free regardless of movement.
