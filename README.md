# alif-san-diego-workshop

# **Setup and Deployment of AI/ML Models onto the Alif Ensemble E8**

This document outlines the step-by-step process a machine learning application onto the Alif Semiconductor's Ensemble E8 Development Kit, as seen in the workshop, *Hands-on Edge AI: Model Deployment and Security with Alif & Thistle*.

These steps are shown in a video tutorial format, available [here](link).

This guide details precisely the steps to recreate the demo featured during the workshop. A more comprehensive walkthrough is listed as part of Alif's official documentation, which can be found [here](https://github.com/alifsemi/alif_ml-embedded-evaluation-kit/blob/main/ML_Embedded_Evaluation_Kit.md).

The example program used in the workshop is a voice-controlled smart camera. This hybrid AI application uses two models running concurrently:
- An image classification model on the Arm® Cortex®-M55 High-Performance (HP) core.
- A keyword spotting (KWS) model on the Cortex-M55 High-Efficiency (HE) core.

The image classification model processes a live video feed from the onboard camera to identify the scene in real-time. The KWS model activates and deactivates the image classifier with keywords "Go" and "Stop".

This document details the following:
- Configuring the host environment
- Compiling both AI models
- Setting up the serial communication with the Alif E8 board
- Provisioning hardware memory correctly
- Verifying deployment

**Host device OS:** Ubuntu 22.04.5 LTS
**Target hardware:** Alif Ensemble E8 Development Kit

---

### Tools & Dependencies

The host machine must first be configured with the appropriate software toolkit and dependencies. This section details the installation of Alif's Security Toolkit (SE Tools), essential system packages, and Arm GNU GCC Compiler required to build the applications.

