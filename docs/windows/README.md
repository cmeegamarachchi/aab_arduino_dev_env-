# Windows Development Guide

## Arduino Uno R3 Dev Container

This guide describes how to develop, compile, upload, and monitor Arduino Uno R3 projects from a **VS Code Dev Container** on a Windows workstation.

The intended development stack is:

- Windows 11
- WSL2
- A Linux distribution such as Ubuntu
- Docker
- Visual Studio Code
- VS Code Dev Containers
- Arduino CLI
- Arduino Uno R3 connected to the Windows host by USB
- `usbipd-win` for making the USB device available to WSL2

The Arduino compiler, AVR toolchain, Arduino board core, and project dependencies are intended to live inside the development container. Windows is primarily responsible for hosting VS Code and the physical USB connection.

> This document is Windows-specific.  
> General project information is available in the repository `README.md`.

---

# 1. Architecture

The development path is:

```text
┌───────────────────────────────────────────┐
│ Windows 11                                │
│                                           │
│  Visual Studio Code                       │
│  usbipd-win                               │
│                                           │
│  Arduino Uno R3                           │
│       │                                   │
│       └──────── USB ──────────────────────┤
└─────────────────────┬─────────────────────┘
                      │
                      │ USB/IP
                      ▼
┌───────────────────────────────────────────┐
│ WSL2                                      │
│                                           │
│  Ubuntu / Linux distribution              │
│                                           │
│  Arduino appears as a Linux device        │
│  e.g. /dev/ttyACM0                        │
│                                           │
│  Project source                           │
│  ~/projects/arduino-project               │
└─────────────────────┬─────────────────────┘
                      │
                      ▼
┌───────────────────────────────────────────┐
│ Docker / VS Code Dev Container            │
│                                           │
│  Arduino CLI                              │
│  Arduino AVR Core                         │
│  AVR-GCC                                  │
│  AVRDUDE                                  │
│  Project libraries                        │
│  VS Code development tools                │
│                                           │
│  Compile                                  │
│  Upload                                   │
│  Serial Monitor                           │
└───────────────────────────────────────────┘
```

The important concept is that there are **three separate boundaries**:

1. Windows must recognise the Arduino USB device.
2. WSL2 must be given access to that device.
3. The Docker container must be given access to the resulting Linux device.

When troubleshooting, test these layers in that order.

---

# 2. Recommended Windows Configuration

For the simplest development experience:

- Use a current Windows 11 installation.
- Keep WSL updated.
- Use WSL2, not WSL1.
- Use Ubuntu or another normal Linux distribution as the development distribution.
- Store the source repository inside the WSL filesystem.
- Open the project through VS Code's WSL integration.
- Start the Dev Container from that WSL-based VS Code session.

For example:

```text
/home/chi/projects/arduino-dev/
```

is preferable to:

```text
/mnt/c/Users/chi/projects/arduino-dev/
```

Docker recommends keeping Linux-oriented development source inside the Linux distribution when using the WSL2 development workflow. This normally provides better Linux filesystem semantics and performance.

---

# 3. Prerequisites

This guide assumes that the workstation already has:

- Windows 11
- WSL2
- Ubuntu or another supported Linux distribution
- Docker
- Visual Studio Code
- VS Code WSL extension
- VS Code Dev Containers extension

Verify WSL from PowerShell:

```powershell
wsl --version
```

List installed distributions:

```powershell
wsl --list --verbose
```

Example:

```text
  NAME      STATE           VERSION
* Ubuntu    Running         2
```

The development distribution must report:

```text
VERSION 2
```

If required, update WSL:

```powershell
wsl --update
```

Then restart WSL:

```powershell
wsl --shutdown
```

Open Ubuntu again after the shutdown.

---

# 4. Verify Docker from WSL

Open the Ubuntu WSL terminal and run:

```bash
docker version
```

and:

```bash
docker run --rm hello-world
```

Both commands should complete successfully before attempting Arduino USB passthrough.

If Docker Desktop is being used, make sure its WSL integration is enabled for the Ubuntu distribution used for development.

In Docker Desktop this is normally under:

```text
Settings
  → Resources
    → WSL Integration
```

The selected Ubuntu distribution should be enabled.

---

# 5. Install usbipd-win

WSL does not directly expose arbitrary Windows USB devices to Linux. Microsoft documents `usbipd-win` as the mechanism for attaching USB devices to WSL2, including development scenarios such as flashing Arduino boards.

