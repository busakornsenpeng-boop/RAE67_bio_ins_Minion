# ใบงาน Lab 05 — Prediction, timing, evidence

ฉบับพิมพ์: [Word](Lab05_Student_Worksheet_TH.docx) / [PDF](Lab05_Student_Worksheet_TH.pdf)

ชื่อ/รหัส Busakorn Senpeng 6752500029, Panida Kansrang 6752500169, Thanit Wongsripuek 6752500436, Baramee Sampaotong 6752500380 กลุ่ม Minionและผ่องเพื่อน วันที่ 10/11/2569

**ขอบเขต:** main runner เป็น proposed course model ใช้ scheduled stimulus และ kinematic body; contact demo ใช้ synthetic signals ไม่ใช่ ALCS terrain experiment ของ paper

Python 3.13.15, รันจากโฟลเดอร์ `lab_05` ด้วยคำสั่งในส่วนที่ 6

## 1. ก่อนรัน: hypothesis และ architecture (10 นาที)

1. แผนภาพ `results/figures/lab05_architecture.png`: S (schedule) → controller (B0/B1/B2/B3) → R_used → bounded map → φ/stride → SO(2) CPG → tripod decoder → kinematic body **จุดที่ไม่มี independent sensory feedback** คือเส้นย้อนกลับจาก body/contact ไปยัง S หรือ H เพราะ S เป็นตารางเวลาที่กำหนดล่วงหน้า และ contact ได้จาก sign(o₂) ของ decoder เอง ไม่ใช่ sensor วัดพื้นจริง
2. **Hypothesis (B1 vs B2, เขียนก่อนรัน):** B2 มี command latency มากกว่า B1 (0.15 s vs 0 s) และ command recovery ยาวกว่า (ประมาณ 2.85 s vs 0 s) มี maxΔφ/step เล็กกว่ามาก (ประมาณ 0.008 vs 0.20 rad) แต่ TV ของ φ ใกล้เคียงกันที่ประมาณ 0.40 rad ความถี่และระยะเดินช่วง on ของ B2 จะต่ำกว่า B1 เพราะ φ ค่อยๆ ขึ้น ส่วนระยะช่วง off ของ B2 จะมากกว่า B1 เพราะ H ยังค้างอยู่หลัง 20 s
3. B3 ตัด **binding term** (k_bind·R = 0) ออกจาก removal ของ B2 แต่ยังคง **hormone state H เป็น memory** และยังมี clearance k_clear·H จึงคาดว่า B3 ขึ้นเร็วกว่าและชน H = 1 (steady state 0.8/0.6 = 1.33) และฟื้นตัวช้ากว่า B2 ประมาณ 1.33 เท่า ทั้งนี้ไม่ได้ตัด prior MI_H หรือ memory ตามแบบใน paper
4. ตาราง provenance

| ประเภท | รายการ |
|---|---|
| Source equation | release = CI/(1 + exp(−SIF)), CI = 1 (Ngamkajornwiwat et al., 2020, Eq. 1) |
| Proposed course equation | H(t_(k+1)) = clamp(H + dt(k_prod·S − k_clear·H − k_bind·R), 0, 1) และ φ = clamp(0.20 + 0.20·R_used, 0.10, 0.45) |
| Expected measurement | segment frequency และ distance ของ kinematic body ช่วง pre/on/off |
| Metric ที่วัดไม่ได้ | Stability, Harmony และ Displacement ตามนิยามของ paper (N/A เพราะไม่มี CoM, แรง หรือ dynamics) |

## 2. รัน comparison และตรวจเวลา (15 นาที)

ค่าจาก `results/comparison/B2_intervals.csv` และ `B2_states.csv`

| command/input time | state end time | H_start | H_end | R_used | φที่ใช้ | stride_half ที่ใช้ |
|---|---|---|---|---|---|---|
| 10.00 | 10.05 | 0.0000 | 0.0400 | 0.0000 | 0.200 rad | 0.0550 m |
| 10.05 | 10.10 | 0.0400 | 0.0784 | 0.0400 | 0.208 rad | 0.0554 m |
| 20.00 | 20.05 | 0.9997 | 0.9597 | 0.9997 | 0.39994 rad | 0.06500 m |

