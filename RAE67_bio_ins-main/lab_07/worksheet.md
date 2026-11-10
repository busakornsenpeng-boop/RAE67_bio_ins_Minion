# Lab07 student worksheet

ชื่อกลุ่ม: Minionและผ่องเพื่อน (Busakorn Senpeng 6752500029, Panida Kansrang 6752500169, Thanit Wongsripuek 6752500436, Baramee Sampaotong 6752500380) วันที่: 10/11/2569 policy version/SHA: `lamp_week07/student_policy.py` (เติม TODO จาก starter เป็นตารางเดียวกับ Lab06) SHA-256 `21791d61e2b64a3e283092d5d7a2264d074e4576b014872602cde8fa9886d0d5` ผ่าน `check_submission.py --contract group` และ baseline 40/40 pairs

1. **Graph** (`results/figures/lab07_ros_graph.png`): `synthetic_inputs` → `interaction` (guard + FSM + bounds/slew) → `output_collector` เป็น 3 process เจ็ด topics: `/lamp/input/button` Bool, `/lamp/input/touch` Bool, `/lamp/input/distance` Float32 [m], `/lamp/interaction` String (event หรือ empty), `/lamp/emotion` String, `/lamp/servo_command` Float32MultiArray [tilt, extend, nod] rad, `/lamp/rgb_command` ColorRGBA (r, g, b ∈ [0, 1], a = 1) QoS ทุกตัว RELIABLE, KEEP_LAST depth 10, VOLATILE **Extension ที่ต้องทำ:** ESP32 sensor node (micro-ROS client อ่านปุ่ม, touch, ToF และเก็บ sequence/acquisition time ใน sidecar), micro-ROS Agent บน PC (serial/UDP ↔ DDS) และ ESP32 actuator node (servo + LED, watchdog, slew ตามเวลาจริง, calibration rad → PWM) Boundary ที่มี timestamp: เฉพาะ log sidecar ของแต่ละ node (`perf_counter_ns`) ไม่มีบน wire

2. **หน่วย:** servo เป็น rad ลำดับ `[tilt, extend, nod]` (extend ยังเป็น rad ของ semantic joint ไม่ใช่ m) RGB = ค่า Lab06/255 alpha = 1 ตัวอย่าง: happy → servo `[0.10, 0.10, 0.18·sin(2π·elapsed)]` rad, rgb `(40/255, 220/255, 80/255, 1) = (0.157, 0.863, 0.314, 1.0)`; afraid → `[−0.22, −0.22, −0.16]` rad, rgb `(0.902, 0.118, 0.118, 1.0)` **แผน calibration:** วัดมุมจริงของแต่ละ servo ที่ 5 จุดด้วย protractor สร้างตาราง rad → PWM ตรวจทิศเครื่องหมายและ mechanical limit ก่อน เริ่มด้วย bound ที่แคบกว่าค่า model และวัด LED ด้วย photodiode เพื่อตรวจว่า 0–1 map เป็นความสว่างเชิงเส้นหรือไม่

3. **ประกาศก่อนรัน:**
   - Question: profile C1 (delay 0–0.18 s + random drop 8% + burst drop 12.50–12.70 s) เปลี่ยน response timing และ robustness ของ policy เดิมเทียบกับ C0 (ideal transport) อย่างไร
   - Hypothesis: C1 มี both latency ของ primary inputs สูงกว่า C0 ประมาณเท่ากับ delay ของ message แรกที่ถูก accept (0.02–0.18 s) แต่ยังต่ำกว่า 1 s, มี fault ticks และ state transitions มากกว่า C0 และอาจพลาด primary 0–1 input ต่อ seed
   - Expected input IDs: baseline 7 ตัว `button_01, touch_01, approach_01, clear_01, near_01, clear_02, button_02` (policy ใช้ baseline mapping ไม่ได้เปลี่ยน positive mapping) deadline 1 s, observation window 22 s ทั้ง trial
   - Condition/control: C0 vs C1 คู่ seed 41/42/43 ใช้ source stimuli, policy, guard, bounds/slew, dt 0.02 s และ scheduler 0.04 s เหมือนกัน
   - Clock/measurement boundary: offline ใช้ virtual seconds เดียวกันทุก role ใน process เดียว latency คือ applied command onset ไม่ใช่ ROS/DDS หรือ physical latency

4. **ผลสาม paired repeats** คำสั่ง: `python offline_lab.py --condition all --repeats 3 --output-dir results/completed_starter` ไฟล์: `RAE67_bio_ins-main/lab_07/results/completed_starter/{C0,C1}_r0{1,2,3}/` (input_trace.csv, command_trace.csv, events.csv, metrics.json, config.json), `results/completed_starter/summary.json` และ `manifest.json` (SHA-256) Metric 1 = primary pass ≤ 1 s (denominator = 7 primary IDs) / median both latency, Metric 2 = fault ticks (denominator = 1101 ticks)

