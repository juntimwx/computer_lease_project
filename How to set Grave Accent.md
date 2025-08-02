# 📜รวบรวมแนวทางการตั้งค่าเปลี่ยนภาษา

## ความต้องการและปัญหาที่เจอ

1. ต้องการตั้งค่าให้สามารถเปลี่ยนภาษาได้ด้วยปุ่ม Grave Accent หรือ ตัวหนอน
2. เดิมมีการตั้ง GPO ไว้แล้ว แต่ไม่สามารถใช้งานได้กับเครื่องที่เปลี่ยน profile อยู่ตลอดเวลา (profile อยู่บน Ad หลัก) เช่น notebook หรือ เครื่องอื่นๆ ที่ใช้งานด้วย WiFi
3. มีการตั้ง GPO ที่ computer config. แทน user config แล้วแต่ก็ยังใช้งานไม่ได้
4. เขียน script และตั้งใช้งานผ่าน GPO แล้ว แต่ก็ยังไม่สามารถใช้งานได้
5. ทดสอบ run script ดังนี้

```powershell
# powershell
# ตั้งให้ใช้ Grave Accent (~) เป็น hotkey สลับภาษาป้อนข้อมูล (input language)
$regPath = "HKCU:\Keyboard Layout\Toggle"
Set-ItemProperty -Path $regPath -Name "Hotkey" -Value "4" -Type String -Force
```

สามารถเปลี่ยนภาษาได้จริง แต่การตั้งค่านี้ไม่ตามไปที่ profile ใหม่ๆ ที่ login ด้วย

## แนวทางและวิธีที่ใช้ในปัจจุบัน

1. สำหรับ โปรไฟล์ที่มีอยู่แล้วบนเครื่องเดียว (local profiles) — รันแบบ elevated เพื่อเขียนในแต่ละ NTUSER.DAT
ถ้าอยากแก้ไขโปรไฟล์ที่มีอยู่แล้ว (เช่นผู้ใช้หลายคนที่ล็อกอินไปก่อนหน้า) โดยไม่ต้องให้เขาล็อกอินเองก่อน ให้รันสคริปต์นี้แบบ Admin / elevated:

```powershell
# powershell
# แก้ทุกโปรไฟล์ใน C:\Users (ยกเว้น Default, Public ฯลฯ)
$exclude = @('Public','Default','Default User','All Users')
$users = Get-ChildItem 'C:\Users' -Directory | Where-Object { $exclude -notcontains $_.Name }

foreach ($u in $users) {
    $hivePath = Join-Path $u.FullName 'NTUSER.DAT'
    if (-not (Test-Path $hivePath)) { continue }
    $tempKey = "HKU\TempHive_$($u.Name)"
    try {
        & reg.exe load $tempKey $hivePath | Out-Null

        $base = "$tempKey\Keyboard Layout\Toggle"
        # สร้าง key ถ้ายังไม่มี
        & reg.exe add $base /f | Out-Null

        # ตั้งค่าที่ต้องการ
        & reg.exe add $base /v "Language Hotkey" /t REG_SZ /d "4" /f | Out-Null
        & reg.exe add $base /v "Layout Hotkey" /t REG_SZ /d "3" /f | Out-Null

        # ลบ fallback ถ้ามี
        & reg.exe delete $base /v "Hotkey" /f 2>$null

    } finally {
        & reg.exe unload $tempKey | Out-Null
    }
}
```

- สคริปต์นี้โหลดแต่ละ NTUSER.DAT เป็น hive ชั่วคราว, เขียนค่าที่ต้องการ, แล้ว unload
- เหมาะกับการแก้ไข existing local profiles โดยไม่ต้องให้ผู้ใช้ล็อกอิน

- สำหรับ โปรไฟล์ใหม่ที่สร้างในอนาคต — แก้ใน Default User
ให้แก้ไข Default profile เพื่อให้ user ใหม่ที่ล็อกอินได้ค่าพวกนี้ตั้งแต่แรก :

```powershell
# powershell
# แก้ Default user hive
$defaultHive = 'C:\Users\Default\NTUSER.DAT'
$tempKey = 'HKU\DefaultTemp'
& reg.exe load $tempKey $defaultHive

$base = "$tempKey\Keyboard Layout\Toggle"
& reg.exe add $base /f
& reg.exe add $base /v "Language Hotkey" /t REG_SZ /d "4" /f
& reg.exe add $base /v "Layout Hotkey" /t REG_SZ /d "3" /f
& reg.exe delete $base /v "Hotkey" /f 2>$null

& reg.exe unload $tempKey
```

- หลังจากนี้ user ใหม่ที่สร้างโปรไฟล์จะได้รับค่าตามที่ตั้งไว้ทันทีเมื่อล็อกอินแรก
