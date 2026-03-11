.
Here is the complete translation for the refined Option 3 Technical Proposal, including the translated PlantUML flowchart exactly as requested.
TECHNICAL PROPOSAL: DEVELOPING AN INDEPENDENT KERNEL MODULE (HARDCODED WIFI/BT) FOR USB RESUME MANAGEMENT ON LINUX 6.12
1. Executive Summary
When upgrading the system to Linux Kernel 6.12, patches that directly intervene in the kernel source code (especially at drivers/usb/core/ and drivers/base/power/main.c) to serve the parallel resume/snapshot feature are no longer suitable, as they easily cause conflicts and critical errors (Kernel Panic).
This proposal presents an alternative solution: Building an independent Loadable Kernel Module (LKM) located outside the kernel tree (Out-of-tree). Specifically, this module will hardcode the Vendor ID/Product ID list of critical devices (namely, the Wifi/Bluetooth module cluster). The LKM will operate as a background process, intercepting power management events to trigger an Instant Resume for the Wifi/BT as soon as the system wakes up, completely bypassing the standard Kernel's sequential waiting mechanism.
2. Background & Problem
 * Issues with the legacy patch: Deep intervention into the Kernel's Device Power Management (DPM) flow creates massive technical debt and blocks the ability to upgrade the kernel (Upstream).
 * Limitations of the Native solution (Option 1): The Kernel's native Asynchronous PM mechanism only helps devices wake up in parallel, but does not allow specifying absolute priority. The Kernel dictates which thread completes first.
 * System objective: The network connection (Wifi/BT) is a "survival" component of the board. It must be the absolute first device ready after the system exits sleep mode, with the lowest possible latency.
3. Proposed Solution
Design a dedicated .ko Kernel Module (e.g., usb_wifi_bt_resume.ko). This module exploits Linux's standard PM Notifier mechanism.
Module's Operational Architecture:
 * Dedicated Targeting (Hardcoded): The VID/PID list of the Wifi and Bluetooth modules is declared directly as a constant array right in the C source code. No Sysfs is used, and there is no communication with User-space, ensuring absolute safety and the fastest possible loading speed.
 * Event Hooking: Registers a callback via register_pm_notifier().
 * High-Priority Intervention:
   * At the PM_SUSPEND_PREPARE phase, the module identifies the pointers of the Wifi/BT devices.
   * At the PM_POST_SUSPEND phase (the earliest stage when the CPU has just woken up), the module triggers an ultra-high priority Workqueue (highpri_wq) to call the function that directly wakes up these devices, completely bypassing the Kernel's default PM scheduler for other USB devices.
4. Implementation Plan
Phase 4.1: LKM Source Code Development
 * Declare the list of critical devices:
<!-- end list -->
static const struct usb_device_id wifi_bt_vip_list[] = {
    { USB_DEVICE(0x0bda, 0xc820) }, /* Example: Realtek Wifi/BT Module */
    { USB_DEVICE(0x8087, 0x0aaa) }, /* Example: Intel Bluetooth */
    { } /* Null-terminated */
};
MODULE_DEVICE_TABLE(usb, wifi_bt_vip_list);

 * Write the PM Notifier Hook function to intercept PM_POST_SUSPEND and force resume for devices matching the list above.
Phase 4.2: Build System Integration
 * Integrate the module source code into the build system (Yocto/Buildroot or a standalone Makefile).
 * Ensure the usb_wifi_bt_resume.ko module is automatically loaded (insmod/modprobe) right during the board's boot process.
5. Pros & Risks
Outstanding Advantages (Pros):
 * Fool-proof: Since Sysfs is eliminated, no process or user in User-space can accidentally delete or alter the priority device list.
 * Ultra-fast Initialization and Execution: Bypasses the overhead of initializing the sysfs interface and dynamic memory allocation. The Instant Resume speed reaches its maximum, approaching the speed of the legacy core-intervention patch.
 * Clean Architecture (Zero Core-Modifications): The original Linux Kernel 6.12 source code remains 100% intact, making it easy to maintain and receive security patches from the community.
