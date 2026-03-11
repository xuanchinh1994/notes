# notes

## Token


Here is the English translation of your Technical Proposal. I kept the tone professional, objective, and technical so it is perfectly suited for presentation to your development team, Project Manager, or System Architect.
TECHNICAL PROPOSAL: OPTIMIZING THE USB RESUME PROCESS ON LINUX KERNEL 6.12
1. Executive Summary
When upgrading the system to Linux Kernel 6.12, maintaining custom patches that deeply intervene in the USB Core (drivers/usb/core/) and Power Management Core (drivers/base/power/main.c) to achieve a "parallel resume/snapshot" is no longer feasible. It causes source code conflicts, increases technical debt, and raises the risk of Kernel Panics.
This proposal presents an alternative solution: Leveraging the Kernel's native Asynchronous Suspend/Resume mechanism via Udev Rule configurations in User-space. This solution ensures that the wake-up time of USB devices is optimized in parallel without requiring any modifications to the USB core/host source code.
2. Background & Problem
 * Legacy Solution: Using global variables and modifying the logic within generic.c or hub.c to force USB devices to bypass sequential execution and resume simultaneously. Occasionally, a priority thread (instant_ctrl) was used to wake up essential devices first.
 * Issues on Kernel 6.12:
   * The Kernel's Power Management (PM) architecture has undergone significant changes. Maintaining out-of-tree patches incurs high maintenance costs.
   * Direct conflicts with security and performance updates from Upstream.
 * Objective: Achieve parallel resume speeds (approaching the legacy solution) while strictly adhering to the "Zero core-modifications" philosophy.
3. Proposed Solution
Utilize the Native Asynchronous PM built into the Linux kernel. Instead of hardcoding logic in the Kernel, we will shift configuration decisions to User-space via the udev device management system.
Operating Principle:
 * When a USB device is recognized, the Linux Kernel assigns it a power management attribute in sysfs (specifically /sys/bus/usb/devices/.../power/async).
 * By default, this feature might not be enabled for certain devices or buses to ensure backward compatibility.
 * We will write Udev Rules to automatically "listen" for hardware events. As soon as the system boots or a device is plugged in, Udev will automatically write the enabled flag to the power/async attribute.
 * When the system receives a Wake-up command from the Suspend state, the Kernel PM Core will read this flag and automatically allocate Worker Threads to invoke the resume functions of the USB devices simultaneously (in parallel), rather than waiting for them to execute one by one.
4. Implementation Plan
Phase 4.1: Target Hardware Identification
System engineers will compile a list of USB devices that require prioritized resume time optimization on the board (e.g., Wi-Fi/BT Module, USB Touchscreen, 4G/5G Modem). Extract the Vendor ID (VID) and Product ID (PID) using the lsusb command.
Phase 4.2: Udev Rule Integration into Rootfs
Create a new rule file, for example, 99-usb-async-resume.rules, and place it in the /etc/udev/rules.d/ directory within the firmware's root file system (Rootfs).
Reference Content (Customizable per project):
# [PROJECT ABC] - Auto-enable Asynchronous Resume for critical peripherals

# 1. Enable Async PM for Wifi/BT Module (e.g., Realtek - VID:0bda PID:c820)
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="0bda", ATTR{idProduct}=="c820", ATTR{power/async}="enabled"

# 2. Enable Async PM for Touchscreen Module (e.g., Goodix - VID:27c6 PID:0118)
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="27c6", ATTR{idProduct}=="0118", ATTR{power/async}="enabled"

# 3. (Optional) Enable for all USB devices if hardware permits
# ACTION=="add", SUBSYSTEM=="usb", ATTR{power/async}="enabled"

Phase 4.3: Integration into Yocto/Buildroot (If applicable)
If the project uses the Yocto Project or Buildroot to build the OS:
 * Create a custom recipe (e.g., usb-async-rules.bb) to copy this .rules file into /etc/udev/rules.d/ during the image build process.
5. Pros & Risks
Pros:
 * Zero Technical Debt: A complete transition to standard Linux mechanisms (Upstream-friendly). There is no need to maintain kernel patches when upgrading to newer Kernel versions (6.15, 6.20...).
 * High Stability: Entirely avoids the risk of Deadlocks or Kernel Panics caused by incorrect PM thread interventions.
 * Stateful: Applications holding open device file descriptors (e.g., /dev/ttyUSB0) will not lose their connections (no I/O errors) during sleep/wake cycles.
 * Easy Configuration: Adding or removing devices for optimization only requires editing a text file in user-space; no kernel recompilation is needed.
Risks & Mitigations:
 * Risk: Cannot assign a "Strict Priority" to a specific device like the legacy solution (instant_resume). Because the kernel self-allocates threads, devices A and B will wake up in parallel, but there is no guarantee that device A will finish before device B.
 * Mitigation: The application layer in user-space must be redesigned to check device states more flexibly, rather than relying on a hardcoded hardware initialization sequence.
6. Test Plan
To validate the effectiveness of the solution, the QA/Dev team will execute the following tests:
 * Configuration State Check:
   * After booting, read the /sys/bus/usb/devices/<bus-id>/power/async file to verify that the system returns the value enabled.
 * Performance Profiling:
   * Use the analyze_suspend.py tool (located in the tools/power/pm-graph/ directory of the kernel tree) to plot the Suspend/Resume timeline.
   * Success Criterion: The graph displays the usb_resume functions of the devices overlapping on the timeline, instead of running sequentially.
 * Functional Test:
   * Perform continuous Suspend/Resume cycles 1,000 times (Stress test). Ensure no device drops or Kernel Oops occur.
7. Conclusion
Transitioning from a "Custom Kernel Patch" to "Native Asynchronous PM via Udev" is the most sustainable, safe, and robust development path for Kernel 6.12. This solution perfectly satisfies the requirement of zero interference with the USB core/host source code, while still delivering outstanding parallel resume performance. We recommend approval to proceed with a Proof of Concept (PoC) directly on the target board.
Would you like me to help you draft an introductory email to go along with this proposal when you send it to your team?
