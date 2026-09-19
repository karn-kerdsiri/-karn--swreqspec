# Plan: กรอกข้อมูลผลการศึกษา

## 1. สรุปแนวทาง
- ฟีเจอร์นี้ให้นักศึกษากรอกข้อมูลผลการศึกษาและแนบไฟล์ Transcript เพื่อส่งข้อมูลที่ต้องผ่านการตรวจความครบถ้วนและรูปแบบก่อนบันทึก
- ผู้ใช้หลักคือนักศึกษาในรอบเปิดของแบบฟอร์ม และระบบจะคัดกรองข้อมูลและไฟล์ตามข้อกำหนดใน DR-EDU-01 ถึง DR-EDU-07
- แนวทางคือแยกงานออกเป็น 4 ส่วนหลัก: แสดงฟอร์ม, ตรวจข้อมูล/ไฟล์, บันทึกข้อมูลร่วมกับไฟล์, และแจ้งเตือนหลังบันทึกสำเร็จ
- ระบบจะใช้ข้อกำหนดใน IF-FILE-01, IF-FILE-02 และ DOM-PDPA-01/02 เพื่อจัดการความปลอดภัยของไฟล์และ audit log โดยคงแนวทาง 1 รอบต่อ 1 นักศึกษา ตาม CON-DATA-01
- หลังบันทึกสำเร็จจะสร้างการแจ้งเตือนในระบบและส่งคำขออีเมลเข้าคิวตาม FR-EDU-09 และ FR-EDU-10 โดยไม่รอผลการส่งอีเมล

## 2. เทคโนโลยีที่ใช้

| สิ่งที่เลือก | มาจาก | หมายเหตุ |
|---|---|---|
| React (Vite) | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้สร้างหน้าฟอร์มแบบฟอร์มผลการศึกษาและหน้าสรุปส่งข้อมูล |
| Python FastAPI | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้จัดการ API validation, submit flow และ business logic |
| Supabase Postgres | IF-FILE-01, CON-DATA-01 | ใช้เก็บข้อมูลผลการศึกษาและ metadata ของไฟล์ตาม unique student_id + round_id |
| Supabase Storage | IF-FILE-01, IF-FILE-02 | ใช้เก็บ Transcript แบบ private และเข้าถึงผ่าน signed URL เท่านั้น |
| Queue สำหรับส่งอีเมลแบบ asynchronous | IF-MAIL-01 | การบันทึกต้องไม่รอผลการส่งอีเมล |
| Audit log store | DOM-PDPA-01, DOM-PDPA-02 | บันทึกผู้กระทำ เวลา รหัสนักศึกษา และชนิดการกระทำอย่างน้อย 1 ปี |

## 3. โมเดลข้อมูล

| Entity | ฟิลด์หลัก | รองรับ FR/Constraint |
|---|---|---|
| AcademicRecord | id, student_id, round_id, education_level, institution_name, major, academic_year, semester, gpa, total_credits, status, created_at, updated_at | FR-EDU-08, FR-EDU-12, CON-DATA-01, DR-EDU-01 ถึง DR-EDU-05 |
| AcademicRecordFile | id, academic_record_id, file_type, storage_bucket, storage_path, original_name, mime_type, file_size, uploaded_at | FR-EDU-03, FR-EDU-07, FR-EDU-13, FR-EDU-14, DR-EDU-06, DR-EDU-07, IF-FILE-01, IF-FILE-02 |
| AuditLog | id, actor_user_id, student_id, action_type, action_at, record_id, details | DOM-PDPA-01, DOM-PDPA-02 |
| Notification | id, student_id, type, message, created_at, status, retry_count, next_retry_at | FR-EDU-09, FR-EDU-10, IF-MAIL-01, NFR-REL-01 |
| FormRound | id, round_id, status, open_at, close_at | IF-FORM-01, FR-EDU-02 |

หมายเหตุ: ไม่มีฟิลด์ใดที่เก็บเลขบัตรประชาชนหรือข้อมูลที่ไม่ระบุใน DR-EDU-01 และตาม constraint ไม่มีการเก็บข้อมูลที่ไม่จำเป็นเพิ่มเติม

## 4. API / หน้าจอ