Install it from an Administrator PowerShell terminal:

```powershell
winget install --interactive --exact dorssel.usbipd-win
```

After installation, verify:

```powershell
usbipd --version
```

You can also list connected USB devices:

```powershell
usbipd list
```

---

# 6. Connect the Arduino Uno R3

Connect the Uno R3 to the Windows host using a USB data cable.

A charge-only USB cable will power the board but will not provide a programming connection.

Run:

```powershell
usbipd list
```

You should see a USB device corresponding to the Arduino or its USB serial interface.

For example:

```text
BUSID  VID:PID    DEVICE                              STATE
4-2    2341:0043  Arduino Uno                         Not shared
```

The exact description and VID/PID may differ, especially for compatible Uno boards using a different USB-to-serial chipset.

Record the `BUSID`.

In the examples below:

```text
4-2
```

is used only as an example.

---

# 7. Share the Arduino with WSL

The first time a device is used, it must be shared.

Open **Administrator PowerShell** and run:

```powershell
usbipd bind --busid 4-2
```

Verify:

```powershell
usbipd list
```

The device should now appear as shared.

The `bind` operation normally persists, so it usually does not need to be repeated every time the board is connected.

---

# 8. Attach the Arduino to WSL

Keep an Ubuntu/WSL terminal open.

From a normal PowerShell terminal run:

```powershell
usbipd attach --wsl --busid 4-2
```

Administrator privileges are normally not required for the attach operation after the device has been bound.

Verify from Windows:

```powershell
usbipd list
```

The device should show as attached.

While the device is attached to WSL, Windows applications generally cannot use it directly.

---

# 9. Verify the Arduino Inside WSL

In Ubuntu, install `usbutils` if necessary:

```bash
sudo apt update
sudo apt install -y usbutils
```

Then run:

```bash
lsusb
```

The Arduino should appear in the list.

You can also inspect recent kernel messages:

```bash
dmesg | tail -n 30
```

For an Uno R3, Linux will commonly create a serial device such as:

```text
/dev/ttyACM0
```

Check:

```bash
ls -l /dev/ttyACM*
```

If no device exists, also check:

```bash
ls -l /dev/ttyUSB*
```

Some compatible boards use USB-to-serial chips that are exposed as `ttyUSB` rather than `ttyACM`.

A useful combined command is:

```bash
ls -l /dev/ttyACM* /dev/ttyUSB* 2>/dev/null
```

---

# 10. Serial Port Permissions

The current Linux user must be allowed to access the serial device.

Check the device:

```bash
ls -l /dev/ttyACM0
```

A typical Linux result looks similar to:

```text
crw-rw---- 1 root dialout ... /dev/ttyACM0
```

If the device belongs to the `dialout` group, add the WSL user to it:

```bash
sudo usermod -aG dialout "$USER"
```

Then close the WSL session and start it again.

Verify:

```bash
groups
```

You should see:

```text
dialout
```

Arduino's Linux documentation recommends the `dialout` group for serial-port access.

## Temporary Permission Test

If you are diagnosing a permissions problem, you can temporarily run:

```bash
sudo chmod a+rw /dev/ttyACM0
```

This is useful as a test, but it is not the preferred permanent solution because the permission may disappear when the board is disconnected and reconnected.

For persistent permissions, use the appropriate Linux group or `udev` rules.

---

# 11. Optional Arduino udev Rules

Arduino publishes a setup script for Linux `udev` rules.

If permissions remain problematic, the rules can be installed inside the WSL distribution:

```bash
wget https://content.arduino.cc/assets/arduino-udev-setup.sh
sudo bash arduino-udev-setup.sh
```

or:

```bash
curl -O https://content.arduino.cc/assets/arduino-udev-setup.sh
sudo bash arduino-udev-setup.sh
```

After changing rules, detach and reattach the USB device.

Because this project ultimately accesses the device from a container, the container user must also have suitable access.

---

# 12. Important Docker Runtime Note

Seeing:

```text
/dev/ttyACM0
```

inside Ubuntu proves that **WSL2** can access the Arduino.

It does **not necessarily prove** that the Docker runtime being used by VS Code can access the device.

This distinction is particularly important with Docker Desktop.

Docker's current documentation states that Docker Desktop does not provide generic direct USB passthrough in the same way a native Linux Docker host does. Docker Desktop provides a USB/IP workflow for supported backends instead.

For this project, always perform the container-device test below before configuring the Dev Container.

---

# 13. Test Arduino Access from Docker

