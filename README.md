.
Here is the English translation of your Technical Proposal. I have maintained the professional, technical, and structured tone of the original text so that it is ready to be presented to your Board of Directors (BOD), Project Manager, or System Architecture Team.
Below is the detailed Technical Proposal for Option 3: Developing an Independent Kernel Module (Out-of-tree LKM).
This proposal is designed for you to present to the Board of Directors (BOD), Project Manager (PM), or System Architecture Team when the project has strict requirements for ultra-low latency and priority control for specific USB devices, which the native solution in Option 1 cannot fully fulfill.
TECHNICAL PROPOSAL: DEVELOPING AN INDEPENDENT KERNEL MODULE FOR USB RESUME MANAGEMENT ON LINUX 6.12
1. Executive Summary
During the system migration to Linux Kernel 6.12, maintaining direct patches to the kernel source code (especially drivers/usb/core/hub.c and drivers/base/power/main.c) to achieve parallel resume or instant resume is no longer feasible due to internal architectural changes in the Kernel and a high risk of conflicts.
This proposal presents Option 3: Building a fully independent Loadable Kernel Module (LKM) located outside the kernel tree (Out-of-tree). This module will act as a hidden "conductor," utilizing standard APIs of the Power Management (PM) system to intercept Suspend/Resume events and proactively wake up high-priority USB devices without modifying any core/host code lines.
2. Background & Problem
 * Legacy Solution (Legacy Patch): Using global variables like instant_ctrl and prevent_disconnection_resume, and directly modifying the generic_resume function to force devices to wake up instantly. Direct intervention in the Kernel's power management flow (DPM - Device Power Management) creates hard-coupling and massive technical debt.
 * Limitations of Option 1 (Native Async PM): Although clean and safe, Option 1 only helps devices wake up in parallel; the Kernel itself decides which thread runs first. We lose the ability to specify "Device A must wake up first."
 * Objective: We need a mechanism that allows Instant Resume for critical devices (such as display clusters, high-priority communication modules) at a speed comparable to the legacy patch, while ensuring the architectural design meets the "Plug-and-Play LKM" standard on Kernel 6.12.
3. Proposed Solution
Design a .ko Kernel Module (e.g., usb_priority_resume.ko). This module will utilize Linux's PM Notifier and Bus Notifier mechanisms.
Module's Operational Architecture:
 * Event Hooking: The module registers a callback with the system via register_pm_notifier(). When the Kernel issues a prepare-to-sleep command (PM_SUSPEND_PREPARE), this callback is triggered.
 * Device Scanning: The module iterates through the USB bus (using bus_for_each_dev). It compares the Vendor ID (VID) and Product ID (PID) or Port Number against a predefined list of "VIP Devices."
 * Intervention: * For VIP devices: The module will inject special flags, such as automatically calling device_enable_async_suspend() from Kernel-space, or attaching the USB_QUIRK_RESET_RESUME flag to the udev structure to force a hot reset upon wake-up (helping to bypass slow sequential initialization steps).
   * Priority Thread Configuration: Right at the PM_POST_SUSPEND phase (when the CPU has just woken up, but peripherals are still sleeping), the Module triggers an ultra-high priority Workqueue (highpri_wq) to directly invoke the resume command for these VIP devices before the Kernel PM Core has a chance to traverse the standard list.
4. Implementation Plan
Phase 4.1: Developing the LKM Skeleton
 * Write a basic C file (usb_priority_resume.c) with module_init and module_exit functions.
 * Register bus_register_notifier(&usb_bus_type, &usb_notifier) to detect as soon as a VIP device is plugged in.
Phase 4.2: Implementing Instant Resume Logic
 * Initialize a Linked List to manage currently connected VIP devices.
 * Write a callback function for the PM Notifier:
<!-- end list -->
static int pm_notify_callback(struct notifier_block *nb, unsigned long action, void *data) {
    switch (action) {
        case PM_SUSPEND_PREPARE:
            // Mark state, prepare for the sleep process
            break;
        case PM_POST_SUSPEND:
            // Call high-priority Workqueue to instantly force resume on VIP devices
            wake_up_vip_devices_instantly();
            break;
    }
    return NOTIFY_OK;
}

Phase 4.3: Dynamic Configuration via Sysfs
 * Create a Sysfs interface (e.g., /sys/kernel/usb_priority_resume/vip_list) allowing dynamic configuration (from User-space) of the VIDs/PIDs that need prioritization without needing to re-compile the module.
5. Pros & Risks Assessment
Pros:
 * Ultra-low Latency: Restores Instant Resume capabilities similar to the legacy solution. VIP devices can wake up 30-50% faster than standard sequential Async methods.
 * Clean Architecture: Leaves the kernel core source code (hub.c, main.c) untouched. The source code is highly maintainable and makes it easy to upstream the system kernel to 6.12 or newer without encountering git merge conflicts.
 * Stateful: Does not use unbind/bind commands. User-space applications will not experience connection drops or I/O errors.
