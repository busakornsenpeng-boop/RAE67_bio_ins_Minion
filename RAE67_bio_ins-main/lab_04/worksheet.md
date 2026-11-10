# ใบงาน Lab04 (รายงาน 1–2 หน้า)

ชื่อ/รหัส Busakorn Senpeng 6752500029, Panida Kansrang 6752500169, Thanit Wongsripuek 6752500436, Baramee Sampaotong 6752500380 กลุ่ม Minionและผ่องเพื่อน วันที่ 10/11/2569 code version/hash `student_hormone.py` SHA-256 `a57ba93fe82d18313ebc1599f0556280557f7605fa7a8658200b58bd01186010`

## A. ก่อนรัน (10 นาที)

Block diagram: `results/figures/lab04_block_diagram.png`

S → P: stimulus กระตุ้นการสร้าง hormone ตาม P = k_prod·S
P/C/B → H: ผลสุทธิ P − C − B คูณ dt แล้วบวกเข้ากับ H_old โดย C = k_clear·H_old และ B = k_bind·R_old ใช้สถานะเก่าทั้งคู่
H → H (memory): H_new กลายเป็น H_old ของรอบถัดไป H จึงเก็บประวัติของ stimulus ที่ผ่านมา แม้ S กลับเป็น 0 แล้ว H ก็ยังค่อยๆ ลดลง ไม่หายทันที
H → R: receptor normalize และ clamp H_new ให้อยู่ใน [0, 1]
R → phi/stride: bounded mapping แปลง R เป็นคำสั่ง phi และ stride ที่อยู่ในขอบเขตเสมอ แล้วส่งต่อให้ CPG ของ Week 03

ค่า steady state ขณะ S = 1 (R = H เพราะ H_min = 0, H_max = 1): k_prod − (k_clear + k_bind)·H = 0

| Comparison | prediction ก่อนรัน | เหตุผลจาก term ในสมการ | ผลจริงหลังรัน |
|---|---|---|---|
| C2 vs C1 | C2 ขึ้นเร็วกว่าและชน H = 1 ส่วน recovery เท่ากับ C1 | k_prod เพิ่มจาก 0.8 เป็น 1.2 ทำให้ steady state = 1.2/0.8 = 1.5 > 1 จึงชน clamp แต่อัตรา removal ขาลงยังเป็น 0.6 + 0.2 = 0.8/s เท่าเดิม | สนับสนุน: rise 1.05 vs 2.70 s, upper clamp 0.290 vs 0.000, recovery 2.85 s เท่ากัน |
| C3 vs C1 | C3 ชน clamp และฟื้นช้ากว่า C1 ประมาณ 1.8 เท่า | k_clear ลดจาก 0.6 เป็น 0.25 ทำให้ removal ขาลงเหลือ 0.45/s เทียบกับ 0.8/s และ steady state = 0.8/0.45 = 1.78 | สนับสนุน: recovery 5.10 vs 2.85 s (×1.79), upper clamp 0.273 |
| C4 vs C1 | C4 ชน clamp และฟื้นช้ากว่า C1 ประมาณ 1.3 เท่า | k_bind = 0 ทำให้ removal เหลือ 0.6/s และ steady state = 0.8/0.6 = 1.33 | สนับสนุน: recovery 3.80 vs 2.85 s (×1.33), upper clamp 0.258 |
| C0 vs C1 | C0 ตอบสนองภายใน 1 sample และไม่มีความจำ | R = S โดยตรง ไม่มี state H | สนับสนุน: latency 0.05 s และ recovery 0.05 s ไม่มี H (N/A) |

เขียนว่า prediction ใดอาจเปลี่ยนเพราะ H มี upper bound=1: peak_H ของ C2, C3 และ C4 จะเท่ากันที่ 1 เพราะ steady state เกิน 1 ทั้งหมด จึงใช้ peak เปรียบเทียบไม่ได้ ต้องดู AUC, rise, recovery และ clamp fraction แทน ส่วน C1 มี steady state พอดี 1 จึงเข้าใกล้ 1 แบบ asymptotic โดยไม่ชน clamp (ผลจริง peak_H = 0.9997, upper clamp = 0)

## B. เติม code และคำนวณมือ (20 นาที)

1. **Source equation** (Homchanthanakul et al., 2019): Hc(t) = β·Hc(t−1) + Hg(t) − Hr(t), Hg = α·f(S), Hr = Σ γᵢ·Receptorᵢ
   **Proposed course model** (Euler): H_new = clamp(H_old + dt·(k_prod·S − k_clear·H_old − k_bind·R_old), 0, 1) โดย k_clear ไม่ได้ใช้แทน β โดยตรง