Assuming the Arduino appears as:

```text
/dev/ttyACM0
```

run from the WSL terminal:

```bash
docker run --rm \
    --device=/dev/ttyACM0:/dev/ttyACM0 \
    ubuntu:24.04 \
    ls -l /dev/ttyACM0
```

If successful, the container should display the device.

For example:

```text
crw-rw---- 1 root dialout ... /dev/ttyACM0
```

This is the key test.

If it succeeds, the normal Dev Container configuration described later in this document should work.

If it fails with an error such as:

```text
error gathering device information while adding custom device
```

or:

```text
no such file or directory
```

then the Docker daemon cannot currently see the device even though the WSL distribution can.

See **Docker Desktop and USB Passthrough** later in this guide.

---

# 14. Determine Which Docker Engine You Are Using

Run:

```bash
docker context show
```

Then:

```bash
docker info
```

If using Docker Desktop, you will normally see references to Docker Desktop in the server information.

You can also inspect:

```bash
docker context ls
```

This matters because device passthrough is straightforward when the Docker daemon runs directly in the Linux environment that owns `/dev/ttyACM0`, but Docker Desktop adds another virtualization/runtime boundary.

---

# 15. Docker Desktop and USB Passthrough

Docker Desktop and WSL2 are closely integrated, but they are not the same Linux environment.

Docker Desktop uses its own internal environment for the Docker Engine.

Docker's documentation currently states:

> Docker Desktop does not support direct USB device passthrough.

Docker documents USB/IP as the supported mechanism for forwarding USB devices into the Docker Desktop VM.

The official Docker Desktop USB/IP guide currently applies to:

- Docker Desktop for Mac
- Docker Desktop for Linux
- Docker Desktop for Windows using the Hyper-V backend

This means a Windows development machine using the **Docker Desktop WSL2 backend** needs special attention.

## Practical Rule for This Project

Do not change a working Docker installation unnecessarily.

First run:

```bash
docker run --rm \
    --device=/dev/ttyACM0:/dev/ttyACM0 \
    ubuntu:24.04 \
    ls -l /dev/ttyACM0
```

### If it works

Continue with the Dev Container configuration.

### If it does not work

You have two architectural options:

1. Use a Docker Engine whose daemon runs directly inside the WSL Linux environment where the Arduino device exists.
2. Use Docker Desktop's supported USB/IP architecture on a backend for which Docker documents USB/IP support.

Do not casually install a second Docker Engine beside an actively integrated Docker Desktop configuration. Docker specifically warns that installing Docker Engine/CLI directly in a WSL distribution while also using Docker Desktop WSL integration can cause conflicts.

Choose one Docker architecture deliberately.

---

# 16. Recommended Repository Layout

A simple repository structure is:

```text
arduino-dev/
├── .devcontainer/
│   ├── devcontainer.json
│   └── Dockerfile
│
├── .vscode/
│   └── tasks.json
│
├── docs/
│   └── windows-development-guide.md
│
├── libraries/
│
├── arduino-dev.ino
│
└── README.md
```

For conventional Arduino sketches, it is useful for the primary `.ino` file to match the sketch directory name.

Example:

```text
arduino-dev/
└── arduino-dev.ino
```

---

# 17. Dev Container Dockerfile

The following is a suitable starting point:

```dockerfile
FROM mcr.microsoft.com/devcontainers/base:ubuntu-24.04

ARG ARDUINO_CLI_VERSION=1.5.1

USER root

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        usbutils \
    && rm -rf /var/lib/apt/lists/*

RUN curl -fsSL \
        https://raw.githubusercontent.com/arduino/arduino-cli/master/install.sh \
    | BINDIR=/usr/local/bin sh -s "${ARDUINO_CLI_VERSION}"

USER vscode

RUN arduino-cli config init \
    && arduino-cli core update-index \
    && arduino-cli core install arduino:avr@1.8.6
```

This deliberately installs:

- Arduino CLI
- Arduino AVR Boards
- AVR compiler/upload tools pulled in by the Arduino platform

The Arduino CLI version and AVR core are pinned so that rebuilding the container does not unexpectedly move the project to a newer toolchain.

At the time this document was prepared:

```text
Arduino CLI:       1.5.1
Arduino AVR Core:  1.8.6
```

These values can be updated deliberately after testing rather than changing implicitly.

---

# 18. Dev Container Configuration

Create:

```text
.devcontainer/devcontainer.json
```

with:

