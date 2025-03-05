# Xcode Toolchain Spy: Monitoring Xcode's Toolchain Executions

**Before you begin** you might want to explore [swift-build](https://github.com/swiftlang/swift-build) instead to find out about the inner workings of xcode-build, playground or spm

**Disclaimer:** Modifying Xcode's toolchain can lead to unexpected behavior, potential instability, and may violate Apple's terms of service. Proceed at your own risk. Ensure you have full backups and understand the implications of altering system binaries.

## Table of Contents

- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
  - [1. Create a Custom Toolchain](#1-create-a-custom-toolchain)
  - [2. Edit `ToolchainInfo.plist`](#2-edit-toolchaininfoplist)
  - [3. Add the Wrapper Script](#3-add-the-wrapper-script)
  - [4. Apply Wrappers to Binaries](#4-apply-wrappers-to-binaries)
  - [5. Select the Custom Toolchain in Xcode](#5-select-the-custom-toolchain-in-xcode)
- [Using the Spy Toolchain](#using-the-spy-toolchain)
- [Viewing Logs](#viewing-logs)
- [Troubleshooting](#troubleshooting)
- [License](#license)

## Introduction

This repository provides a method to monitor and log the execution of Xcode’s toolchain binaries. By creating a custom toolchain that wraps the original binaries, you can capture detailed logs of how Xcode interacts with its toolchain during build processes. This can be useful for debugging, performance analysis, or educational purposes.

## Prerequisites

- **Operating System:** macOS (compatible with your Xcode version)
- **Xcode:** Installed and updated to the latest version
- **Permissions:** Administrative access (`sudo`) to modify Xcode's toolchain
- **Backup Solution:** Ensure you have a complete backup of your system or, at minimum, Xcode installation

## Setup Instructions

Follow these steps to set up the custom toolchain for spying on Xcode.

### 1. **Create a Custom Toolchain**

Copy the default Xcode toolchain to your user directory to create a custom toolchain.

```
cp -R /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain ~/Library/Developer/Toolchains/MyCustomToolchain.xctoolchain
```
It could be that your toolchain is not part of Xcode and instead residesd in `/Library/Developer/Toolchains` or user level on `~/Library/Developer/Toolchains`

### 2. **Edit ToolchainInfo.plist**

Modify the ToolchainInfo.plist to recognize your custom toolchain.

  - Navigate to the Toolchain Directory:

```
cd ~/Library/Developer/Toolchains/MyCustomToolchain.xctoolchain/
```

  - Remove ToolchainInfo.plist. Open ToolchainInfo.plist for Editing:
You can use nano, vim, or any plist editor.

```
rm ToolchainInfo.plist
nano Info.plist
```

  - Update to match the Following (Inspired from Swift toolchain):

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>Aliases</key>
	<array>
		<string>swift</string>
	</array>
	<key>CFBundleIdentifier</key>
	<string>MyCustomToolchain</string>
	<key>CompatibilityVersion</key>
	<integer>2</integer>
	<key>CompatibilityVersionDisplayString</key>
	<string>Xcode 16.2</string>
	<key>CreatedDate</key>
	<date>2025-01-01T09:41:00Z</date>
	<key>DisplayName</key>
	<string>My Custom Toolchain</string>
	<key>OverrideBuildSettings</key>
	<dict>
		<key>ENABLE_BITCODE</key>
		<string>NO</string>
		<key>OTHER_SWIFT_FLAGS</key>
		<string>$(inherited) -plugin-path $(TOOLCHAIN_DIR)/usr/lib/swift/host/plugins</string>
		<key>SWIFT_DEVELOPMENT_TOOLCHAIN</key>
		<string>YES</string>
		<key>SWIFT_DISABLE_REQUIRED_ARCLITE</key>
		<string>YES</string>
		<key>SWIFT_LINK_OBJC_RUNTIME</key>
		<string>YES</string>
		<key>SWIFT_USE_DEVELOPMENT_TOOLCHAIN_RUNTIME</key>
		<string>YES</string>
	</dict>
	<key>ReportProblemURL</key>
	<string>https://bugs.swift.org/</string>
	<key>ShortDisplayName</key>
	<string>My custom toolchain</string>
	<key>Version</key>
	<string>1.0.0</string>
</dict>
</plist>
```

  - Save and Exit:
	-	If using nano, press CTRL + O to save, then CTRL + X to exit.

 ### 3. **Add the Wrapper Script**

Create the wrapper script that will log invocations and results of toolchain binaries.
  - Create the Wrapper Script File:

```
touch wrap_binaries.sh
chmod +x wrap_binaries.sh
```

  - Open the Script for Editing:

```
nano wrap_binaries.sh
```

  - Paste the Following Script:

```
#!/bin/bash

# ======================================================================
# Script Name: wrap_binaries.sh
# Description: Wraps all executables in the custom Xcode toolchain's
#              usr/bin directory to log invocations and results,
#              then execute the original binaries in the correct context.
# Usage: sudo ./wrap_binaries.sh
# ======================================================================

# --------------------------
# Ensure the script is run as root
# --------------------------
if [ "$EUID" -ne 0 ]; then
  echo "Error: Please run this script with sudo or as root."
  exit 1
fi

# --------------------------
# Define Paths and Variables
# --------------------------
TOOLCHAIN_BIN_DIR="$(pwd)"
LOG_FILE="$HOME/xcode_toolchain_logs.txt"

# --------------------------
# Function to Create Wrapper
# --------------------------
create_wrapper() {
  local bin_path="$1"
  local bin_name="$(basename "$bin_path")"

  # Remove bin
  rm -f "$bin_name"

  # Create the wrapper script
  echo "Creating wrapper for $bin_name..."
  cat <<EOF > "$TOOLCHAIN_BIN_DIR/$bin_name"
#!/bin/bash

# Define paths
LOG_FILE="$LOG_FILE"

# Log the invocation with timestamp and arguments
echo "$bin_name called with arguments: \$@" >> "\$LOG_FILE"

# Execute the original binary, replacing the current shell
exec "$(xcode-select -p)/Toolchains/XcodeDefault.xctoolchain/usr/bin/$bin_name" "\$@"

EOF

  # Make the wrapper executable
  chmod +x "$TOOLCHAIN_BIN_DIR/$bin_name"
}

# --------------------------
# Iterate Over Each Binary
# --------------------------
echo "Starting to wrap binaries in $TOOLCHAIN_BIN_DIR..."

for BIN_PATH in "$TOOLCHAIN_BIN_DIR"/*; do
  BIN_NAME="$(basename "$BIN_PATH")"

  # Skip directories
  if [ -d "$BIN_PATH" ]; then
    continue
  fi

  # Skip this file
  if [ "$BIN_NAME" == "wrap_binaries.sh" ]; then
    continue
  fi

  # Check if it's a regular file and executable
  if [ -f "$BIN_PATH" ] && [ -x "$BIN_PATH" ]; then
    create_wrapper "$BIN_PATH"
  fi
done

echo "All applicable executables have been wrapped successfully."
echo "Log file located at: $LOG_FILE"

```

  - Save and Exit:
	-	If using nano, press CTRL+O to save, then CTRL+X to exit.

### 4. **Apply Wrappers to Binaries**

Execute the wrap_binaries.sh script to wrap all relevant binaries in your custom toolchain.
  - Navigate to the Custom Toolchain’s usr/bin/ Directory:

```
cd ~/Library/Developer/Toolchains/MyCustomToolchain.xctoolchain/usr/bin/
```

  - Place the Wrapper Script in the Directory:
Ensure wrap_binaries.sh is in the usr/bin/ directory.

```
cp /path/to/your/wrap_binaries.sh .
chmod +x wrap_binaries.sh
```

Replace /path/to/your/wrap_binaries.sh with the actual path where you created the script.

  - Run the Wrapper Script with sudo:

```
sudo ./wrap_binaries.sh
```

-	The script will create wrapper scripts that log invocations and results, then execute the original binaries.
  
Note: This process may take a few minutes depending on the number of binaries.

### 5. **Select the Custom Toolchain in Xcode**

-	Open Xcode.
-	Navigate to Preferences:
  -	Click on Xcode in the menu bar.
  -	Select Preferences... (or press ⌘ + ,).
-	Select the Toolchains Tab:
  -	Click on the Toolchains tab.
-	Choose Your Custom Toolchain:
  -	Your custom toolchain (My Custom Toolchain) should appear in the list.
  -	Select it to activate.

## Using the Spy Toolchain

With the custom toolchain selected, proceed to build your projects as usual. The wrapper scripts will automatically log all invocations and results of the toolchain binaries used during the build process.

## Viewing Logs

All logs are stored in the ~/xcode_toolchain_logs.txt file.
	1.	Open Terminal.
	2.	View the Log File:

```
cat ~/xcode_toolchain_logs.txt
```

Sample Log Entries:

```
2024-04-27 12:34:56 : swiftc called with arguments: --version
2024-04-27 12:34:56 : swiftc output: Apple Swift version 5.7 (swiftlang-1300.0.29 clang-1300.0.29.1)
Target: x86_64-apple-darwin20.3.0
2024-04-27 12:34:56 : swiftc exited with status: 0

2024-04-27 12:35:10 : swiftc called with arguments: main.swift -o main
2024-04-27 12:35:10 : swiftc output: (no output if successful)
2024-04-27 12:35:10 : swiftc exited with status: 0
```

## Troubleshooting
	
-	dyld Errors:
    -	Ensure that the wrapper scripts change the directory to usr/bin before executing the original binaries.
    -	Verify that the ORIGINAL_BIN path is correct in the wrapper scripts.
-	Logging Not Working:
    -	Check permissions of the log file (~/xcode_toolchain_logs.txt).
    -	Ensure that the wrapper scripts are executable.
-	Xcode Build Failures:
    -	Revert changes using the Reverting Changes section.
    -	Check the log file for any logged errors.
-	Performance Issues:
    -	Implement log rotation to prevent large log files from impacting performance.
    -	Optimize wrapper scripts to minimize overhead.

Be cautious about the information being logged to prevent accidental exposure of proprietary code or sensitive data.

## License

MIT License

Note: This repository is intended for educational and debugging purposes. Unauthorized modification of system or application binaries can lead to security vulnerabilities and violate software licensing agreements. Always ensure compliance with applicable laws and software terms of service.