Risks & Mitigations:
 * Risk: Hardcoding tightly couples the module to a specific hardware configuration. If the Wifi/BT module vendor changes (changing VID/PID), the module will not recognize it.
 * Mitigation: The development team must update the wifi_bt_vip_list array in the C source code and re-compile the module whenever there is a change in the hardware BOM (Bill of Materials).
6. Operational Flowchart (PlantUML)
Below is the architectural flow diagram describing the 3 main phases: Initialization, Suspend, and Instant Resume.
@startuml
!theme plain
skinparam defaultTextAlignment center
skinparam ArrowColor #333333
skinparam packageStyle rectangle

skinparam rectangle {
    BackgroundColor<<LKM>> #D4EDDA
    BorderColor<<LKM>> #28A745
    
    BackgroundColor<<Kernel>> #CCE5FF
    BorderColor<<Kernel>> #007BFF
    
    BackgroundColor<<Instant>> #FFF3CD
    BorderColor<<Instant>> #FFC107
}

package "Phase 1: Module Initialization (Hardcoded Wifi/BT)" {
    rectangle "System Boot / Load LKM" as boot
    rectangle "LKM: Load hardcoded\nWifi/BT VID/PID list" as load_ids <<LKM>>
    rectangle "LKM: Register PM Notifier\nwith Kernel Core" as register_pm <<LKM>>
    rectangle "LKM: Standby for\nevents" as standby <<LKM>>
    
    boot --> load_ids
    load_ids --> register_pm
    register_pm --> standby
}

package "Phase 2: System Enters Sleep Mode (Suspend)" {
    rectangle "System receives Suspend command" as suspend_cmd
    rectangle "Kernel Core emits\nPM_SUSPEND_PREPARE event" as pm_suspend_prep <<Kernel>>
    rectangle "LKM: Callback catches event" as lkm_catch_suspend <<LKM>>
    hexagon "Scan USB Bus:\nAre there Wifi/BT devices?" as check_wifi <<LKM>>
    rectangle "LKM: Mark Wifi/BT Pointers,\nprepare priority thread" as mark_pointer <<LKM>>
    rectangle "LKM: Ignore" as ignore_suspend <<LKM>>
    rectangle "System completes\nSuspend and sleeps" as sleep_done
    
    suspend_cmd --> pm_suspend_prep
    pm_suspend_prep --> lkm_catch_suspend
    lkm_catch_suspend --> check_wifi
    check_wifi -down-> mark_pointer : Match found
    check_wifi -right-> ignore_suspend : No match
    mark_pointer --> sleep_done
    ignore_suspend --> sleep_done
}

package "Phase 3: System Wakes Up (Instant Resume for Wifi/BT)" {
    rectangle "Receive Wake-up signal" as wakeup_cmd
    rectangle "Kernel Core emits\nPM_POST_SUSPEND event (Early stage)" as pm_post_suspend <<Kernel>>
    rectangle "LKM: Activate highest\npriority Workqueue" as wq_activate <<LKM>>
    rectangle "LKM: Forcefully call resume_device\nDIRECTLY for Wifi/BT" as force_resume <<LKM>>
    rectangle "Wifi/Bluetooth wakes up INSTANTLY\n(Instant Resume)" as instant_wake <<Instant>>
    rectangle "Wifi/BT network ready\nto reconnect immediately" as network_ready
    rectangle "Kernel PM Core sequentially\nresumes other USBs (Mouse, KB...)" as normal_resume <<Kernel>>
    rectangle "System completes Resume" as resume_done
    
    wakeup_cmd --> pm_post_suspend
    pm_post_suspend --> wq_activate
    wq_activate --> force_resume
    force_resume --> instant_wake
    instant_wake -down-> network_ready
    instant_wake -right-> normal_resume
    normal_resume --> resume_done
}
@enduml

7. Conclusion
The option to develop an independent LKM with a hardcoded device list is a "Dedicated Solution" design. It completely protects the original Linux 6.12 kernel source code, eliminates the risk of misconfiguration from User-space, and ensures that the network module (Wifi/BT) always has the absolute highest wake-up privilege in the system. We recommend approval to proceed with PoC programming and real-world testing on the board.
Would you like me to generate a side-by-side comparison table of Option 1 and Option 3 to include as an appendix for your presentation?