```json
{
    "name": "Arduino Uno R3 Development",

    "build": {
        "dockerfile": "Dockerfile",
        "context": ".."
    },

    "runArgs": [
        "--device=/dev/ttyACM0:/dev/ttyACM0"
    ],

    "containerEnv": {
        "ARDUINO_FQBN": "arduino:avr:uno",
        "ARDUINO_PORT": "/dev/ttyACM0",
        "ARDUINO_UPDATER_ENABLE_NOTIFICATION": "false"
    },

    "remoteUser": "vscode",

    "customizations": {
        "vscode": {
            "extensions": [
                "ms-vscode.cpptools"
            ]
        }
    }
}
```

This configuration:

- builds the project-specific development image,
- exposes the Arduino serial device to the container,
- defines the Uno R3 FQBN,
- defines the expected serial port,
- runs VS Code as the `vscode` user.

---

# 19. Container Serial Permissions

Even when Docker successfully exposes `/dev/ttyACM0`, the non-root `vscode` user may not initially have permission to use it.

Check from inside the Dev Container:

```bash
ls -l /dev/ttyACM0
```

Then:

```bash
id
```

For a local single-user development workstation, a convenient development-only approach is to add the following to `devcontainer.json`:

```json
"postStartCommand": "sudo chmod a+rw /dev/ttyACM0 || true"
```

For example:

```json
{
    "name": "Arduino Uno R3 Development",

    "build": {
        "dockerfile": "Dockerfile",
        "context": ".."
    },

    "runArgs": [
        "--device=/dev/ttyACM0:/dev/ttyACM0"
    ],

    "containerEnv": {
        "ARDUINO_FQBN": "arduino:avr:uno",
        "ARDUINO_PORT": "/dev/ttyACM0",
        "ARDUINO_UPDATER_ENABLE_NOTIFICATION": "false"
    },

    "postStartCommand": "sudo chmod a+rw /dev/ttyACM0 || true",

    "remoteUser": "vscode",

    "customizations": {
        "vscode": {
            "extensions": [
                "ms-vscode.cpptools"
            ]
        }
    }
}
```

This is acceptable for a local development container but is intentionally permissive.

For shared or security-sensitive environments, use proper Linux device groups and `udev` permissions instead.

---

# 20. Open the Repository through WSL

Open the WSL terminal:

```powershell
wsl
```

Navigate to the repository:

```bash
cd ~/projects/arduino-dev
```

Launch VS Code:

```bash
code .
```

VS Code should open with an indication that it is connected to WSL.

Then use the Command Palette:

```text
Dev Containers: Reopen in Container
```

VS Code will:

1. build the development image,
2. create the container,
3. mount the project,
4. attach VS Code to the container.

---

# 21. Verify the Development Container

Open the integrated terminal inside the Dev Container.

Check Arduino CLI:

```bash
arduino-cli version
```

Check installed cores:

```bash
arduino-cli core list
```

You should see:

```text
arduino:avr
```

Check the device:

```bash
ls -l "$ARDUINO_PORT"
```

Check board detection:

```bash
arduino-cli board list
```

A genuine Uno R3 will normally be identified automatically.

Even if Arduino CLI reports the board as unknown, upload can still work if the correct port and FQBN are supplied manually.

The project target is:

```text
arduino:avr:uno
```

---

# 22. First Test Sketch

A simple Blink sketch is ideal for validating the complete toolchain.

Create or update the project `.ino` file:

```cpp
void setup()
{
    pinMode(LED_BUILTIN, OUTPUT);
}

void loop()
{
    digitalWrite(LED_BUILTIN, HIGH);
    delay(1000);

    digitalWrite(LED_BUILTIN, LOW);
    delay(1000);
}
```

---

# 23. Compile the Project

Inside the Dev Container:

```bash
arduino-cli compile \
    --fqbn "$ARDUINO_FQBN" \
    .
```

For the Uno R3, this resolves to:

```bash
arduino-cli compile \
    --fqbn arduino:avr:uno \
    .
```

A successful build should report memory usage similar to:

```text
Sketch uses ... bytes (...) of program storage space.
Global variables use ... bytes (...) of dynamic memory.
```

Compilation does not require the physical board to be connected.

This is useful because normal coding and CI builds can run without hardware.

---

# 24. Upload to the Arduino

With the board attached and accessible:

```bash
arduino-cli upload \
    --port "$ARDUINO_PORT" \
    --fqbn "$ARDUINO_FQBN" \
    .
```

Equivalent:

