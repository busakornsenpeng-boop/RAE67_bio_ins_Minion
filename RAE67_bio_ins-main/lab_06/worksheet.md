# ใบงาน Lab06 — Student

**แบบจำลองทั้งหมดใน runner เป็น proposed course model** โปรแกรมเป็น simulation ไม่ใช่ข้อมูลจาก robot และไม่มี learning

กลุ่ม Minionและผ่องเพื่อน: Busakorn Senpeng 6752500029, Panida Kansrang 6752500169, Thanit Wongsripuek 6752500436, Baramee Sampaotong 6752500380 วันที่ 10/11/2569 contract ที่เลือก: **baseline** (`python check_submission.py`)

## 1. Prediction ก่อนรัน

- C0 ไม่มี debounce และ hysteresis ปุ่มที่ bounce จะถูกนับหลายครั้ง และระยะ approach ที่มี noise ±0.035 m รอบ threshold 0.80 m จะเข้าออกซ้ำ ทำให้ C0 มี recognized events, state transitions และ command ที่ถูก preempt ก่อน apply มากกว่า C1 ซึ่งควรมีประมาณ 1 recognized event ต่อ source
- C1 มี debounce 0.06 s สำหรับ button/touch latency จึงควรเท่ากับ C0 + 0.06 s โดยประมาณ คาด both latency ≈ debounce (0 หรือ 0.06) + scheduler 0.04 + 1 tick 0.02 ≈ 0.06–0.12 s ต่ำกว่า 1 s ทั้งสอง condition ส่วน slew limit จำกัดระยะการเคลื่อน ไม่ได้เลื่อนเวลาที่เริ่มเคลื่อน
- Input ที่ควร**ไม่มี onset ใหม่**: touch_conflict (state เป็น afraid แล้ว policy จึงไม่รับ touch) และ fault ที่เกิดขณะ output เป็น neutral normal อยู่แล้ว (invalid_01, stale_01 → no_state_change) ส่วน primary input อีก 7 ตัวควรได้ทั้ง motion และ RGB onset
- ช่วงเงียบ 12–18 s: normal นาน 6 s → idle → sleepy แล้ว button_02 ที่ 18 s → interested

## 2. FSM

แผนภาพ: `results/figures/lab06_fsm.png` (5 state พร้อม entry/exit guard, timeout 2 s และ idle 6 s)

- Entry guard ของ afraid: near (ระยะ ≤ 0.20 m ใน C1) จากทุก state ออกได้ทางเดียวคือ clear (≥ 0.28 m) หรือ fault
- Timeout/recovery: interested และ happy กลับ normal หลังไม่มี event 2 s ส่วน normal ไม่มี interaction 6 s → sleepy
- Fault (stale > 0.12 s, invalid, nonfinite) → normal จากทุก state เป็น neutral fallback

## 3. ตาราง 5 state × 8 event

| current \ event | fault | near | clear | touch | button | approach | timeout | idle |
|---|---|---|---|---|---|---|---|---|
| normal | normal* | afraid | normal* | happy | interested | interested | hold | sleepy |
| interested | normal | afraid | normal | happy | interested* | interested* | normal | hold |
| happy | normal | afraid | normal | happy* | interested | interested | normal | hold |
| afraid | normal | afraid* | normal | hold | hold | hold | hold | hold |
| sleepy | normal | afraid | normal | happy | interested | interested | hold | hold |

hold = policy คืน None (engine บันทึก `policy_held`); * = next state เท่ากับ state เดิม engine บันทึก `no_state_change` และไม่สร้าง expression ใหม่ priority เมื่อหลาย event recognized ใน tick เดียวกัน: fault > near > clear > touch > button > approach > timeout > idle เช่น near + touch ใน tick เดียวกันจะใช้ near และ touch ถูกบันทึกเป็น `discarded_priority`

## 4. เติม TODO และรัน checker