2. receptor(H = 0.2, H_min = 0.1, H_max = 0.5) = (0.2 − 0.1)/(0.5 − 0.1) = **0.25**
3. `step` เก็บ `h_old` และ `r_old = receptor(h_old)` ก่อน แล้วคำนวณ P, C, B จากค่าเก่าทั้งหมด จากนั้นอัปเดต H เพียงครั้งเดียว
4. คำนวณมือ (C1: k_prod 0.8, k_clear 0.6, k_bind 0.2, dt 0.05)
   - รอบแรก: H_old = 0, S = 1 → P = 0.8, C = 0, B = 0, H_raw = 0 + 0.05(0.8) = 0.04, H = R = 0.04, phi = 0.20 + 0.20(0.04) = 0.208, stride = 0.0554
   - รอบสอง: H_old = 0.04 → P = 0.8, C = 0.024, B = 0.008, H_raw = 0.04 + 0.05(0.768) = 0.0784, H = R = 0.0784, phi = 0.21568, stride = 0.055784
5. checker ผ่าน 13/13 case ตั้งแต่ครั้งแรก

| input_time_s | time_s | P | C | B | H_raw | H | R | phi |
|---:|---:|---:|---:|---:|---:|---:|---:|---:|
|10.00|10.05|0.8|0|0|0.0400|0.0400|0.0400|0.2080|
|10.05|10.10|0.8|0.024|0.008|0.0784|0.0784|0.0784|0.2157|

แถว CSV ที่เลือกสอดคล้องกับ time convention อย่างไร? ใช้แถว time_s = 10.05 และ 10.10 ใน `results/comparison/C1_step.csv` ซึ่งเป็น state *หลัง* update ที่ใช้ stimulus ของช่วงเริ่มต้นที่ input_time_s = 10.00 และ 10.05 ไม่ใช่สองแถวแรกของไฟล์ ค่าในแถวตรงกับการคำนวณมือทุกช่อง

## C. Controlled comparison (20 นาที)

ค่าจาก `results/comparison/metrics.json` (dt 0.05 s, duration 30 s, H0 = 0, stimulus step 10–20 s, mapping เดียวกัน)

| Condition | peak_H | AUC_H | latency s | rise s | recovery s | upper clamp fraction | phi TV rad |
|---|---|---|---|---|---|---|---|
|C0|N/A|N/A|0.05|0.00|0.05|N/A|0.400|
|C1|0.9997|9.9996|0.15|2.70|2.85|0.000|0.3998|
|C2|1.0000|10.6724|0.10|1.05|2.85|0.290|0.3999|
|C3|1.0000|11.3891|0.15|1.45|5.10|0.273|0.3979|
|C4|1.0000|10.7547|0.15|1.70|3.80|0.258|0.3995|

- C3 ฟื้นช้ากว่า C1 เท่าไร? 5.10 − 2.85 = **2.25 s ช้ากว่า (×1.79)** ตรงกับอัตราส่วน removal 0.8/0.45 = 1.78
- C2/C3 peak เท่ากันหรือไม่? **เท่ากันที่ 1.0** เพราะชน upper bound ทั้งคู่ metric ที่แยกได้คือ AUC (10.67 vs 11.39), recovery (2.85 vs 5.10 s), rise (1.05 vs 1.45 s) และ upper clamp fraction (0.290 vs 0.273)
- C0 มี H หรือไม่? **ไม่มี** C0 ส่ง R = S โดยตรง ช่อง H/raw/p/c/b ใน CSV จึงว่าง และ peak_H, AUC, clamp เป็น N/A ไม่ใช่ศูนย์ latency 0.05 s คือความละเอียดของการ log หนึ่ง sample ไม่ใช่ delay จริง
- TV ที่ต่ำกว่าเพียงอย่างเดียวพิสูจน์คุณภาพ gait ได้หรือไม่? **ไม่ได้** ทุก condition มี phi TV ประมาณ 0.40 rad เท่ากัน (0.3979–0.4000) ทั้งที่ C0 กระโดดทันที 0.20 → 0.40 rad ใน sample เดียว ส่วน C1 เปลี่ยนแบบค่อยเป็นค่อยไปตลอด 2.7 s TV วัดผลรวมของการเปลี่ยนแปลง ไม่ได้วัดความเร็วของการเปลี่ยน และ Lab04 ยังไม่ได้รัน gait จริง

## D. Extreme tests และ handoff (10 นาที)

| Profile (C1) | ผลที่ตรวจ |
|---|---|
| zero | H = 0 ตลอด 30 s, peak_H = 0, AUC = 0, phi คงที่ 0.20 rad |
| one | H เข้าใกล้ 1 (peak 0.99999999998) โดยไม่ชน clamp (upper clamp 0), AUC 28.75 |
| spike | มี on interval เพียง **1 แถว** (input_time_s 10.00 → time_s 10.05), H สูงสุด 0.04 แล้วลดลงต่อเนื่อง phi เปลี่ยนเพียง 0.008 rad |
| fault | แถว time_s 15.05: fault = 1, H = R = 0, raw/p/c/b = 0, phi = 0.20, stride = 0.055 ตรงตาม README แถว 15.10 เริ่มสะสมใหม่ที่ H = 0.04 เหมือนเพิ่งเปิด stimulus fault_count = 1 |

