# WaveSpeed Desktop Linux Fixes and CUDA Support

This document provides detailed information for the original WaveSpeed Desktop developer
to fix known Linux issues and add CUDA support.

## Issues Found on Ubuntu 24.04

### Issue 1: Electron Sandbox Crash

Symptom:
  App crashes immediately with:
  "LaunchProcess: failed to execvp: /opt/WaveSpeed"
  "FATAL:zygote_host_impl_linux.cc(207)] Check failed: . : Invalid argument (22)"
  "Trace/breakpoint trap (core dumped)"

Cause:
  Electron sandbox requires either SUID chrome-sandbox or --no-sandbox flag.
  The SUID approach fails on Ubuntu 24.04.

Solution:
  Add --no-sandbox flag for both CLI and GUI launches.

Files to modify:

  1. build/linux/after-install.sh
     Replace the symlink creation with a wrapper script:

     #!/bin/bash

     # Create wrapper script with --no-sandbox for Ubuntu 24.04+ compatibility
     # The Electron sandbox requires either SUID chrome-sandbox or --no-sandbox flag
     cat > '/usr/bin/${executable}' << 'EOF'
     #!/bin/bash
     exec '/opt/${sanitizedProductName}/${executable}' --no-sandbox "$@"
     EOF
     chmod +x '/usr/bin/${executable}'

     if hash update-mime-database 2>/dev/null; then
         update-mime-database /usr/share/mime || true
     fi

     if hash update-desktop-database 2>/dev/null; then
         update-desktop-database /usr/share/applications || true
     fi

  2. package.json (in "build.linux" section)
     Add desktop.exec with --no-sandbox:

     "linux": {
       "target": [
         {
           "target": "deb",
           "arch": ["x64"]
         }
       ],
       "category": "Development",
       "maintainer": "WaveSpeed <support@wavespeed.ai>",
       "desktop": {
         "exec": "wavespeed-desktop --no-sandbox %U"
       }
     }

### Issue 2: libstable-diffusion.so Not Found

Symptom:
  SD binary fails with:
  "error while loading shared libraries: libstable-diffusion.so: cannot open shared object file"

Cause:
  Linux doesn't search the binary's directory for shared libraries by default.

Solution:
  Set LD_LIBRARY_PATH before spawning the SD process.

File to modify:

  electron/lib/sdGenerator.ts

  Find the spawn call (around line 158) and add LD_LIBRARY_PATH:

    import { dirname } from 'path'  // add to imports if not present

    // Before the spawn call, add:
    const binaryDir = dirname(binaryPath)
    const ldLibraryPath = process.env.LD_LIBRARY_PATH
      ? `${binaryDir}:${process.env.LD_LIBRARY_PATH}`
      : binaryDir

    // Modify the spawn call:
    const childProcess = spawn(binaryPath, args, {
      cwd: outputDir,
      env: { ...process.env, LD_LIBRARY_PATH: ldLibraryPath }
    })

### Issue 3: AVX512 Illegal Instruction

Symptom:
  SD binary crashes with "Illegal instruction (core dumped)"

Cause:
  The pre-built Linux binary (sd-master-700a797-bin-Linux-Ubuntu-24.04-x86_64-avx512.zip)
  uses AVX512 CPU instructions that are not supported by all CPUs.

Solution:
  Provide CUDA builds for Linux (and/or builds with lower CPU requirements).

## Adding CUDA Support

### Step 1: Build CUDA Binaries

Requirements:
  - Ubuntu 24.04
  - NVIDIA GPU
  - NVIDIA driver 580+ with CUDA 12.0+

Build commands:

  sudo apt install nvidia-cuda-toolkit
  git clone --recursive https://github.com/leejet/stable-diffusion.cpp /tmp/sd-cpp
  cd /tmp/sd-cpp && mkdir build && cd build
  cmake .. -DSD_CUDA=ON
  cmake --build . --config Release -j$(nproc)

Output binaries:
  /tmp/sd-cpp/build/bin/sd-cli
  /tmp/sd-cpp/build/bin/sd-server

### Step 2: Add CUDA Build to GitHub Releases

Create a new release asset:
  sd-master-XXXXX-bin-Linux-Ubuntu-24.04-x86_64-cuda12.zip

