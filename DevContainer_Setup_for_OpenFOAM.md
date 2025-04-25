---
title: "Dev Container Setup for OpenFOAM (Source Build with Intel Compilers)"
author: "OpenFOAM Development Team"
date: "2025-04-25"
abstract: |
    This document provides a comprehensive guide to setting up a development container for OpenFOAM using Intel compilers. It includes detailed instructions for creating a Docker-based environment with a `Dockerfile` and `devcontainer.json`, ensuring a repeatable and self-contained setup. The guide covers prerequisites, environment configuration, source compilation, and troubleshooting tips to streamline the OpenFOAM development workflow.
keywords: ["OpenFOAM", "Dev Containers", "Intel Compilers", "Docker", "Source Build", "Development Environment", "HPC", "Computational Fluid Dynamics"]
---

### Dev Container Setup for OpenFOAM (Source Build with Intel Compilers)

This setup aims to create a self-contained environment using a `Dockerfile` and `devcontainer.json`.

**1. Prerequisites:**

* Visual Studio Code installed.
* Docker Desktop (Windows/macOS) or Docker Engine (Linux) installed and **running**.
* `Dev Containers` extension installed in VS Code.

**2. Project Setup:**

* Create a new folder for your project (or use an existing one).
* Open this folder in VS Code.
* Create a subfolder named `.devcontainer` inside your project folder.

**3. Create `Dockerfile`:**

* Inside the `.devcontainer` folder, create a file named `Dockerfile`.
* Populate it with the following content, adapting paths and versions as needed:

```dockerfile
# Start from the Ubuntu base specified in the document
ARG UBUNTU_VERSION=22.04
FROM ubuntu:${UBUNTU_VERSION}

# Avoid interactive prompts during package installation
ENV DEBIAN_FRONTEND=noninteractive

# === Phase 1: Prerequisites (Adapted from) ===
# Update and install essential build tools and dependencies
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
    build-essential \
    sudo \
    wget \
    gnupg \
    lsb-release \
    ca-certificates \
    autoconf \
    automake \
    cmake \
    git-core \
    flex \
    bison \
    zlib1g-dev \
    libfl-dev \
    libboost-system-dev \
    libboost-thread-dev \
    # libopenmpi-dev openmpi-bin # Install system OpenMPI only if needed by other tools, Intel MPI will be primary
    gawk \
    texinfo \
    libreadline-dev \
    libgmp-dev \
    libmpfr-dev \
    libmpc-dev \
    libscotch-dev \
    libptscotch-dev \
    libmetis-dev \
    # libparmetis-dev # Often depends on system MPI, might conflict if using Intel MPI solely
    libfftw3-dev \
    libqt5x11extras5-dev \
    # Add any other specific dependencies identified for your OpenFOAM version
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# === Phase 1b: Install Intel oneAPI Base & HPC Toolkits (Adapted from) ===
# ** CRITICAL STEP - Automation highly dependent on Intel's installer **
# This section requires significant adaptation based on how you obtain/run the installers.
# Example using Intel's APT repository setup (verify current instructions from Intel):
RUN wget -O- https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB \
    | gpg --dearmor | sudo tee /usr/share/keyrings/oneapi-archive-keyring.gpg > /dev/null && \
    echo "deb [signed-by=/usr/share/keyrings/oneapi-archive-keyring.gpg] https://apt.repos.intel.com/oneapi all main" | sudo tee /etc/apt/sources.list.d/oneAPI.list && \
    apt-get update && \
    # Adjust package names based on Intel's current offering for Base + HPC toolkit components
    # This might require accepting license terms interactively or via config files/flags if possible
    apt-get install -y intel-oneapi-compiler-dpcpp-cpp intel-oneapi-compiler-fortran intel-oneapi-mpi-devel # Add other components as needed
    # Clean up
    # && apt-get clean && rm -rf /var/lib/apt/lists/*

# Default install location for oneAPI via apt is often /opt/intel/oneapi
ENV INTEL_ONEAPI_ROOT=/opt/intel/oneapi

# === Phase 2: Download OpenFOAM Source (Adapted from) ===
# Create user and directories
ARG USERNAME=vscode
ARG USER_UID=1000
ARG USER_GID=$USER_UID
RUN groupadd --gid $USER_GID $USERNAME && \
    useradd --uid $USER_UID --gid $USER_GID -m $USERNAME && \
    echo $USERNAME ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME && \
    chmod 0440 /etc/sudoers.d/$USERNAME

USER $USERNAME
WORKDIR /home/$USERNAME

# Set up OpenFOAM directory
ENV FOAM_INSTALL=/home/$USERNAME/OpenFOAM
RUN mkdir -p $FOAM_INSTALL && \
    # Clone desired version (Example: OpenFOAM.org development branch)
    git clone --depth 1 https://develop.openfoam.com/Development/openfoam.git $FOAM_INSTALL/OpenFOAM-dev && \
    git clone --depth 1 https://develop.openfoam.com/Development/ThirdParty-common.git $FOAM_INSTALL/ThirdParty-dev
    # Add lines here to checkout specific tags/branches if needed [cite: 23, 24]

WORKDIR $FOAM_INSTALL/OpenFOAM-dev

# === Phase 3: Configure Build Environment (Adapted from) ===
# Create prefs.sh to specify Intel Compilers
# ** CRITICAL: Verify WM_COMPILER value (e.g., intelicx) for your OpenFOAM version/Intel compilers [cite: 34, 35] **
RUN echo '#!/bin/bash' > $FOAM_INSTALL/OpenFOAM-dev/prefs.sh && \
    echo '# Compiler Selection (Verify value!)' >> $FOAM_INSTALL/OpenFOAM-dev/prefs.sh && \
    echo 'export WM_COMPILER=intelicx' >> $FOAM_INSTALL/OpenFOAM-dev/prefs.sh && \
    echo '# MPI Selection (Use Intel MPI)' >> $FOAM_INSTALL/OpenFOAM-dev/prefs.sh && \
    echo 'export WM_MPLIB=INTELMPI' >> $FOAM_INSTALL/OpenFOAM-dev/prefs.sh && \
    chmod +x $FOAM_INSTALL/OpenFOAM-dev/prefs.sh

# === Phase 4: Compile OpenFOAM (Adapted from) ===
# ** This will take a very long time and increase image size significantly **
# Source environments and compile ThirdParty
RUN . $INTEL_ONEAPI_ROOT/setvars.sh && \
    . $FOAM_INSTALL/OpenFOAM-dev/etc/bashrc && \
    cd $FOAM_INSTALL/ThirdParty-dev && \
    # Set FOAM_INST_DIR if needed by ThirdParty scripts
    export FOAM_INST_DIR=$FOAM_INSTALL && \
    # Clean and compile ThirdParty (adjust -j $(nproc) as needed)
    # ./Allwclean && \ # Optional clean
    ./Allwmake -j $(nproc)

# Source environments and compile OpenFOAM
RUN . $INTEL_ONEAPI_ROOT/setvars.sh && \
    . $FOAM_INSTALL/OpenFOAM-dev/etc/bashrc && \
    cd $FOAM_INSTALL/OpenFOAM-dev && \
    # Clean and compile OpenFOAM (adjust -j $(nproc) as needed)
    # ./Allwclean && \ # Optional clean
    ./Allwmake -j $(nproc)

# === Phase 5: Post-Install / Environment Setup (Adapted from) ===
# Set up environment sourcing in .bashrc for interactive terminals
RUN echo '\n# Load Intel oneAPI and OpenFOAM environment' >> /home/$USERNAME/.bashrc && \
    echo "if [ -f $INTEL_ONEAPI_ROOT/setvars.sh ]; then . $INTEL_ONEAPI_ROOT/setvars.sh > /dev/null; fi" >> /home/$USERNAME/.bashrc && \
    echo "if [ -f $FOAM_INSTALL/OpenFOAM-dev/etc/bashrc ]; then . $FOAM_INSTALL/OpenFOAM-dev/etc/bashrc > /dev/null; fi" >> /home/$USERNAME/.bashrc

# Set WORKDIR for new terminals
WORKDIR /home/$USERNAME/OpenFOAM/OpenFOAM-dev/run
RUN mkdir -p /home/$USERNAME/OpenFOAM/OpenFOAM-dev/run

# Default command (optional)
# CMD ["/bin/bash"]

# Switch back to root for potential VS Code setup steps if needed later
USER root
```

**4. Create `devcontainer.json`:**

* Inside the `.devcontainer` folder, create a file named `devcontainer.json`.
* Populate it with the following content:

```json
{
    "name": "OpenFOAM (Source Build - Intel Compilers)",
    "build": {
        "dockerfile": "Dockerfile",
        "args": {
            // Optionally change Ubuntu version if needed in Dockerfile ARG
            // "UBUNTU_VERSION": "22.04"
        }
    },

    // Set container user for VS Code terminal
    "remoteUser": "vscode",

    // Set workspace folder inside the container
    "workspaceFolder": "/home/vscode/OpenFOAM/OpenFOAM-dev/run", // Or your preferred workspace root

    // Add required VS Code extensions
    "customizations": {
        "vscode": {
            "extensions": [
                "ms-vscode.cpptools", // C/C++ IntelliSense, debugging
                "ms-vscode.cmake-tools", // If using CMake for user projects
                "jeff-hykin.better-cpp-syntax" // Improved C++ syntax highlighting
                // Add Fortran extension if needed: e.g., "fortran-lang.linter-gfortran" (adapt if using ifx primarily)
            ]
        }
    },

    // Optional: Forward ports if needed (e.g., for ParaView server)
    // "forwardPorts": [],

    // Optional: Run commands after container creation (alternative to Dockerfile RUN for some steps)
    // "postCreateCommand": "echo 'Container created!'",

    // Optional: Mount local folders if needed (e.g., for input files, license files)
    // "mounts": [
    //  "source=/path/on/host,target=/path/in/container,type=bind,consistency=cached"
    // ],

    // Increase resources if needed, compilation is resource intensive [cite: 58]
    "runArgs": [
        "--shm-size=2g" // Example: Increase shared memory
        // Add other Docker run arguments as necessary
    ]

}
```

**5. Build and Launch the Dev Container:**

* Ensure Docker Desktop/Engine is running.
* Open the command palette in VS Code (`Ctrl+Shift+P` or `Cmd+Shift+P`).
* Run the command: `Dev Containers: Reopen in Container`.
* VS Code will build the Docker image based on your `Dockerfile`. This will take a **very long time** (potentially hours) due to the compilation steps[cite: 45]. Monitor the build log shown in the VS Code terminal.
* Once the build is successful, VS Code will connect to the container.

**6. Verify the Environment:**

* Open a new terminal inside VS Code (`Terminal` > `New Terminal`).
* The terminal should automatically source the Intel and OpenFOAM environments via the `.bashrc` setup[cite: 47, 49].
* Verify the environment[cite: 38, 50]:
    ```bash
    echo "Compiler: $WM_COMPILER" # Should show 'intelicx' or your setting [cite: 34]
    echo "MPI Lib:  $WM_MPLIB"   # Should show 'INTELMPI' [cite: 36]
    which icpx                  # Should point to Intel compiler
    which mpirun                # Should point to Intel MPI
    which blockMesh simpleFoam  # Should find OpenFOAM utilities
    foamInstallationTest        # Run built-in test
    ```
* Try running a tutorial case[cite: 50, 51]:
    ```bash
    cd $FOAM_RUN
    cp -r $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity .
    cd cavity
    blockMesh
    icoFoam
    # Check logs for errors
    ```

**7. Development Workflow:**

* You can now edit code within VS Code, and it will operate directly on the files inside the container.
* Use the integrated terminal for all compilation, execution, and Git commands related to your OpenFOAM work.
* If you modify the `devcontainer.json` or `Dockerfile`, rebuild the container using `Dev Containers: Rebuild Container` from the command palette.

**Troubleshooting Notes (Adapted from):**

* **Build Failures:** Carefully examine the build logs in the VS Code terminal for errors during `apt-get`, Intel oneAPI setup, or `Allwmake`[cite: 51]. Missing dependencies [cite: 55] or incorrect compiler/MPI settings [cite: 53, 54] are common causes.
* **Intel oneAPI Installer:** The automated installation [cite: 16] is the most likely point of failure. Check Intel's documentation for non-interactive installation options or consider alternative installation methods.
* **Environment Sourcing:** Ensure the `setvars.sh` and OpenFOAM `bashrc` are sourced correctly and in the right order [cite: 47, 57] within the `Dockerfile` RUN commands and the final `.bashrc`.
* **Resource Limits:** OpenFOAM compilation requires significant RAM and CPU[cite: 58]. Ensure Docker has sufficient resources allocated. Adjust `runArgs` in `devcontainer.json` if needed.
* **Clean Build:** If you encounter persistent errors after changes, use `Dev Containers: Rebuild Container without Cache` or add `./Allwclean` steps [cite: 57] back into the `Dockerfile` before the `Allwmake` commands (commented out in the example above).

This comprehensive setup provides a repeatable OpenFOAM development environment based on your source build requirements using Intel compilers, leveraging the power of Dev Containers. Remember to adapt the specifics (versions, compiler flags, dependencies) according to the official documentation for the OpenFOAM version you choose[cite: 2, 9, 55].