```bash
arduino-cli upload \
    --port /dev/ttyACM0 \
    --fqbn arduino:avr:uno \
    .
```

Arduino CLI will invoke the AVR upload tooling required by the installed board core.

After upload, the Uno should reset and execute the new sketch.

For Blink, the built-in LED should flash approximately once per second.

---

# 25. Serial Monitor

For a sketch using:

```cpp
Serial.begin(9600);
```

start the serial monitor with:

```bash
arduino-cli monitor \
    --port "$ARDUINO_PORT" \
    --config baudrate=9600
```

Exit using:

```text
Ctrl+C
```

Arduino CLI's monitor is intentionally fairly simple. More advanced serial-terminal tools can be added to the container later if required.

---

# 26. VS Code Tasks

Common Arduino operations can be exposed as VS Code tasks.

Create:

```text
.vscode/tasks.json
```

with:

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Arduino: Board List",
            "type": "shell",
            "command": "arduino-cli board list",
            "problemMatcher": []
        },
        {
            "label": "Arduino: Compile",
            "type": "shell",
            "command": "arduino-cli compile --fqbn ${env:ARDUINO_FQBN} ${workspaceFolder}",
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "problemMatcher": []
        },
        {
            "label": "Arduino: Upload",
            "type": "shell",
            "command": "arduino-cli upload --port ${env:ARDUINO_PORT} --fqbn ${env:ARDUINO_FQBN} ${workspaceFolder}",
            "problemMatcher": []
        },
        {
            "label": "Arduino: Compile & Upload",
            "dependsOrder": "sequence",
            "dependsOn": [
                "Arduino: Compile",
                "Arduino: Upload"
            ],
            "problemMatcher": []
        },
        {
            "label": "Arduino: Serial Monitor",
            "type": "shell",
            "command": "arduino-cli monitor --port ${env:ARDUINO_PORT} --config baudrate=9600",
            "problemMatcher": []
        }
    ]
}
```

The normal build shortcut:

```text
Ctrl+Shift+B
```

will run the default compile task.

Other tasks can be started from:

```text
Terminal
  → Run Task...
```

---

# 27. Daily Development Workflow

Once the workstation has been configured, a normal development session should be:

## Step 1 — Connect the Arduino

Plug the Uno R3 into Windows.

## Step 2 — Find the BUSID

```powershell
usbipd list
```

## Step 3 — Attach to WSL

```powershell
usbipd attach --wsl --busid 4-2
```

Replace `4-2` with the actual BUSID.

## Step 4 — Verify in WSL

```bash
ls -l /dev/ttyACM*
```

## Step 5 — Open the repository

```bash
cd ~/projects/arduino-dev
code .
```

## Step 6 — Reopen in Dev Container

Use:

```text
Dev Containers: Reopen in Container
```

## Step 7 — Check the board

```bash
arduino-cli board list
```

## Step 8 — Develop

Edit the sketch normally.

## Step 9 — Build

```text
Ctrl+Shift+B
```

or:

```bash
arduino-cli compile --fqbn "$ARDUINO_FQBN" .
```

## Step 10 — Upload

Run the VS Code upload task or:

```bash
arduino-cli upload \
    --port "$ARDUINO_PORT" \
    --fqbn "$ARDUINO_FQBN" \
    .
```

---

# 28. Disconnecting the Arduino

When finished, the board can simply be disconnected physically.

Alternatively, detach it explicitly from Windows:

```powershell
usbipd detach --busid 4-2
```

Once detached, Windows regains normal ownership of the USB device.

---

# 29. USB Attachment Is Not Persistent

Sharing/binding and attachment are different operations.

The initial:

```powershell
usbipd bind --busid 4-2
```

is generally persistent.

The:

```powershell
usbipd attach --wsl --busid 4-2
```

connection is not intended to be permanently maintained.

You should expect to re-run the attach command after events such as:

- unplugging and reconnecting the board,
- restarting Windows,
- some WSL restarts,
- some USB topology changes.

This is normal.

---

# 30. The Serial Port Number May Change

The board may initially appear as:

```text
/dev/ttyACM0
```

and later appear as:

```text
/dev/ttyACM1
```

This can happen after reconnection or when multiple USB serial devices are present.

Check:

```bash
arduino-cli board list
```

or:

```bash
ls -l /dev/ttyACM*
```

If the device changed, update:

```json
"ARDUINO_PORT": "/dev/ttyACM1"
```

and:

```json
"--device=/dev/ttyACM1:/dev/ttyACM1"
```

Then rebuild/recreate the Dev Container.

A future project improvement could automate serial-device discovery instead of assuming `/dev/ttyACM0`.

---

# 31. Optional PowerShell Attach Helper

The repeated attach operation can be made slightly easier.

Create:

```text
scripts/attach-arduino.ps1
```

with:

```powershell
param(
    [Parameter(Mandatory = $true)]
    [string]$BusId
)