The zip should contain:
  - sd-cli (or sd)
  - sd-server (optional)

### Step 3: Update useZImage.ts to Download CUDA Binary

File: src/hooks/useZImage.ts

Find the Linux binary URL selection (around line 241-244) and update:

  Current code:
    } else {
      // Linux: Use AVX512 build from WaveSpeed release
      url = `${githubBaseUrl}/sd-master-700a797-bin-Linux-Ubuntu-24.04-x86_64-avx512.zip`
    }

  Updated code:
    } else {
      // Linux: Use CUDA build if available, otherwise AVX512
      if (acceleration === 'CUDA') {
        console.log('[useZImage] Using Linux CUDA build from WaveSpeed release')
        url = `${githubBaseUrl}/sd-master-XXXXX-bin-Linux-Ubuntu-24.04-x86_64-cuda12.zip`
      } else {
        console.log('[useZImage] Using Linux AVX512 build from WaveSpeed release')
        url = `${githubBaseUrl}/sd-master-700a797-bin-Linux-Ubuntu-24.04-x86_64-avx512.zip`
      }
    }

### Step 4: Update Locale Message (Optional)

Since CUDA is now supported, update the message.

Files: src/i18n/locales/*.json (all 18 files)

Change key name and message:
  From: "linuxCpuOnly": "Local Z-Image runs on CPU only on Linux..."
  To:   "linuxCudaSupported": "Z-Image uses GPU acceleration when available (CUDA on Linux, Metal on macOS, Vulkan on Windows)."

File: src/pages/ZImagePage.tsx

Update the translation key reference:
  From: {t('zImage.tips.linuxCpuOnly')}
  To:   {t('zImage.tips.linuxCudaSupported')}

## GitHub Actions Workflow for CUDA Builds

Add a new job to .github/workflows/build.yml for building CUDA binaries:

  build-sd-cuda:
    runs-on: ubuntu-24.04
    steps:
      - name: Install CUDA Toolkit
        run: sudo apt-get update && sudo apt-get install -y nvidia-cuda-toolkit

      - name: Clone stable-diffusion.cpp
        run: git clone --recursive https://github.com/leejet/stable-diffusion.cpp

      - name: Build with CUDA
        run: |
          cd stable-diffusion.cpp
          mkdir build && cd build
          cmake .. -DSD_CUDA=ON
          cmake --build . --config Release -j$(nproc)

      - name: Package
        run: |
          cd stable-diffusion.cpp/build/bin
          zip -r sd-cuda-linux-x64.zip sd-cli sd-server

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: sd-cuda-linux-x64
          path: stable-diffusion.cpp/build/bin/sd-cuda-linux-x64.zip

Note: GitHub-hosted runners don't have NVIDIA GPUs, so the build will work but
      you cannot test it on the runner. Consider using a self-hosted runner with
      NVIDIA GPU for building CUDA binaries.

## Summary of All Changes

Files modified:
  - build/linux/after-install.sh (wrapper script with --no-sandbox)
  - package.json (desktop.exec with --no-sandbox, optional: remove AppImage)
  - electron/lib/sdGenerator.ts (LD_LIBRARY_PATH for shared libraries)
  - src/hooks/useZImage.ts (CUDA binary URL selection)
  - src/pages/ZImagePage.tsx (locale key rename)
  - src/i18n/locales/*.json (18 files - locale key rename and message update)

New assets to release:
  - sd-master-XXXXX-bin-Linux-Ubuntu-24.04-x86_64-cuda12.zip

## Testing

After applying these changes:

1. Build the deb package: npm run build:linux
2. Install: sudo dpkg -i dist/WaveSpeed-Desktop-linux-amd64.deb
3. Run: wavespeed-desktop
4. Navigate to Z-Image and test image generation

Expected behavior:
  - App launches without sandbox crash
  - Z-Image downloads CUDA binary (if NVIDIA GPU detected)
  - Image generation uses GPU acceleration

## Contact

These fixes were developed and tested by tchung1970 on Ubuntu 24.04 with
NVIDIA RTX 4000 Ada Generation GPU.

Repository with pre-built CUDA binaries:
https://github.com/tchung1970/wavespeed-desktop-cuda
