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
ARG MY_OPENFOAM_VERSION="v2406"
ENV FOAM_INSTALL=/home/$USERNAME/OpenFOAM
RUN mkdir -p $FOAM_INSTALL && \
    # Clone desired version (Example: OpenFOAM.org development branch)
    wget no-check-certificate https://cyfuture.dl.sourceforge.net/project/openfoam/${MY_OPENFOAM_VERSION}/OpenFOAM-${MY_OPENFOAM_VERSION}.tgz && \
    wget no-check-certificate https://cyfuture.dl.sourceforge.net/project/openfoam/${MY_OPENFOAM_VERSION}/ThirdParty-${MY_OPENFOAM_VERSION}.tgz
    # Add lines here to checkout specific tags/branches if needed

WORKDIR $FOAM_INSTALL/OpenFOAM-${MY_OPENFOAM_VERSION}

# === Phase 3: Configure Build Environment (Adapted from) ===
# Create prefs.sh to specify Intel Compilers
# ** CRITICAL: Verify WM_COMPILER value (e.g., intelicx) for your OpenFOAM version/Intel compilers **
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