Write-Host "Attaching USB device $BusId to WSL..."

usbipd attach --wsl --busid $BusId

if ($LASTEXITCODE -ne 0) {
    Write-Error "Failed to attach USB device $BusId"
    exit $LASTEXITCODE
}

Write-Host ""
Write-Host "Current USB/IP state:"
usbipd list
```

Run:

```powershell
.\scripts\attach-arduino.ps1 -BusId 4-2
```

Keeping the BUSID explicit is safer than automatically attaching the first USB serial device found on the system.

---

# 32. Troubleshooting Strategy

Always troubleshoot from the outside inward:

```text
Windows
   ↓
usbipd
   ↓
WSL2
   ↓
Linux serial device
   ↓
Docker
   ↓
Dev Container
   ↓
Arduino CLI
   ↓
AVRDUDE / upload
```

Do not begin by debugging Arduino CLI if the device is not visible in WSL.

---

# 33. Troubleshooting — Device Not Listed by usbipd

Run:

```powershell
usbipd list
```

If the board does not appear:

1. Try another USB cable.
2. Try another USB port.
3. Check Windows Device Manager.
4. Confirm the board powers on.
5. Avoid charge-only cables.
6. Disconnect and reconnect the board.

Compatible Uno boards may appear by the name of their USB serial chip rather than as "Arduino Uno".

Common USB-to-serial interfaces include:

- CH340/CH341
- CP210x
- FTDI devices

---

# 34. Troubleshooting — usbipd Says Device Is Not Shared

Run from Administrator PowerShell:

```powershell
usbipd bind --busid <BUSID>
```

Then:

```powershell
usbipd list
```

After it shows as shared:

```powershell
usbipd attach --wsl --busid <BUSID>
```

---

# 35. Troubleshooting — Device Attached but Missing in WSL

Check:

```bash
lsusb
```

Then:

```bash
dmesg | tail -n 50
```

Update WSL if necessary:

```powershell
wsl --update
wsl --shutdown
```

Then reopen the WSL distribution and reattach the board.

Microsoft's current WSL USB documentation requires a sufficiently recent WSL kernel for USB/IP support.

---

# 36. Troubleshooting — lsusb Works but No tty Device Exists

Run:

```bash
dmesg | tail -n 50
```

Look for driver messages such as:

```text
cdc_acm ... ttyACM0: USB ACM device
```

Also check:

```bash
ls -l /dev/ttyUSB*
```

A compatible Uno may use a different serial driver from the original Uno R3.

---

# 37. Troubleshooting — Permission Denied

Typical upload error:

```text
avrdude: ser_open(): can't open device "/dev/ttyACM0": Permission denied
```

First check:

```bash
ls -l /dev/ttyACM0
```

Then:

```bash
groups
```

If required:

```bash
sudo usermod -aG dialout "$USER"
```

For a temporary diagnostic test:

```bash
sudo chmod a+rw /dev/ttyACM0
```

If the temporary permission change fixes the problem, the issue is permissions rather than Arduino CLI.

---

# 38. Troubleshooting — Device or Resource Busy

A serial port can only normally be opened by one process at a time.

Check:

```bash
lsof /dev/ttyACM0
```

or:

```bash
fuser /dev/ttyACM0
```

Close:

- Arduino IDE serial monitors,
- other terminal programs,
- old `arduino-cli monitor` sessions,
- scripts holding the serial port.

Then retry the upload.

---

# 39. Troubleshooting — Docker Cannot See the Device

If:

```bash
ls -l /dev/ttyACM0
```

works in WSL but:

```bash
docker run --rm \
    --device=/dev/ttyACM0:/dev/ttyACM0 \
    ubuntu:24.04 \
    ls -l /dev/ttyACM0