**1. Download Alif SE Tools (Security Toolkit)**
* Navigate to the Ensemble E8 Dev Kit page on [Alif Semiconductor's website](https://alifsemi.com/support/kits/ensemble-e8devkit/).
* Download the **Linux version** of the Alif Security Toolkit (SE Tools). This workflow uses v1.109.00.
* Extract the downloaded archive to any directory (Home directory recommended).
* The system should now contain a folder named `app-release-exec-linux`.

**2. Install system dependencies**
The following software versions were used during the workshop session:
-   Python 3.10
-   Pip 22.1
-   CMake 3.23
-   Arm GNU GCC Compiler

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install git
sudo apt install libncurses5 libc6-i386 lib32stdc++6
sudo apt install curl dos2unix
sudo apt install python3.10 python3.10-venv libpython3.10 libpython3.10-dev
snap install cmake --classic
```
*Note: While developers can set up a VS Code / CMSIS environment for standard C/C++ development, this guide relies entirely on CLI for these ML examples.*

**3. Install the Arm GNU GCC Compiler (v12.3)**
The steps outlined uses Arm GCC compiler rather than Arm Clang. The repository documentation linked above states how to use the latter.

* Navigate to the [Arm GNU GCC download page](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads) and locate version **12.3.Rel1**.
* Locate the **x86_64 Linux hosted cross toolchains** section.
* Locate **AArch32 bare-metal target (arm-none-eabi)**.
* Download `arm-gnu-toolchain-12.3.rel1-x86_64-arm-none-eabi.tar.xz`.
* Extract the archive to the `usr/local/bin` directory.
* Add to system PATH.
```bash
# Navigate to downloads and extract to usr/local/bin
sudo tar xf arm-gnu-toolchain-12.3.rel1-x86_64-arm-none-eabi.tar.xz -C /usr/local/bin

# Add to system PATH
sudo sh -c 'echo export PATH=/usr/local/bin/arm-gnu-toolchain-12.3.rel1-x86_64-arm-none-eabi/bin:\$PATH >/etc/profile.d/arm-compiler.sh'
```
Verify the installation by running the command `arm-none-eabi-gcc --version`. If it fails, re-log into the Ubuntu session.

---

### Setting up the ML Embedded Evaluation Kit Environment

Once the tools required are installed, the next step involves acquiring the necessary machine learning source code. This section guides the user through cloning Alif's Machine Learning Embedded Evaluation Kit repository and fetching the pre-trained models.

**1. Clone the repository & initialize submodules**
```bash
# In any directory
git clone https://github.com/alifsemi/alif_ml-embedded-evaluation-kit.git
cd alif_ml-embedded-evaluation-kit

# Initialize and update required submodules
git submodule update --init --recursive
```

**2. Download AI resources**
Run the automated script to fetch the models and weights required for the example.
```bash
# This uses Tensorflow as the ML framework
python3 set_up_default_resources.py
```

---

### **Compiling the AI Models**

In this section, the clones ML repositories will be built into executable binaries, targeting the M55 HE core for keyword spotting and the M55 HP core for image classification.

**1. Compile the Keyword Spotting Model (M55-HE)**
```bash
# Create and enter the build directory for the HE core
mkdir build_alif_kws
cd build_alif_kws

# Configure the build with CMake
cmake -DTARGET_PLATFORM=alif \
-DTARGET_SUBSYSTEM=RTSS-HE \
-DTARGET_BOARD=DevKit-e8 \
-DCMAKE_TOOLCHAIN_FILE=scripts/cmake/toolchains/bare-metal-gcc.cmake \
-DGLCD_UI=OFF \
-DLINKER_SCRIPT_NAME=RTSS-HE-TCM \
-DCMAKE_BUILD_TYPE=Release \
-DMLEK_LOG_LEVEL=MLEK_LOG_LEVEL_DEBUG \
-DUSE_CASE_BUILD=alif_kws ..

# Build the binary
make ethos-u-alif_kws -j4
```

**2. Compile the Image Classification Model (M55-HP, using Ethos-U85)**
```bash
# Create and enter the build directory for the HP core
mkdir build_alif_img_class
cd build_alif_img_class

# Configure the build
cmake -DTARGET_PLATFORM=alif \
-DTARGET_SUBSYSTEM=RTSS-HP \
-DTARGET_BOARD=DevKit-e8 \
-DCMAKE_TOOLCHAIN_FILE=scripts/cmake/toolchains/bare-metal-gcc.cmake \
-DCONSOLE_UART=4 \
-DCMAKE_BUILD_TYPE=Release \
-DMLEK_LOG_LEVEL=MLEK_LOG_LEVEL_DEBUG \
-DUSE_CASE_BUILD=alif_img_class
-DETHOS_U_NPU_ID=U85 ..


# Build the binary
make ethos-u-alif_img_class -j4
```

---

### **Extracting Binaries**

Following compilation, the generated binaries must be prepared for flashing to the E8 Ensemble board. This section details how to locate and transfer the binary files into SE Tools for deployment.

**1. KWS binary**
```bash
# Locate the KWS MRAM binary
cd ~/ml-embedded-evaluation-kit/build_alif_kws/bin/sectors/alif_kws/

# Rename the binary
mv mram.bin ethos-u-alif-kws.bin

# Move it to the SE Tools images folder
mv ethos-u-alif-kws.bin ~/app-release-exec-linux/build/images/
```

**2. Image classification binary**
```bash
# Locate the Image Classification MRAM binary
cd ~/ml-embedded-evaluation-kit/build_alif_img_class/bin/sectors/alif_img_class/

# Rename the binary
mv mram.bin ethos-u-alif-img-class.bin

# Move it to the SE Tools images folder
mv ethos-u-alif-image-class.bin ~/app-release-exec-linux/build/images/
```

---

### **Creating the JSON Configuration (ATOC) file**

For the hardware to execute the deployed models correctly, it requires precise memory mapping instructions. This section covers the creation of an Application Table of Contents (ATOC) configuration file, which dictates how the binaries are loaded the cores within the E8 Fusion processor.

```bash
# Navigate to the config folder inside SE tools
cd ~/app-release-exec-linux/build/config/

# Create a new configuration file combining both models
nano kws_IC_demo.json
```
Populate the .json file with the following content:
```json
{
	"IC": {
		"binary": "ethos-u-alif_img_class.bin",
		"version": "1.0.0",
		"mramAddress": "0x80008000",
		"cpu_id": "M55_HP",
		"flags": ["boot"],
		"signed": false
	},
	"KWS": {
		"binary": "ethos-u-alif_kws.bin",
		"version": "1.0.0",
		"mramAddress": "0x80480000",
		"cpu_id": "M55_HE",
		"flags": ["boot"],
		"signed": false
	},
	"DEVICE": {
		"disabled" : false,
		"binary": "app-device-config.json",
		"version" : "0.5.00",
		"signed": true
	}
}
```

---

### **Hardware Setup & Flashing**

This section outlines the correct physical configuration of the E8 Development Kit, followed by process to flash the firmware and custom AI models directly into the device's MRAM.

**1. Prepare the E8 board**
Before connecting the USB cable, the engineer must verify the hardware switches on the back of the E8 board:
*   **USB Port:** Connect the USB-C cable to the **PRG** port (the top port).
*   **SW3 Switch:** Ensure this is flipped to **EN**.
*   **SW4 Switch:** Ensure this is flipped fully to the **SE**.
*   Plug the USB-C cable into the host machine.

**2. Update the on-board firmware**
```bash
cd ~/app-release-exec-linux

./update_system_package
```
*Note: On first run, the tool will ask which serial port the board is connected to. It will list available ports and appropriate one must be selected (for example: `/dev/ttyACM0`)*

**3. Generate ATOC & flash to MRAM**
```bash
# Generate the ATOC file using the JSON file created earlier
./app_gen_toc -f build/config/kws_IC_demo.json

# Write to MRAM
./app_write_mram -p
```

---

### **Hardware Verification**

This section overviews how to verify that both the keyword spotting and image classification models are operating correctly.

**1. Attach the external display**
*   Disconnect the E8 board from power.
*   Locate the two ribbon connectors on the bottom right of the board.
*   Plug the external display into the connectors.
*   Reconnect the board to power.

**2. Run the demo**
*   If the program does not boot, press the **RESET** button on the board.
*   Observe the external display. The user interface should be loaded.
*   Say the word **"GO"**. The image classification model will activate and begin identifying the scene in view.
*   Say the word **"STOP"**. The image classification will pause until re-activated.

Congratulations, you have successfully compiled, flashed, and verified a dual-core ML inference application on the Alif Ensemble E8 Development Kit.

For more information and a detailed guide in flashing your own models to the board, please refer to [Alif's official documentation](https://github.com/alifsemi/alif_ml-embedded-evaluation-kit/blob/main/ML_Embedded_Evaluation_Kit.md).
