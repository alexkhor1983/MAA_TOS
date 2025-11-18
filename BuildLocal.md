# Local Build Guide

## Understanding the GitHub Workflows

Based on analyzing both workflows, here's what they do:

### **check.yml** - Validation Workflow

- **Purpose**: Validates resource files (pipelines, images) are correctly formatted
- **Process**: Installs `maafw` Python package and runs `check_resource.py` to validate assets

### **install.yml** - Build & Package Workflow

- **Purpose**: Creates distributable packages with the executable
- **Key Steps**:
  1. Downloads **MaaFramework** binaries from [MaaXYZ/MaaFramework](https://github.com/MaaXYZ/MaaFramework)
  2. Downloads **MFAAvalonia.exe** (the GUI executable) from [SweetSmellFox/MFAAvalonia](https://github.com/SweetSmellFox/MFAAvalonia)
  3. Runs `install.py` to package everything
  4. Copies MFAAvalonia.exe to the install folder

## How to Build Locally (Windows)

To generate the executable locally, follow these steps:

### **Step 1: Download MaaFramework**

1. Go to <https://github.com/MaaXYZ/MaaFramework/releases/latest>
2. Download `MAA-win-x86_64-*.zip` (or `aarch64` for ARM)
3. Extract it to a folder named `deps` in your project root
4. The structure should be: `C:\Development\MAA_TOS\deps\bin\`, `deps\lib\`, etc.

### **Step 2: Download MFAAvalonia (GUI Executable)**

1. Go to <https://github.com/SweetSmellFox/MFAAvalonia/releases/latest>
2. Download the MFAAvalonia Windows zip file
3. Extract it to a folder named `MFA` in your project root
4. You should have `C:\Development\MAA_TOS\MFA\MFAAvalonia.exe` (or the executable inside a nested folder)

### **Step 3: Run the Install Script**

```bash
cd C:\Development\MAA_TOS
python install.py v1.0.0
```

This will:

- Copy MaaFramework binaries to `install/`
- Copy your assets/resources to `install/`
- Configure OCR models
- Copy agent files

### **Step 4: Copy the GUI Executable**

```powershell
$gui = Get-ChildItem -Path .\MFA -Filter "MFAAvalonia*.exe" -Recurse | Select-Object -First 1
if (-not $gui) { throw "MFAAvalonia executable not found under .\MFA" }
Copy-Item $gui.FullName .\install\MFAAvalonia.exe
```

### **Step 5: Update interface.json (Optional)**

If you want to customize the project metadata you can edit `install/interface.json` manually, or use PowerShell with `jq` (if you have it installed):

```powershell
# Install jq if needed: choco install jq
jq --arg version_name "Your Project Name" --arg url "https://github.com/YourUsername/MAA_TOS" --arg version "v1.0.0" '. + {"name": $version_name, "url": $url, "version": $version}' .\assets\interface.json > .\install\interface.json
```

### **Result**

Your complete distributable package will be in the `install/` folder with:

- `MFAAvalonia.exe` — The main GUI executable
- MaaFramework DLLs
- Your game automation resources (pipelines, images, etc.)