```

fails, the problem is between WSL and the Docker daemon.

Do not debug the Dev Container yet.

Determine the Docker architecture using:

```bash
docker context show
docker context ls
docker info
```

If Docker Desktop is being used, review the **Docker Desktop and USB Passthrough** section of this guide.

---

# 40. Troubleshooting — Dev Container Fails to Start

A configuration containing:

```json
"--device=/dev/ttyACM0:/dev/ttyACM0"
```

requires the device to exist when Docker creates the container.

If the Arduino is not attached to WSL when the Dev Container is started, Docker may fail to create the container.

The correct order is:

```text
Connect Arduino
    ↓
Attach Arduino to WSL
    ↓
Verify /dev/ttyACM0
    ↓
Start/Rebuild Dev Container
```

If the container was created before the device became available, use:

```text
Dev Containers: Rebuild Container
```

after attaching the board.

---

# 41. Troubleshooting — Arduino CLI Does Not Detect Uno

Inside the Dev Container:

```bash
arduino-cli board list
```

Then verify that the AVR core is installed:

```bash
arduino-cli core list
```

You should see:

```text
arduino:avr
```

If necessary:

```bash
arduino-cli core update-index
arduino-cli core install arduino:avr@1.8.6
```

The board can still be compiled and uploaded manually using:

```text
arduino:avr:uno
```

even if automatic board identification is imperfect.

---

# 42. Troubleshooting — Upload Fails but Compile Works

Compilation is independent of the physical board.

Therefore:

```text
Compile succeeds
Upload fails
```

usually indicates one of:

- incorrect serial port,
- port permission problem,
- port already open elsewhere,
- board not attached to WSL,
- Docker device not available,
- incorrect board FQBN,
- bootloader/board hardware issue.

Check in this order:

```bash
arduino-cli board list
ls -l "$ARDUINO_PORT"
```

then:

```bash
arduino-cli upload \
    --port "$ARDUINO_PORT" \
    --fqbn "$ARDUINO_FQBN" \
    .
```

---

# 43. Troubleshooting — BRLTTY Conflicts

Some Linux distributions include BRLTTY, which can claim certain generic USB-to-serial converters.

This is more commonly relevant to boards using devices such as:

- CH340
- CP210x
- FTDI serial adapters

Arduino documents methods for resolving BRLTTY conflicts.

Do not remove BRLTTY blindly on a system that actually requires Braille-device support. Arduino's current guidance includes more selective options such as masking the relevant rules or disabling only the conflicting service.

---

# 44. Rebuilding the Dev Container

Rebuild the container after changing:

```text
.devcontainer/Dockerfile
.devcontainer/devcontainer.json
```

Use:

```text
Dev Containers: Rebuild Container
```

A rebuild is also appropriate when changing the device path from:

```text
/dev/ttyACM0
```

to:

```text
/dev/ttyACM1
```

because Docker device mappings are established when the container is created.

---

# 45. Updating Arduino CLI

The guide intentionally pins the Arduino CLI version.

To update:

1. change:

```dockerfile
ARG ARDUINO_CLI_VERSION=1.5.1
```

2. rebuild the Dev Container,
3. compile the project,
4. upload to a test Uno,
5. verify serial operation,
6. commit the version change after successful testing.

This avoids having a new CLI version appear unexpectedly on the next container rebuild.

---

# 46. Updating the Arduino AVR Core

Likewise, the board core is pinned:

```dockerfile
arduino-cli core install arduino:avr@1.8.6
```

To inspect available versions:

```bash
arduino-cli core search arduino:avr
```

After changing the core version, rebuild and run the normal compile/upload test before committing the change.

---

# 47. Adding Arduino Libraries

Arduino CLI can manage project libraries.

Search:

```bash
arduino-cli lib search Servo
```

Install:

```bash
arduino-cli lib install Servo
```

List:

```bash
arduino-cli lib list
```

For a reproducible project, avoid depending on whatever the newest library version happens to be.

Pin versions where practical:

```bash
arduino-cli lib install "LibraryName@x.y.z"
```

For important dependencies, document the selected version in the project or automate installation from the Dockerfile/setup script.

---

# 48. Source Control Recommendations

Commit:

```text
.devcontainer/
.vscode/
docs/
*.ino
project source
project scripts
dependency configuration
```

Do not commit:

- compiler build output,
- temporary files,
- serial logs unless intentionally required,
- secrets,
- machine-specific temporary configuration.

The objective is that another developer can clone the repository, configure host USB access, start the Dev Container, and receive the same Arduino toolchain.

---

# 49. Security Notes

Avoid using:

```text
--privileged
```

unless it is genuinely required.

For an Uno serial device, exposing only:

```text
/dev/ttyACM0
```

is preferable to granting the container unrestricted access to all host devices.

Likewise, avoid exposing all USB devices to the development container when only one serial endpoint is necessary.

This project is a development environment, but applying least-privilege principles still reduces accidental access to unrelated host hardware.

---

# 50. Quick Verification Checklist

Before attempting an upload, verify each layer.

## Windows

```powershell
usbipd list
```

Expected:

```text
Arduino device is present and Attached
```

## WSL

```bash
lsusb
```

Expected:

```text
Arduino or USB serial interface present
```

## Linux Device

```bash
ls -l /dev/ttyACM*
```

Expected:

```text
/dev/ttyACM0
```

or similar.

## Docker

```bash
docker run --rm \
    --device=/dev/ttyACM0:/dev/ttyACM0 \
    ubuntu:24.04 \
    ls -l /dev/ttyACM0