Risks & Mitigations:
 * Risk: Deep intervention using a high-priority Workqueue can cause Race Conditions with the Kernel's native PM system, potentially leading to a Kernel Panic if memory or locks (Mutex/Spinlock) are not strictly managed.
 * Mitigation: The development team (Kernel Devs) must strictly adhere to Linux Locking rules; utilize the Lockdep tool in the Kernel configuration to detect deadlocks during development; and conduct thorough testing on the QEMU platform before flashing to the physical board.
6. Test Plan
 * Basic Functional Testing:
   * Load the module (insmod usb_priority_resume.ko).
   * Write the VID/PID of the touchscreen to the sysfs configuration file.
   * Run the system sleep command (echo mem > /sys/power/state). Connect the monitor and observe the latency when the system wakes up.
 * Performance Profiling:
   * Use analyze_suspend.py.
   * Success Criterion: The dmesg graph shows the VIP device's resume function completing before other USB devices can even begin their recovery process.
 * Stability Test:
   * Enable the CONFIG_PROVE_LOCKING option in the kernel.
   * Run a continuous Suspend/Resume script via UART/SSH port for 72 hours. Objective: 0 Kernel Oops, 0 Memory Leaks.
7. Conclusion
Building an independent LKM is the perfect combination of the legacy patch's Performance and the Safety and Maintainability of the Linux 6.12 platform. This option requires a higher level of Kernel-space C programming effort than Option 1 (Udev), but it is the only viable solution if the project strictly mandates an "Instant Resume" feature for critical peripherals. We respectfully request budget/resource approval to proceed with developing a Proof of Concept (PoC) for this module.
Would you like me to format this into a markdown file so you can easily copy and paste it into GitHub, Confluence, or your internal wiki?


graph TD
    classDef sysfs fill:#f9f,stroke:#333,stroke-width:2px;
    classDef kernel fill:#bbf,stroke:#333,stroke-width:2px;
    classDef lkm fill:#dfd,stroke:#333,stroke-width:2px;
    classDef instant fill:#ff9,stroke:#333,stroke-width:2px,stroke-dasharray: 5 5;

    %% GIAI ĐOẠN 1: KHỞI TẠO VÀ CẤU HÌNH
    subgraph Phase1 ["Giai đoạn 1: Khởi tạo Module và Cấu hình VIP"]
        A1(Boot hệ thống / Tải LKM) --> A2[LKM: Đăng ký PM Notifier và Bus Notifier]:::lkm
        A2 --> A3[LKM: Sẵn sàng lắng nghe sự kiện hệ thống]:::lkm
        A4(User-space: Ghi danh sách VID/PID ưu tiên) --> A5[Sysfs: Cập nhật danh sách VIP Devices]:::sysfs
        A5 --> A6[LKM: Lưu danh sách VIP vào bộ nhớ nội bộ]:::lkm
    end

    %% GIAI ĐOẠN 2: HỆ THỐNG SUSPEND
    subgraph Phase2 ["Giai đoạn 2: Hệ thống đi vào chế độ ngủ (Suspend)"]
        B1(Hệ thống nhận lệnh Suspend) --> B2[Kernel Core phát sự kiện PM_SUSPEND_PREPARE]:::kernel
        B2 --> B3[LKM: PM Callback bắt được sự kiện]:::lkm
        B3 --> B4{Duyệt USB Bus: Có thiết bị VIP không?}:::lkm
        B4 -- Có --> B5[LKM: Đánh dấu thiết bị VIP và chuẩn bị cờ Instant Resume]:::lkm
        B4 -- Không --> B6[LKM: Bỏ qua, không làm gì cả]:::lkm
        B5 --> B7(Hệ thống hoàn tất Suspend và ngủ)
        B6 --> B7
    end

    %% GIAI ĐOẠN 3: HỆ THỐNG RESUME
    subgraph Phase3 ["Giai đoạn 3: Hệ thống thức dậy (Instant Resume)"]
        C1(Nhận tín hiệu Wake-up) --> C2[Kernel Core phát sự kiện PM_POST_SUSPEND (Rất sớm)]:::kernel
        C2 --> C3[LKM: PM Callback bắt được sự kiện sớm]:::lkm
        C3 --> C4[LKM: Kích hoạt High-Priority Workqueue]:::lkm
        C4 --> C5[LKM: Gọi ép hàm resume trực tiếp cho các thiết bị VIP]:::lkm
        C5 --> C6[Thiết bị VIP thức dậy TỨC THÌ (Instant Resume)\nTrước cả khi Kernel kịp xử lý]:::instant
        C6 --> C7[Kernel PM Core bắt đầu duyệt và Resume các thiết bị USB thông thường]:::kernel
        C7 --> C8(Hệ thống hoàn tất Resume)
        C8 --> C9(Ứng dụng trên User-space kết nối lại ngay lập tức với VIP Device)
    end