เติม touch → happy (ยกเว้น afraid ซึ่ง guard ไว้ก่อนแล้ว), timeout ของ interested/happy → normal และ idle ของ normal → sleepy ผล: `Checked 40 transition pairs; contract=baseline; failures=0`

**Failed case ที่ช่วยแก้ logic:** เมื่อรัน checker กับ starter เดิม พบ failures = 7 เช่น `sleepy + touch: returned None` และ `interested + timeout: returned None` จุดที่ต้องระวังคือลำดับของเงื่อนไข: ต้องตรวจ touch **หลัง** afraid guard ถ้าวางไว้ก่อน afraid + touch จะคืน happy ซึ่งผิด contract (afraid ต้อง hold)

## 5. C0/C1 ด้วย 3 paired seeds

## ตารางบันทึกของนักศึกษา

ค่าจาก `results/completed/metrics.json` (3 seeds เป็น synthetic replications ไม่ใช่ physical trials อิสระ)

| Condition/seed | All external inputs | Both onset | ≤1s both observed | No both onset/status | Max observed latency [s] | State transitions |
|---|---:|---:|---:|---|---:|---:|
| C0/41 | 10 | 7 | 7 | 3: discarded_priority 1 (touch_conflict), no_state_change 2 (invalid, stale) | 0.24 | 67 |
| C1/41 | 10 | 7 | 7 | 3: policy_held 1 (touch_conflict), no_state_change 2 | 0.14 | 9 |
| C0/42 | 10 | 6 | 6 | 4: filtered_not_recognized 1 (clear_01), preempted_before_apply 1 (touch_conflict), no_state_change 2 | 0.18 | 72 |
| C1/42 | 10 | 7 | 7 | 3: policy_held 1, no_state_change 2 | 0.14 | 10 |
| C0/43 | 10 | 6 | 6 | 4: discarded_priority 1, no_state_change 3 (invalid, stale, **clear_02**) | 0.04 | 63 |
| C1/43 | 10 | 7 | 7 | 3: policy_held 1, no_state_change 2 | 0.14 | 9 |

Primary 7 inputs: C1 ได้ both onset ≤ 1 s ครบ 7/7 ทุก seed ส่วน C0 ได้ 7/7, 6/7 และ 6/7 (seed 42 พลาด clear_01, seed 43 พลาด clear_02)

## 6. ตรวจหนึ่ง event ตั้งแต่ source ถึง onset

| Input ID | Source s | Recognized s | Decision s | Actual apply s | Motion onset s | RGB onset s | Both latency s | Status/reason |
|---|---:|---:|---:|---:|---:|---:|---:|---|
| button_01 (C1/41) | 1.00 | 1.10 | 1.10 | 1.14 | 1.14 | 1.14 | 0.14 | responded |
| touch_01 (C1/41) | 3.00 | 3.06 | 3.06 | 3.10 | 3.10 | 3.10 | 0.10 | responded |

คำนวณเองจาก `inputs_r01.csv` และ `raw_C1_r01.csv`: ปุ่มกดที่ 1.00 แล้ว bounce เป็น 0 ที่ 1.02 และกลับเป็น 1 ตั้งแต่ 1.04 debounce ต้องคงค่า 0.06 s จึง recognized ที่ 1.04 + 0.06 = 1.10 จากนั้น scheduler delay 0.04 s ทำให้ apply ที่ 1.14 ซึ่ง tilt เริ่มขยับ (0.016 rad = 0.8 rad/s × 0.02 s ตาม slew limit) และ RGB เปลี่ยนเป็น 255 ใน tick เดียวกัน ได้ latency = 0.04 (bounce) + 0.06 (debounce) + 0.04 (scheduler) = **0.14 s ตรงกับ CSV** ส่วน touch ไม่มี bounce จึงได้ 0.06 + 0.04 = 0.10 s

## 7. Conflict, fallback, sleepy และ recovery