การ reset เมื่อ fault ทำให้ phi ตกจาก 0.397 เป็น 0.200 rad ภายใน 1 sample และ phi TV ของ fault profile เพิ่มเป็น 0.786 rad เกือบ 2 เท่าของ step ปกติ software bound ของ Lab รับประกันเพียงว่าคำสั่งอยู่ในช่วง [0.10, 0.45] rad แต่ไม่ได้จำกัดอัตราการเปลี่ยน ไม่ได้ตรวจ actuator limit (torque, ความเร็วข้อต่อ) และไม่ใช่ emergency stop ซึ่งต้องอยู่ในชั้นฮาร์ดแวร์แยกต่างหาก ระบบจริงควรเพิ่ม rate limit หรือ hold ค่าเดิมระหว่าง fault

ทดลองส่ง contract ของ Week03 (`../lab_03/results/week4_bridge/handoff_contract.json`) ด้วย `--contract` แล้ว runner ยอมรับ และ CSV ของ C1 **เหมือนกับรันด้วย contract เริ่มต้นทุก byte** ที่ R = 0 ได้ phi = 0.20 rad, stride = 0.055 m และที่ R ≈ 1 ได้ phi = 0.40 rad, stride = 0.065 m ตรงกับ segment m = 0 และ m = 1 ของ Week03

สิ่งที่ Lab04 ยังไม่ได้ทดสอบ: การป้อน phi/stride เข้า CPG จริงและการวัด body motion (speed, frequency, support), noise หรือ delay ของ sensor ที่สร้าง S, receptor range อื่นนอกจาก [0, 1], dt อื่น และฮาร์ดแวร์จริง ทั้งหมดอยู่ใน Week05

## E. สรุปส่งงาน (10 นาที)

(1) **กลไกและ provenance:** สมการต้นทางมาจาก Homchanthanakul et al. (2019) ซึ่งใช้ decay β, การสร้าง α·f(S) และการถูกจับโดย receptor ส่วน Lab ใช้ proposed course model แบบ Euler ที่แยก production (k_prod·S), clearance (k_clear·H_old) และ binding (k_bind·R_old) ออกจากกัน ทุก term ใช้สถานะเก่า แล้วอัปเดต H ครั้งเดียวต่อ step และ clamp ใน [0, 1] H จึงทำหน้าที่เป็นความจำที่ทำให้คำสั่ง phi/stride เปลี่ยนช้ากว่า stimulus ตัวเลขทั้งหมดเป็นค่าสำหรับการสอน ไม่ใช่ค่าจากบทความหรือฮาร์ดแวร์

(2) **Prediction ที่ข้อมูลสนับสนุน:** ทุก prediction ที่เขียนจากสมการก่อนรันได้รับการสนับสนุน C2 ขึ้นเร็วกว่า (rise 1.05 vs 2.70 s) แต่ recovery เท่าเดิม (2.85 s) เพราะอัตรา removal เท่าเดิม C3 และ C4 ฟื้นช้ากว่าตามอัตราส่วน removal (×1.79 และ ×1.33 เทียบกับที่คาดไว้ ×1.78 และ ×1.33) และ C2/C3/C4 มี peak_H = 1 เท่ากัน จึงต้องใช้ AUC (10.67/11.39/10.75) กับ recovery แยกความต่าง สิ่งที่ข้อมูลชี้เพิ่มเติมคือ phi TV แทบเท่ากันทุก condition (≈0.40 rad) จึงใช้ TV วัดความนุ่มนวลของคำสั่งไม่ได้

(3) **ข้อจำกัดและ Week05:** Lab04 จำลองเฉพาะ subsystem hormone → receptor → mapping แบบ deterministic ยังไม่ได้ทดสอบ gait การ reset เมื่อ fault ทำให้คำสั่งกระโดด 0.197 rad ใน 1 sample ซึ่งไม่ใช่ safety ระดับฮาร์ดแวร์ Week05 ต้องต่อ phi/stride เข้า CPG ของ Week03 แล้ววัด speed, frequency และ support ภายใต้ step และ fault เปรียบเทียบ C0 กับ C1 และทดสอบ noise ของ S

คำสั่งรันซ้ำ (จากโฟลเดอร์ `lab_04`):

```sh
python check_submission.py --student-file student_hormone.py
python lab04.py --controller-source student --condition all --profile step --output-dir results/comparison
python lab04.py --controller-source student --condition C1 --profile zero --output-dir results/zero
python lab04.py --controller-source student --condition C1 --profile one --output-dir results/one
python lab04.py --controller-source student --condition C1 --profile spike --output-dir results/spike
python lab04.py --controller-source student --condition C1 --profile fault --output-dir results/fault
python lab04.py --controller-source student --condition C1 --profile step --contract ../lab_03/results/week4_bridge/handoff_contract.json --output-dir results/week3_contract
```

Checklist: ☑ code ☑ comparison CSV 5 files ☑ extreme CSV 4 files ☑ config ☑ plots ☑ metrics ☑ report ☑ คำสั่งรันซ้ำ