อธิบายจาก CSV ว่าเหตุใด B1 command latency กับ state latency จึงต่างกัน: B1 เป็น direct path ที่ใช้ S_k ทันที ใน `B1_intervals.csv` แถว command_time 10.00 จึงมี r_used = 1.0, φ = 0.40 และ stride = 0.065 (command latency = 0) แต่ `B1_states.csv` บันทึก R เป็น state *หลัง* update แถว state_time 10.00 จึงยังเป็น r = 0 และ r = 1 ปรากฏครั้งแรกที่ 10.05 (state latency = 0.05 s) ส่วนต่าง 0.05 s นี้คือ sampling offset ของการ log ไม่ใช่ hardware latency

ตรวจ warmup/initialstates ใน config: ทุก condition เริ่ม CPG เหมือนกันหรือไม่? **เหมือนกัน** หลักฐาน: `config.json` → `initial_states` ของ B0, B1, B2 และ B3 มี o₁ = −0.026965, o₂ = −0.560863 และ body_x = 0 เท่ากันทุกตัว หลัง warmup 10 s ด้วย φ = 0.20 และ distance ช่วง pre ของทั้ง 4 condition เท่ากันที่ 0.776320 m

## 3. เปรียบเทียบผลและข้อจำกัด (15 นาที)

ค่าจาก `results/comparison/metrics.csv` และ `segment_metrics.csv` (step profile, mapping both)

| condition | TV φ(rad) | maxΔφ/step(rad) | TV stride(m) | cmd latency(s) | cmd recovery(s) | on frequency(Hz) | on distance(m) |
|---|---|---|---|---|---|---|---|
| B0 | 0.000 | 0.000 | 0.0000 | N/A | N/A | 0.625 | 0.779 |
| B1 | 0.400 | 0.200 | 0.0200 | 0.00 | 0.00 | 1.268 | 1.825 |
| B2 | 0.3998 | 0.0080 | 0.0200 | 0.15 | 2.85 | 1.203 | 1.706 |
| B3 | 0.3995 | 0.0080 | 0.0200 | 0.15 | 3.80 | 1.226 | 1.727 |

B0 ไม่มี crossing ของ R_used จึงเป็น N/A ไม่ใช่ 0

1. ผลข้อใดสนับสนุน hypothesis และผลข้อใดไม่สนับสนุน? **สนับสนุนทุกข้อ** ได้แก่ latency 0.15 vs 0 s, recovery 2.85 vs 0 s, maxΔφ/step 0.008 vs 0.200 rad (B2 เล็กกว่า 25 เท่า), TV เท่ากัน (0.3998 vs 0.400 rad), on frequency 1.203 vs 1.268 Hz, on distance 1.706 vs 1.825 m และ off distance 0.900 vs 0.776 m **ข้อค้นพบแบบ null result:** ระยะรวม 30 s ของ B1 (3.378 m) กับ B2 (3.383 m) ต่างกันเพียง 0.005 m (0.15%) hormone ไม่ได้เปลี่ยนระยะรวม เพียงย้ายระยะจากช่วง on ไปช่วง off
2. การลด TV ยืนยันว่าหุ่นยนต์ไม่ล้มหรือใช้พลังงานน้อยลงหรือไม่? **ไม่ได้** และในการทดลองนี้ TV ไม่ได้ลดด้วย (0.3998 vs 0.400 rad) สิ่งที่ลดคือ maxΔφ/step ซึ่งก็ยังเป็นเพียงความนุ่มนวลของคำสั่ง ข้อมูลที่ขาดคือ body dynamics (มวล inertia), แรงสัมผัสพื้นและแรงเสียดทาน, ตำแหน่ง CoM เทียบกับ support polygon, joint torque และพลังงานที่ใช้
3. Bounds เหมือนกันทำให้ commandamplitude เท่ากันหรือไม่? **ไม่เท่ากัน** peak R_used ของ B1 = 1.000 แต่ B2 = 0.9997 และค่าเฉลี่ย φ ช่วง on ต่างกัน (B1 0.400, B3 0.382, B2 0.375 rad) เพราะ B2/B3 ใช้เวลาไต่ขึ้น (rise 2.70 และ 1.70 s) ไม่มี condition ใดชน bounds เลย (phi/stride clamp count = 0) bounds จึงจำกัดเพียงค่าสูงสุดที่เป็นไปได้ ไม่ได้ทำให้ amplitude เท่ากัน
4. ทำไมไม่ใช้ frequency เฉลี่ย 30 s แทน pre/on/off? เพราะค่าเฉลี่ย 30 s ผสมสามช่วงที่ φ ต่างกัน (0.625 → 1.27 → 0.625 Hz สำหรับ B1) และกลบ transient ที่ต่างกันระหว่าง condition เช่น B2 ช่วง off ยังมีความถี่ 0.698 Hz ขณะที่ B1 กลับเป็น 0.625 Hz ทันที ค่าเฉลี่ยรวมจะซ่อนความต่างนี้ไว้