| รายการ | รายละเอียด | รองรับ FR |
|---|---|---|
| GET /student/academic-records/form?round_id=... | เปิดฟอร์มกรอกข้อมูลตามรอบ เปิดอยู่หรือปิดอยู่ และแสดงสถานะ closed เมื่อเป็น read-only | FR-EDU-01, FR-EDU-02 |
| GET /student/academic-records/{round_id} | ดึงข้อมูลเดิมของนักศึกษาคนปัจจุบันเพื่อแสดงในฟอร์มเมื่อมีข้อมูลอยู่แล้ว | FR-EDU-12 |
| POST /student/academic-records/validate | ตรวจความครบถ้วนและรูปแบบของฟิลด์ตาม DR-EDU-02 ถึง DR-EDU-05 | FR-EDU-05, FR-EDU-06 |
| POST /student/academic-records/upload-transcript | ตรวจชนิด/ขนาด/จำนวนไฟล์ และยอมรับเฉพาะ Transcript | FR-EDU-03, FR-EDU-07, FR-EDU-13, DR-EDU-06, DR-EDU-07 |
| POST /student/academic-records/preview | สร้างหน้าสรุปข้อมูลและรายชื่อไฟล์แนบก่อนยืนยันส่ง | FR-EDU-04 |
| POST /student/academic-records/submit | บันทึกข้อมูล หลักฐาน และสร้าง notification/queue พร้อม rollback หาก DB error | FR-EDU-08, FR-EDU-09, FR-EDU-10, FR-EDU-11, FR-EDU-14 |
| GET /storage/transcript/{path} | เข้าถึงไฟล์ผ่าน signed URL เท่านั้น ไม่อนุญาตให้เรียก path ตรง | IF-FILE-02, NFR-SEC-03 |
| GET /notifications | แสดงการแจ้งเตือนในระบบหลังบันทึกสำเร็จ | FR-EDU-09 |

## 5. ตารางตรวจ Constraints

| Constraint ID | ถูกนำไปใช้ที่ไหนใน plan | สถานะ |
|---|---|---|
| IF-AUTH-01 | ทุก request ของ submit/validate/upload จะตรวจ session ก่อนเข้าการทำงาน | ใช้แล้ว |
| IF-FORM-01 | FormRound status ถูกอ่านก่อนเปิดฟอร์มหรือ submit ทุกครั้ง | ใช้แล้ว |
| IF-FILE-01 | AcademicRecordFile เก็บใน Supabase Storage และมี bucket + storage_path เป็น metadata | ใช้แล้ว |
| IF-FILE-02 | storage access ของ Transcript จะผ่าน signed URL และมีระยะเวลาไม่เกิน 15 นาที | ใช้แล้ว |
| IF-MAIL-01 | Notification queue จะแยกส่งอีเมลแบบ asynchronous ไม่ delay submit flow | ใช้แล้ว |
| DOM-PDPA-01 | ทุกการ create/update/view ของ AcademicRecord จะสร้าง AuditLog | ใช้แล้ว |
| DOM-PDPA-02 | AuditLog schema ครอบคลุม actor_user_id, student_id, action_type, action_at | ใช้แล้ว |
| CON-DATA-01 | AcademicRecord model ใช้ unique index student_id + round_id และ submit flow update โครงสร้างเดิมแทนการสร้างซ้ำ | ใช้แล้ว |

## 6. แผนทดสอบจาก Acceptance Criteria

