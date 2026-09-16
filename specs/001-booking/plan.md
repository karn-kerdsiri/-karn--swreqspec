# แผนงานฟีเจอร์: จองคิวตรวจสุขภาพ (Booking)

## 1. สรุปแนวทาง
- ฟีเจอร์นี้ให้ผู้รับบริการที่ยืนยันตัวตนแล้ว เลือกแพ็กเกจ วัน และช่วงเวลาตรวจสุขภาพ เพื่อดูช่วงเวลาและจำนวนที่นั่งคงเหลือก่อนยืนยันการจองตาม FR-BKG-01 และ FR-BKG-06
- ระบบจะตรวจว่าผู้รับบริการมีคิวที่ยังไม่ได้ใช้ในวันเดียวกันหรือไม่ เพื่อปฏิเสธการจองซ้ำตาม FR-BKG-02 และแสดงหมายเลขคิวเดิม
- เมื่อช่วงเวลาที่เลือกเต็มจะให้แจ้งเตือนและเสนอ 3 ตัวเลือกใกล้เคียง โดยไม่สร้างรายการจอง ตาม FR-BKG-03 และ AC-BKG-03
- เมื่อยืนยันสำเร็จ ระบบจะบันทึกการจอง ลดจำนวนที่นั่งทันที ออกหมายเลขคิว และส่งคำขอส่งข้อความยืนยันแบบ asynchronous ตาม FR-BKG-04 และ FR-BKG-05
- ภาพรวมของการสร้างจะเน้นระบบจองคิวและการแจ้งเตือนที่เป็นส่วนสำคัญของ UC-01 โดยคงขอบเขตตาม Scope และไม่รวมการยกเลิก/เลื่อนคิวและการจัดการโควตา UC-09

## 2. เทคโนโลยีที่ใช้

| สิ่งที่เลือก | มาจาก | หมายเหตุ |
|---|---|---|
| React (Vite) สำหรับหน้าเว็บผู้รับบริการ | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้แสดงรายการช่วงเวลาและยืนยันการจอง |
| Python FastAPI สำหรับ API แก่บริการจองคิว | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้จัดการความถูกต้องของการจองและสรุปข้อมูลที่ว่าง |
| MySQL | CON-TECH-01 | ใช้เก็บข้อมูลการจอง คิวที่ว่าง และ audit log |
| ระบบแจ้งเตือน SMS/LINE แบบ asynchronous | IF-NOT-01 | ส่งคำขอแจ้งเตือนแยกจากกระบวนการจองหลัก เพื่อให้การจองไม่รอผลส่งข้อความ |
| HIS lookup ด้วยเลขบัตรประชาชนแล้วเก็บ HN เท่านั้น | IF-HIS-01 | ไม่เก็บเลขบัตรประชาชนในตารางการจอง |
| ผู้ใช้ต้องยืนยันตัวตนก่อนเข้าถึงข้อมูลผู้รับบริการ | IF-IDP-01 | ใช้เป็น precondition ก่อนเข้าถึงฟีเจอร์ |
| Audit log ทุกครั้งที่เข้าถึงข้อมูลสุขภาพ | DOM-PDPA-01 | บันทึกผู้เข้าถึง เวลา และรหัสผู้รับบริการ |

## 3. โมเดลข้อมูล

| Entity | ฟิลด์หลัก | รองรับ FR/AC |
|---|---|---|
| Booking | booking_id, patient_hn, package_id, booking_date, slot_id, queue_no, status, created_at, confirmed_at, notification_status | FR-BKG-02, FR-BKG-04, FR-BKG-05, AC-BKG-01, AC-BKG-02, AC-BKG-04 |
| SlotAvailability | slot_id, booking_date, start_time, end_time, package_id, capacity, remaining_seats | FR-BKG-01, FR-BKG-03, FR-BKG-06, AC-BKG-03, AC-BKG-05 |
| Package | package_id, name, duration_minutes, required_criteria | FR-BKG-01, FR-BKG-06 |
| NotificationJob | notification_job_id, booking_id, channel, payload, status, retry_count, next_retry_at, created_at | FR-BKG-05, NFR-REL-02, AC-BKG-04 |
| AuditLog | audit_id, accessed_by, accessed_at, patient_hn, action, resource_type | DOM-PDPA-01, AC-BKG-06 |

หมายเหตุ: ตารางการจองจะเก็บ HN แทนเลขบัตรประชาชน เพื่อปฏิบัติตาม IF-HIS-01 และไม่เก็บเลขบัตรประชาชนในตารางการจอง

## 4. API / หน้าจอ

- GET /api/bookings/slots?dateFrom=YYYY-MM-DD&dateTo=YYYY-MM-DD&packageId=... -> แสดงช่วงเวลาว่างและจำนวนที่นั่งคงเหลือ รองรับ FR-BKG-01 และ FR-BKG-06
- GET /api/bookings/check-existing?date=YYYY-MM-DD&patientHn=... -> ตรวจว่าผู้รับบริการมีคิวที่ยังไม่ได้ใช้ในวันเดียวกันหรือไม่ รองรับ FR-BKG-02
- POST /api/bookings/confirm -> ยืนยันการจองและบันทึกรายการ พร้อมลดจำนวนที่นั่งทันที รองรับ FR-BKG-03, FR-BKG-04, AC-BKG-01, AC-BKG-03
- POST /api/bookings/notifications/retry -> ส่งซ้ำข้อความยืนยัน หากส่งไม่สำเร็จ รองรับ FR-BKG-05, NFR-REL-02, AC-BKG-04
- หน้า BookingSelectionPage -> เลือกแพ็กเกจ วัน และช่วงเวลา สำหรับผู้รับบริการที่ยืนยันตัวตนแล้ว รองรับ FR-BKG-01, FR-BKG-06
- หน้า BookingConfirmationDialog -> แจ้งผลการจองและแสดงหมายเลขคิว พร้อมการแจ้งเตือนแบบ asynchronous รองรับ FR-BKG-04, FR-BKG-05, AC-BKG-01, AC-BKG-04