## 4. เปลี่ยนหนึ่งปัจจัยและตรวจ fault (20 นาที)

เลือก **phi-only** (รัน stride-only ประกอบด้วย)

สิ่งที่เปลี่ยน: φ ผ่าน R_used ตาม mapping เดิม สิ่งที่คงที่: stride_half = 0.055 m (ตรวจจาก `phi_only/B2_intervals.csv` มีค่า stride เพียงค่าเดียวคือ 0.055) รวมถึง gains, bounds, schedule และ warmup metric หลัก: on frequency และ on distance prediction: frequency ช่วง on เท่ากับ both mapping ส่วน distance ลดลงตามอัตราส่วน stride 0.055/0.065 = 0.846

ผลและข้อจำกัด causalclaim: frequency ช่วง on ของ phi-only เท่ากับ both mapping ทุกตัว (B1 1.268, B2 1.203 Hz) ส่วน on distance ลดลงเหลือ B1 1.549 m (×0.849 ของ 1.825) และ B2 1.467 m (×0.860 ของ 1.706) ตรงกับ prediction ฝั่ง stride-only มี frequency คงที่ 0.625 Hz และ on distance ของ B1 เพิ่มเป็น 0.915 m (×1.17 เทียบ B0 0.779) ผลของสองปัจจัยคูณกันได้ประมาณผลรวม (1.99 × 1.17 = 2.33 เทียบกับ both 1.825/0.779 = 2.34) ข้อจำกัด: เป็น deterministic run ครั้งเดียว ไม่ใช่ independent trials ใช้ได้เฉพาะ kinematic model และ mapping เชิงเส้นนี้ และสรุปได้เพียงว่าใน model นี้ φ ควบคุม frequency ส่วน stride ควบคุมระยะต่อรอบ ไม่ได้บอกผลบนหุ่นยนต์จริง

รัน faultprofile เปิด input15.00 แล้วบันทึก (`results/fault/B2_intervals.csv`):

| field | B2 value |
|---|---|
| fault at command15.00 | 1 (stimulus = NaN) |
| H_used / R_used | 0 / 0 (H_state_start ก่อน reset = 0.9831) |
| φ / stride ที่ใช้ | 0.200 rad / 0.055 m |
| H state15.05 | 0.0000 |
| H state15.10 | 0.0400 |

อธิบายว่า fallback ทำก่อนหรือหลัง command และเหตุใด policy นี้ยังไม่รับรอง hardware safety: fallback ทำ **ก่อน** command ใน interval เดียวกัน [15.00, 15.05) คือตรวจพบ NaN แล้วตั้ง R_used = 0, φ = 0.20, stride = 0.055 ก่อน CPG update และ reset H เป็น 0 แต่คำสั่ง φ ตกจาก 0.396 เป็น 0.200 rad ภายใน 1 step ทำให้ maxΔφ/step ของ B2 ใน fault profile เพิ่มเป็น 0.196 rad (จาก 0.008 ใน step ปกติ) เกือบเท่า B1 (0.200) และ TV เพิ่มเป็น 0.786 rad policy นี้ไม่มี rate limit ไม่ได้ตรวจขีดจำกัดของ actuator และไม่ใช่ emergency stop จึงเป็นเพียง software fallback ใน model

## 5. Contact mini-lab (20 นาที)

ค่าจาก `results/contact/metrics.json` (window 50 samples, valid ตั้งแต่ 2.45 s)

| case | SI ของแต่ละขาโดยทั่วไป | SIF | absoluteerror | สิ่งที่อนุมานไม่ได้ |
|---|---|---|---|---|
| matched | 1.0 ทุกขา | 0.00 | 0.000 | ไม่ได้บอกว่า expected ถูกต้องเมื่อเทียบกับพื้นจริง |
| gain-offset | 1.0 ทุกขา | 0.00 | 0.163 | amplitude และ offset ที่ผิด (correlation มองไม่เห็น) |
| mixed-delay | 1.0 สามขา, ≈0.00 สามขา (delay 0.25 s = 90° ที่ 1 Hz) | 0.50 (release 0.62) | 0.226 | ขนาดของ delay เป็นวินาที และสาเหตุของ delay |
| all-inverted | −1.0 ทุกขา | 0.00 (release 0.50) | 0.631 | ว่าระบบผิดทั้งหมด เพราะ SIF ไม่ต่างจาก matched |
| constant | N/A ทุกขา (zero variance) | N/A | 0.000 | ความสัมพันธ์ใดๆ เพราะไม่มีการเปลี่ยนแปลงให้เทียบ |