| AC ID | ชื่อ test | ทดสอบอย่างไร |
|---|---|---|
| AC-EDU-01 | test_AC_EDU_01_submit_success_and_status | ใช้ session นักศึกษา, รอบเปิด, กรอกข้อมูลครบ ถูกต้อง และแนบ Transcript 1 ไฟล์ แล้ว verify status = รอตรวจสอบคุณสมบัติ และมีข้อความยืนยัน |
| AC-EDU-02 | test_AC_EDU_02_missing_required_gpax_and_semester | เว้นว่าง GPAX และภาคเรียน แล้ว submit; verify ไม่มี row ใหม่ และแสดง error ที่สองฟิลด์ |
| AC-EDU-03 | test_AC_EDU_03_invalid_gpax_range | กรอก GPAX = 4.50 แล้ว submit; verify fail และ error ระบุช่วงที่ถูกต้อง |
| AC-EDU-04 | test_AC_EDU_04_reject_invalid_file_type | แนบไฟล์ .exe ขนาด 1 MB; verify upload ถูก reject โดยไม่ล้างข้อมูลที่กรอกไว้ |
| AC-EDU-05 | test_AC_EDU_05_db_failure_rolls_back_upload | simulate DB failure หลัง upload สำเร็จ; verify AcademicRecord ไม่เกิด และ file ถูกลบจาก Supabase Storage |
| AC-EDU-06 | test_AC_EDU_06_email_failure_keeps_record_and_retry_queue | simulate email service timeout; verify record still saved, notification system created, and retry queue exists |
| AC-EDU-07 | test_AC_EDU_07_closed_round_is_read_only | รอบ closed; access form and submit; verify UI read-only และ response HTTP 409 |
| AC-EDU-08 | test_AC_EDU_08_update_existing_record_for_same_round | มี record เก่าแล้ว; submit อีกครั้ง; verify update ไม่สร้าง row เพิ่ม และ unique count เท่ากับ 1 |
| AC-EDU-09 | test_AC_EDU_09_other_student_data_forbidden | ใช้ session นักศึกษา A เรียกข้อมูลนักศึกษา B; verify HTTP 403 |
| AC-EDU-10 | test_AC_EDU_10_audit_log_created | ดำเนินการ create/view/edit; verify มี AuditLog 1 รายการที่มี actor, time, student_id, action_type |
| AC-EDU-11 | test_AC_EDU_11_p95_latency_under_2_seconds | load test 300 concurrent users; verify p95 response <= 2 วินาที |
| AC-EDU-12 | test_AC_EDU_12_required_markers_and_summary_review | เปิดฟอร์มยืนยันเครื่องหมาย * และหน้าสรุปแสดงข้อมูลครบตรงกับผู้ใช้กรอก |
| AC-EDU-13 | test_AC_EDU_13_transcript_required | ส่งข้อมูลโดยไม่มี Transcript; verify fail และข้อความแจ้งต้องแนบ Transcript อย่างน้อย 1 ไฟล์ |
| AC-EDU-14 | test_AC_EDU_14_direct_storage_access_denied | เรียก storage path โดยตรงโดยไม่มี signed URL; verify HTTP 401 หรือ 403 |

## 7. ลำดับงาน

1. สร้าง schema ของ AcademicRecord, AcademicRecordFile, AuditLog และ Notification พร้อม index ตาม CON-DATA-01 และ DOM-PDPA-02 (รองรับ FR-EDU-08, FR-EDU-12, DOM-PDPA-01, DOM-PDPA-02)
2. สร้าง API เปิดฟอร์มและโหลดข้อมูลเดิมของรอบปัจจุบัน รวมทั้งตรวจ session และ status round (รองรับ FR-EDU-01, FR-EDU-02, FR-EDU-12, IF-AUTH-01, IF-FORM-01)
3. สร้าง validation สำหรับฟิลด์บังคับและรูปแบบข้อมูลตาม DR-EDU-02 ถึง DR-EDU-05 พร้อมข้อความ error รายตัว (รองรับ FR-EDU-05, FR-EDU-06)
4. สร้าง upload Transcript พร้อมตรวจชนิด/ขนาด/จำนวนไฟล์ และจัดเก็บใน Supabase Storage แบบ private (รองรับ FR-EDU-03, FR-EDU-07, DR-EDU-06, DR-EDU-07, IF-FILE-01, IF-FILE-02)
5. สร้างหน้าสรุปข้อมูลและรายชื่อไฟล์ก่อนยืนยันส่ง (รองรับ FR-EDU-04, AC-EDU-12)
6. สร้าง submit flow: save record, save file metadata, create notification, enqueue email job, และ rollback เมื่อ DB error (รองรับ FR-EDU-08, FR-EDU-09, FR-EDU-10, FR-EDU-11, FR-EDU-14, AC-EDU-05, AC-EDU-06)
7. เพิ่ม audit log สำหรับทุก create/update/view ของข้อมูลและตรวจสิทธิ์ผู้ใช้ (รองรับ NFR-SEC-02, DOM-PDPA-01, DOM-PDPA-02, AC-EDU-09, AC-EDU-10)
8. ทดสอบความเร็วและความปลอดภัยรวมกัน: p95 latency, signed URL access, และ closed-round behavior (รองรับ AC-EDU-07, AC-EDU-11, AC-EDU-14)

## 8. สิ่งที่ยังไม่ทำ
- Open Questions: ไม่มีใน spec ปัจจุบัน (ข้อมูลทั้งหมดถูกย้ายเป็น Assumptions ในหัวข้อ Assumptions & Open Questions แล้ว)
- ส่วนที่เกี่ยวข้องกับข้อนี้จะยังไม่สร้างจนกว่าจะได้คำตอบ: ไม่มีข้อที่ต้องรอคำตอบเพิ่มเติมในเวอร์ชันปัจจุบัน

---

หมายเหตุ: Plan นี้สร้างจาก spec.md เท่านั้น โดยยึด ID ของ FR / NFR / CON / IF / DOM / AC และไม่มีการเพิ่ม requirement ใหม่นอกเหนือจาก spec