```

Expected:

```text
device visible inside container
```

## Dev Container

```bash
arduino-cli version
arduino-cli core list
arduino-cli board list
```

Expected:

```text
Arduino CLI installed
arduino:avr installed
serial device detected
```

## Build

```bash
arduino-cli compile \
    --fqbn arduino:avr:uno \
    .
```

Expected:

```text
successful compilation
```

## Upload

```bash
arduino-cli upload \
    --port /dev/ttyACM0 \
    --fqbn arduino:avr:uno \
    .
```

Expected:

```text
successful upload
```

---

# 51. Daily Command Summary

### Windows PowerShell

```powershell
usbipd list
usbipd attach --wsl --busid <BUSID>
```

### WSL

```bash
ls -l /dev/ttyACM*
cd ~/projects/arduino-dev
code .
```

### Dev Container

```bash
arduino-cli board list

arduino-cli compile \
    --fqbn "$ARDUINO_FQBN" \
    .

arduino-cli upload \
    --port "$ARDUINO_PORT" \
    --fqbn "$ARDUINO_FQBN" \
    .

arduino-cli monitor \
    --port "$ARDUINO_PORT" \
    --config baudrate=9600
```

---

# 52. Final Recommended Workflow

The target workflow for this project is:

```text
1. Plug Uno into Windows

2. usbipd attach --wsl --busid <BUSID>

3. Confirm /dev/ttyACM0 exists in WSL

4. Open project from WSL:
       code .

5. Reopen project in Dev Container

6. Develop normally

7. Compile using VS Code task

8. Upload using VS Code task

9. Use Arduino CLI serial monitor as required
```

Once the host has been configured, only the USB attach step should normally be Windows-specific during everyday development.

Everything from compilation onward should happen inside the project Dev Container.

---

# References

This guide was prepared using the current official documentation available in September 2026.

- Microsoft — **Connect USB devices to WSL**  
  https://learn.microsoft.com/en-us/windows/wsl/connect-usb

- Docker — **Docker Desktop WSL 2 backend on Windows**  
  https://docs.docker.com/desktop/features/wsl/

- Docker — **Develop with Docker Desktop using WSL 2 on Windows**  
  https://docs.docker.com/desktop/features/wsl/use-wsl/

- Docker — **Docker Desktop General FAQs — USB passthrough**  
  https://docs.docker.com/desktop/troubleshoot-and-support/faqs/general/

- Docker — **Using USB/IP with Docker Desktop**  
  https://docs.docker.com/desktop/features/usbip/

- Visual Studio Code — **Developing inside a Container**  
  https://code.visualstudio.com/docs/devcontainers/containers

- Arduino — **Arduino CLI Getting Started**  
  https://arduino.github.io/arduino-cli/dev/getting-started/

- Arduino — **Arduino CLI Installation**  
  https://docs.arduino.cc/arduino-cli/installation

- Arduino — **Fix port access on Linux**  
  https://support.arduino.cc/hc/en-us/articles/360016495679-Fix-port-access-on-Linux

- Arduino — **Fix udev rules on Linux**  
  https://support.arduino.cc/hc/en-us/articles/9005041052444-Fix-udev-rules-on-Linux

---

## Notes

The USB/container boundary is the most host-dependent part of this project.

If the following command succeeds from WSL:

```bash
docker run --rm \
    --device=/dev/ttyACM0:/dev/ttyACM0 \
    ubuntu:24.04 \
    ls -l /dev/ttyACM0
```

the remainder of the Dev Container setup is comparatively straightforward.

If it fails, resolve the Docker runtime/device architecture first rather than attempting to work around the problem inside Arduino CLI.
