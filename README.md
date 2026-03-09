# notes

## Token
`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`

Note over Kth: Thực thi Logic Parallel
Resume [8, 9]
Kth->>Core:
usb_autoresume_device (WiFi_udev)
Core->> Dev: Hardware Resume (WiFi)
Note over Kth: Chờ WiFi hoàn tất
(WIFI_THEN_BT logic) [10-12]
Kth->>Kth:
wait _for_completion_timeout(wifi_done) [8-10]
Kth->>Core:
usb_autoresume_device (BT_udev)
Core->>Dev: Hardware Resume (BT)
Kth->> Mod: complete_all(resume_done)
[13-15]
deactivate Kth
else Thiết bị USB thông thường
PM->>Core: dpm_resume() (Xử lý tuần tự
mặc định)
end
Note over Mod: Quá trình Resume ưu tiên kết
thúc