1. ทำไม gain-offset ให้ correlation สูงแม้ error ไม่เป็น 0? เพราะ Pearson ลบค่าเฉลี่ยและหารด้วย SD ก่อนเทียบ จึงไม่ไวต่อการแปลงเชิงเส้น actual = 0.2 + 0.5·expected มีรูปคลื่นและจังหวะเหมือน expected ทุกประการ SI จึงเป็น 1 แต่ค่าจริงต่างกัน mean absolute error จึงเท่ากับ 0.163
2. เมื่อทุกขา inverted ทำไม SIF จึงยังเป็น 0? SIF คือ population SD ของ SI ระหว่าง 6 ขา เมื่อทุกขามี SI = −1 เท่ากัน SD จึงเป็น 0 SIF วัดความ*ไม่สม่ำเสมอระหว่างขา* ไม่ได้วัดว่าผิดมากน้อยเพียงใด release ที่ SIF = 0 เท่ากับ 1/(1 + e⁰) = **0.5**
3. constant ควรเป็น SI0 หรือ N/A? **N/A** เพราะ variance เป็น 0 ทำให้ตัวหารของ Pearson เป็น 0 และ correlation ไม่มีนิยาม CSV ระบุ validity = `zero_variance` การแทนด้วย 0 จะสื่อผิดว่าไม่สัมพันธ์กัน
4. fault หนึ่ง sample มีผลต่อ rollingwindows กี่ rows ไม่นับ startup49rows? **50 rows** (sample time 15.00 ถึง 17.45 s) คือทุก window ขนาด 50 samples ที่มี sample 15.00 อยู่ ตรวจจาก `fault.csv`: unavailable SIF ทั้งหมด 99 rows − startup 49 rows = 50 rows และ LF_validity = `nonfinite_window`
5. pairedPearson ของ demo มีสถานะ source/proposed ใด? ต่อ Cg/MI แล้วหรือยัง? เป็น **proposed course definition** ไม่ใช่ Eq. (3) ของ paper ตามที่พิมพ์ และ**ยังไม่ได้ต่อ** C_g/HR/MI_H release คำนวณได้ แต่ไม่ได้ส่งกลับเข้า main runner

## 6. ส่งงานและ exit ticket (10 นาที)

ไฟล์ส่ง: `results/comparison`, `results/phi_only`, `results/stride_only`, `results/fault`, `results/zero|one|spike`, `results/contact` (แต่ละโฟลเดอร์มี config.json, manifest.json พร้อม SHA-256, raw CSV และ source_snapshot), กราฟใน `results/figures/` และรายงาน `Lab05_Report_Minion.docx`

คำสั่งรันซ้ำ (Python 3.13.15, จากโฟลเดอร์ `lab_05`):

```sh
python lab05.py --condition all --profile step --mapping both --output-dir results/comparison
python contact_demo.py --output-dir results/contact
python lab05.py --mapping phi-only --output-dir results/phi_only
python lab05.py --mapping stride-only --output-dir results/stride_only
python lab05.py --condition B2 --profile zero --output-dir results/zero
python lab05.py --condition B2 --profile one --output-dir results/one
python lab05.py --condition B2 --profile spike --output-dir results/spike
python lab05.py --condition all --profile fault --output-dir results/fault
```

**Exit:**
- Claim ที่ข้อมูลรองรับ: ใน kinematic model นี้ hormone path (B2) ทำให้คำสั่ง φ เปลี่ยนช้าลง maxΔφ/step ลดจาก 0.200 เป็น 0.008 rad แลกกับ command latency 0.15 s และ recovery 2.85 s โดยระยะรวม 30 s แทบไม่เปลี่ยน (3.378 vs 3.383 m)
- Claim ที่ยังรองรับไม่ได้: hormone path ทำให้หุ่นยนต์เดินมั่นคงขึ้นหรือประหยัดพลังงานขึ้น ต้องมี body dynamics, แรงสัมผัสพื้นจริง, CoM/support polygon และพลังงาน (Stability/Harmony ตามนิยามของ paper) และต้องมี sensory feedback จริงแทน scheduled stimulus