## 5. ตารางตรวจ Constraints

| Constraint ID | ถูกนำไปใช้ที่ไหนใน plan | สถานะ |
|---|---|---|
| CON-TECH-01 | ตารางเทคโนโลยีที่ใช้: MySQL สำหรับข้อมูลการจองและ audit log | ใช้แล้ว |
| DOM-PDPA-01 | Entity AuditLog และกระบวนการบันทึกทุกครั้งที่เข้าถึงข้อมูลสุขภาพ | ใช้แล้ว |
| IF-IDP-01 | Precondition ในหน้า BookingSelectionPage และ API validation ก่อนเข้าถึงข้อมูลผู้รับบริการ | ใช้แล้ว |
| IF-HIS-01 | Entity Booking เก็บ patient_hn แทนเลขบัตรประชาชน และอ่านข้อมูลจาก HIS ผ่านเลขบัตรประชาชนก่อนเข้าระบบ | ใช้แล้ว |
| IF-NOT-01 | NotificationJob + async queue สำหรับ SMS/LINE และไม่ให้การจองรอผลส่งข้อความ | ใช้แล้ว |

## 6. แผนทดสอบจาก Acceptance Criteria

| AC ID | ชื่อ test | ทดสอบอย่างไร |
|---|---|---|
| AC-BKG-01 | test_AC_BKG_01_confirm_booking_reduces_slot | ตั้งค่า slot 09.00 มีที่นั่งว่าง 1 ที่ แล้วยืนยันการจอง ให้ตรวจว่า booking ถูกบันทึก หมายเลขคิวปรากฏ และ remaining_seats == 0 |
| AC-BKG-02 | test_AC_BKG_02_reject_same_day_active_queue | ตั้งค่าผู้รับบริการมีคิวที่ยังไม่ได้ใช้ในวันเดียวกัน แล้วลองจองใหม่ในวันเดียวกัน ให้ตรวจว่า API ปฏิเสธและคืนหมายเลขคิวเดิม |
| AC-BKG-03 | test_AC_BKG_03_show_alternatives_when_slot_full | ตั้ง slot ที่เลือกเต็มระหว่างการยืนยัน แล้วตรวจว่ามีข้อความ “ช่วงเวลาเต็ม” และเสนอช่วงเวลา 3 ตัวเลือก โดยไม่มีการสร้างการจองซ้อน |
| AC-BKG-04 | test_AC_BKG_04_retry_queue_when_notification_fails | จำลอง SMS/LINE ไม่ตอบสนอง เมื่อยืนยันการจอง ให้ตรวจว่าการจองยังถูกบันทึก แสดงหมายเลขคิว และมี task ใน retry queue ภายใน 5 นาที |
| AC-BKG-05 | test_AC_BKG_05_slot_lookup_p95_under_2s | จำลองผู้ใช้พร้อมกัน 200 คน เรียก GET /api/bookings/slots แล้วตรวจค่า p95 <= 2 วินาที |
| AC-BKG-06 | test_AC_BKG_06_audit_log_written_after_access | พยายามเข้าถึงข้อมูลการจองแบบจำลอง แล้วตรวจว่ามี audit log ที่ระบุผู้เข้าถึง เวลา และ patient_hn |

## 7. ลำดับงาน

1. วิเคราะห์ข้อมูลและกำหนด entity Booking, SlotAvailability, NotificationJob, AuditLog ตาม FR-BKG-01, FR-BKG-02, DOM-PDPA-01
2. สร้าง schema MySQL และ migration สำหรับ booking/slot/notification/audit ตาม CON-TECH-01 และ IF-HIS-01
3. สร้าง API ดึงช่วงเวลาและจำนวนที่นั่งคงเหลือ พร้อมรองรับแพ็กเกจและวันที่ 30 วันข้างหน้า ตาม FR-BKG-01, FR-BKG-06, AC-BKG-05
4. สร้าง API ตรวจสิทธิ์และตรวจคิวในวันเดียวกัน ตาม FR-BKG-02 และ AC-BKG-02
5. สร้าง workflow ยืนยันการจอง รั้ง slot และบันทึกการจอง ตาม FR-BKG-03, FR-BKG-04, AC-BKG-01, AC-BKG-03
6. สร้าง retry queue สำหรับ SMS/LINE แบบ asynchronous ตาม FR-BKG-05, NFR-REL-02, AC-BKG-04
7. เพิ่ม audit logging และ validation ของ precondition การยืนยันตัวตน ตาม IF-IDP-01, DOM-PDPA-01, AC-BKG-06
8. ทดสอบ end-to-end ตาม AC-BKG-01 ถึง AC-BKG-06 และตรวจการเชื่อมโยงกับ FR-BKG ทั้งหมด

## 8. สิ่งที่ยังไม่ทำ
- Q-01 “ช่วงเวลาใกล้เคียง” นับเฉพาะวันเดียวกัน หรือรวมวันถัดไปด้วย? -> ส่วนที่เกี่ยวข้องกับข้อนี้จะยังไม่สร้างจนกว่าจะได้คำตอบ
- Q-02 หมายเลขคิวรีเซ็ตรายวัน หรือนับต่อเนื่อง? -> ส่วนที่เกี่ยวข้องกับข้อนี้จะยังไม่สร้างจนกว่าจะได้คำตอบ