- **near + touch (7.00 s):** C1 recognized near ที่ 7.00 ก่อน touch ซึ่งผ่าน debounce ที่ 7.06 เมื่อนั้น state เป็น afraid แล้ว touch จึงเป็น `policy_held` (afraid guard) ใน C0 ทั้งสองอยู่ใน tick เดียวกัน touch จึงเป็น `discarded_priority` (seed 41, 43) หรือ `preempted_before_apply` (seed 42)
- **stale/invalid fallback:** invalid_01 (11 s) และ stale_01 (12 s) เกิดตอน state เป็น normal อยู่แล้ว จึงเป็น `no_state_change` ไม่มี onset ซึ่งถูกต้องตาม prediction
- **quiet-to-sleepy:** internal idle ที่ 15.00 s (6 s หลัง clear_02 ที่ 9.00) → sleepy มี onset ที่ 15.04
- **recovery button:** button_02 ที่ 18.00 → interested มี onset ที่ 18.10 (C1) แล้ว timeout ที่ 20.06 → normal
- ความต่างของ status: `discarded_priority` = ถูกตัดเพราะมี event ลำดับสูงกว่าใน tick เดียวกัน; `policy_held` = policy คืน None; `no_state_change` = next state เท่ากับ state เดิมจึงไม่ต้องมี expression ใหม่; `preempted_before_apply` = command ถูก epoch ใหม่แทนก่อน apply; `censored_end` = trial จบก่อนรอ onset ได้ (ไม่พบในการทดลองนี้)

## 8. Replay

รัน `--replay results/completed/inputs_r01.csv` แล้วเทียบ: `raw_C1_r01.csv` และ `events_C1_r01.csv` **เหมือนกันทุก byte** (ตรวจด้วย `cmp`) เพราะ policy, config และ source ไม่เปลี่ยน seed ใน replay metrics เป็น N/A ตามที่ README ระบุ

## 9. สรุปสำหรับรายงาน

- C1 ลด state transitions จาก 63–72 เหลือ 9–10 ครั้ง (ลด ~86%) และลด preempted commands จาก 34–46 เหลือ 0–1 แลกกับ latency ของ button/touch ที่เพิ่มจาก 0.04 s เป็น 0.10–0.14 s
- **Negative result:** prediction เรื่อง latency ผิดบางส่วน button_01 ของ C1 ได้ 0.14 s เกินช่วงที่คาด (0.06–0.12 s) เพราะ bounce ทำให้ debounce เริ่มนับใหม่ อีกข้อคือ C0 ที่ไม่มี filter กลับพลาด primary input (clear) ใน 2 จาก 3 seeds เพราะ noise ทำให้ state กลับเป็น normal ก่อนถึงเวลา clear จริง จึงไม่มี crossing ใหม่ให้ตอบสนอง
- **Hardware limitation:** latency ทั้งหมดเป็น simulated command onset ไม่ใช่ physical latency หุ่นจริงต้องวัดด้วย clock, กล้อง หรือ photodiode ที่สอบเทียบแล้ว แยก motion กับ RGB พร้อมระบุ resolution และ uncertainty และ joint เป็นเพียงแกนเชิงความหมาย ไม่มี linkage, torque หรือ collision

คำสั่งรันซ้ำ (จากโฟลเดอร์ `lab_06`):

```sh
python check_submission.py
python lab06.py --policy student --condition all --repeats 3 --output-dir results/completed
python lab06.py --policy student --condition C1 --repeats 1 --replay results/completed/inputs_r01.csv --output-dir results/replay
```

ก่อนส่งตรวจ: ทุก state/movement/input ครบ; bounds/slewไม่ถูกปิด; latencyใช้ออนเซ็ตจริงของmodel; missingไม่แทน0; ระบุ3seedsเป็นsynthetic replications; ส่งpolicy/config/raw/events/metrics; student filesไม่มีเฉลยหรือข้อมูลผู้สอน