| Pair/seed | C0metric1 | C1metric1 | C0metric2 | C1metric2 | Missing/censored | Interpretation |
|---|---|---|---|---|---|---|
| 1/41 | 7/7, median 0.04 s (max 0.14) | 6/7, median 0.15 s (max 0.20) | 20 | 54 | C1 missing clear_02; C1 censored 15 msgs, dropped 257/3303 | stale fault ของ button channel ที่ 9.02 s ส่ง lamp ไป normal ก่อน clear มาถึง clear จึงไม่มี state change ใหม่ |
| 2/42 | 7/7, median 0.06 s (max 0.14) | 6/7, median 0.14 s (max 0.22) | 20 | 81 | C1 missing button_02; censored 9, dropped 287 | stale fault ของ touch channel ซ้ำรอบ 18 s ทำให้สลับ normal ↔ sleepy (fault/idle) และ button ไม่ถูก recognize transitions เพิ่มเป็น 36 |
| 3/43 | 7/7, median 0.04 s (max 0.14) | 7/7, median 0.10 s (max 0.18) | 20 | 56 | ไม่มี missing; censored 12, dropped 300 | ทุก primary ผ่าน latency เพิ่ม 0.00–0.08 s ตาม delay |

ทุก trial เก็บ 10 source inputs (touch_conflict, invalid_01, stale_01 ไม่มี onset ทั้ง C0/C1 ตามที่ policy กำหนด) C1 มี received = accepted + discarded out-of-order และ discarded out-of-order 1393–1421 messages (ประมาณ 46% ของ received) เพราะ delay สุ่มทำให้ message ใหม่มาก่อนเก่า mean transport age 0.077 s, max 0.18 s การ replay `results/completed_starter/C1_r02/input_trace.csv` ให้ command_trace, events และ input_trace **เหมือนกันทุก byte**

5. **แยก probe จาก stress:** C0 มี fault ticks 20/1101 ทุก seed ซึ่งเป็น invalid/stale probes ที่ fixture ใส่ไว้เหมือนกันทั้ง C0/C1 (invalid_01 ที่ 11 s, stale_01 ที่ 12 s) ส่วนที่เกินใน C1 (+34 ถึง +61 ticks) มาจาก communication stress ที่ทำให้ acquisition sample age > 0.12 s แต่ C1 เปลี่ยน delay และ drop พร้อมกัน จึงบอกไม่ได้ว่าแต่ละผลมาจาก jitter หรือ drop เช่น stale fault อาจเกิดจาก delay 0.14–0.18 s อย่างเดียว หรือจาก drop หลาย message ติดกัน ต้องรัน delay-only และ drop-only แยก (one-factor) จึงจะอ้างเหตุได้

6. **Timestamp:** Bool/Float32/String/ColorRGBA ไม่มี Header จึงสรุปไม่ได้ว่า (ก) message ที่รับเป็นของ input ใด เมื่อมี drop ลำดับ publish/receive ไม่ตรงกัน, (ข) ค่าที่รับเป็นค่าเก่าหรือใหม่ ROS ตรวจได้เพียง receipt freshness sensor ที่ส่งค่าเก่าซ้ำยังดู fresh, (ค) source → physical latency ข้ามเครื่องเพราะ monotonic clock ของแต่ละเครื่องต่างกัน **แผน:** ใส่ input_id + sequence ใน log wrapper ของทุก node (หรือ custom message ที่มี Header ใน contract version ใหม่), sync เวลา ESP32 กับ PC ผ่าน micro-ROS time sync และบันทึก offset/round-trip เป็น error bound (เป้า ≤ 10 ms), วัด physical onset ของ servo ด้วยกล้อง 120 fps และ LED ด้วย photodiode โดยมี LED marker ผูก clock ของ video กับ log และรายงาน resolution กับ uncertainty

7. **Proposal:** `proposal.json` ผ่าน `check_proposal.py` (fields complete) และ `Lab07_Proposal_Minion.docx` (2–3 หน้า): AHM 1 ตัว (H_social), SO(2) CPG ของ nod, integration AHM → φ ของ CPG และ AHM → sleepy guard, 2 conditions (C0 baseline FSM, C1 AHM), 3 metrics, 3 repeats/condition, simulation AND real robot, human scenario 600 s, safety/calibration/controls/timeline ส่วนที่กลุ่มต้องยืนยันเองคือ hardware ที่มีจริง วันที่ และผู้เล่น/consent

Checklist: [x] graph7topics [x] units/QoS [x] completed policy checker [x] rawdata/config/SHA [x] pairedprotocol [x] misses [x] measurementlimitations [x] completeproposal

หมายเหตุ: ทำเฉพาะ offline track เพราะเครื่องนี้เป็น Windows ไม่มี Ubuntu 24.04 + ROS 2 Jazzy ยังไม่ได้รัน ROS 2 track และ ros2 bag
