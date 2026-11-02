# K6 lab results

## Part 0
- Step 0.1, http_req_duration: avg=277.8ms min=277.8ms med=277.8ms max=277.8ms p(90)=277.8ms p(95)=277.8ms
  - ทุกค่าเท่ากันเพราะไม่ได้ใส่ option จึงรัน 1 VU แค่ 1 iteration มีเพียง 1 request (checks_succeeded 100.00% 1 out of 1, http_reqs = 1, iterations = 1)
  - iteration_duration = 1.92s ซึ่งนานกว่า sleep(1) บวกเวลาของ request เพราะ request แรกต้องเปิด connection (DNS, TCP, TLS) ก่อน และเวลาส่วนนี้ไม่ได้นับรวมใน http_req_duration
- Step 0.2, requests, rate and p(95), and why the rate stays below 5: 78 requests, 3.65 requests/s (http_reqs 3.653307/s), p(95) = 325.2ms
  - VU หนึ่งตัวส่ง request แค่ 1 ครั้งต่อ 1 iteration และ iteration หนึ่งใช้เวลาเฉลี่ย 1.35s (รอ response ประมาณ 0.3s แล้ว sleep(1) อีก 1s) แต่ละ VU จึงส่งได้ประมาณ 1 / 1.35 ≈ 0.74 request/s และ 5 VUs รวมกันได้ประมาณ 5 × 0.74 ≈ 3.7 request/s ซึ่งตรงกับที่วัดได้
  - ทุก iteration มี sleep(1) อยู่แล้ว iteration หนึ่งจึงใช้เวลามากกว่า 1 วินาทีเสมอ rate จึงไม่มีทางถึง 5 request/s นอกจากนี้ iteration แรกของแต่ละ VU ต้องเปิด connection ใหม่จึงนานกว่าปกติ (iteration_duration max = 2.39s)
- Step 0.3, a fixed number of iterations: iterations = 20 พอดี (20 requests, checks 100.00% 20 out of 20, p(95) = 260.4ms) และจบใน 5.6s ทันทีที่ iteration ที่ 20 เสร็จ โดย 5 VUs แบ่ง 20 iterations กันทำ (shared iterations) เฉลี่ยตัวละ 4 รอบ
  - --duration ให้ load ต่อเนื่องตลอดช่วงเวลาที่กำหนด จำนวน request จึงขึ้นกับความเร็วของระบบ ส่วน --iterations ให้ปริมาณงานที่แน่นอน เวลาที่ใช้จึงขึ้นกับความเร็วของระบบแทน
- Step 0.4, testing an API: checks ผ่านครบทั้ง 3 ข้อ (100.00% 3 out of 3) และ http_req_duration = 378.14ms
  - check ข้อที่ 3 อ่าน pizza.ingredients ใน JSON แล้วตรวจว่าทุก ingredient เป็น vegetarian จริง ถ้าดูแค่ status 200 จะไม่รู้ว่า API ตอบเร็วแต่ตอบผิดหรือเปล่า
- Step 0.5, a request that must be refused: API ตอบ 401 ตามที่ควร check "request without a token is refused" ผ่าน (100.00% 1 out of 1) และ http_req_failed = 0.00% 0 out of 1
  - http.expectedStatuses(401) บอก k6 ว่า 401 คือคำตอบที่ถูกต้องของ request นี้ ถ้าไม่ใส่ k6 จะนับ 401 เป็น failed request เพราะ status ตั้งแต่ 400 ขึ้นไปถือว่า fail ทั้งหมด
- Step 0.6, exit code of the normal run and of the run with P95=300: รันปกติได้ exit code 0 และรันด้วย P95=300 ได้ exit code 99
  - รันปกติ: ผ่านครบทั้ง 3 rules คือ checks rate=100.00%, http_req_duration p(95)=374.05ms (< 1500) และ http_req_failed rate=0.00% (68 requests, 2.19 requests/s)
  - รันด้วย `-e P95=300`: rule `p(95)<300` ไม่ผ่าน เพราะวัดได้ p(95)=391.85ms และ k6 แจ้งว่า "thresholds on metrics 'http_req_duration' have been crossed" แม้ checks จะผ่าน 100% แต่ test ทั้งชุดก็ fail เพราะ threshold
  - pipeline อย่าง GitHub Actions อ่าน exit code 99 แล้วสั่งให้ build fail ได้เลย
- Step 0.7, rules that passed, p(95), p(99), exit code after your change: ผ่านครบทั้ง 3 rules ได้ p(95) = 384.45ms (< 1000), p(99) = 389.7ms (< 1500), http_req_failed = 0.00% (< 1%) และ exit code 0 จากนั้นเปลี่ยน requirement จาก "95% ภายใน 1 วินาที" เป็น "95% ภายใน 0.3 วินาที" (รันด้วย `-e P95=300`) rule `p(95)<300` จึงไม่ผ่าน (วัดได้ p(95) = 371.09ms) และได้ exit code 99
  - script ตั้ง `vus: 3`, `duration: '30s'` และใส่ thresholds ตาม requirement พอดี คือ `http_req_duration: ['p(95)<1000', 'p(99)<1500']` กับ `http_req_failed: ['rate<0.01']` ไม่ได้ใส่ threshold ของ checks เพราะ requirement ไม่ได้พูดถึง และเพิ่ม `summaryTrendStats` ให้ summary แสดง p(99) ด้วย
- Step 0.8, what was wrong, and how the checks showed it: request ส่งชื่อเป็นตัวพิมพ์เล็ก (`'pepperoni'`, `'knife'`) แต่ QuickPizza ใช้ชื่อ `"Pepperoni"` และ `"Knife"` (explore.js แสดง tools เป็น `["Knife","Pizza cutter","Scissors"]` และ topping ชื่อ `"Pepperoni"`) API เทียบชื่อแบบตรงตัว จึงหาของที่ต้องตัดออกไม่เจอและไม่ได้ตัดอะไรออกเลย แต่ก็ไม่แจ้ง error ใด ๆ เพราะ status ยังเป็น 200 ทุก request และ http_req_failed = 0.00%
  - ก่อนแก้ (5 VUs 20s, 75 requests): checks_succeeded 87.55% (197 out of 225) โดย "no pepperoni" ผ่าน 86% (✓ 65 / ✗ 10) และ "no knife needed" ผ่าน 76% (✓ 57 / ✗ 18) แปลว่ามีพิซซ่า 10 ถาดที่ใส่ pepperoni และ 18 ถาดที่ต้องใช้มีดตัด ซึ่งลูกค้าไม่ได้ขอ
  - แก้เป็น `excludedIngredients: ['Pepperoni']` และ `excludedTools: ['Knife']` แล้วรันใหม่ (5 VUs 20s, 75 requests) checks ผ่าน 100.00% (225 out of 225)
  - ทำไม load ถึงจับได้: API สุ่มพิซซ่าใหม่ทุกครั้ง และ request ส่วนใหญ่ (อย่างน้อย 47 จาก 75 ครั้ง) ได้พิซซ่าที่บังเอิญผ่านทั้งสองข้ออยู่แล้ว ถ้าลองส่งด้วยมือครั้งเดียวก็มีโอกาสสูงที่จะเห็นผลถูกและคิดว่าระบบไม่มีปัญหา แต่ภายใต้ load มี 75 requests และทุก request ถูก check จึงเห็นบั๊กที่เกิดแค่บางครั้งออกมาเป็นอัตราที่ชัดเจนใน summary

## Part A results
| Step | Test type    | Users | Req/s | p(95) | Max | Failed | Checks |
| ---- | ------------ | ----- | ----- | ----- | --- | ------ | ------ |
| A1   | Smoke        | 1     | 0.53  | 361.1ms | 375.13ms | 0.00% | 100.00% |
| A2   | Average load | 3     | 1.34  | 362.57ms | 377.38ms | 0.00% | 100.00% |
| A3   | Stress       | 5     | 2.16  | 375.33ms | 662.43ms | 0.00% | 100.00% |
| A4   | Spike        | 5     | 1.04  | 371.87ms | 394.02ms | 0.00% | 100.00% |
| A5   | Soak         | 2     | 0.99  | 367.55ms | 1.06s | 0.00% | 100.00% |

หมายเหตุของแต่ละ step
- A1: checks ผ่าน 24 out of 24 และ thresholds ทั้ง 2 ข้อผ่าน (exit code 0) แสดงว่า script ใช้งานได้และพร้อมรัน test ที่ใหญ่กว่า ใน 30s มี 1 VU ทำได้ 6 iterations (18 requests) เพราะแต่ละ iteration ใช้เวลาเฉลี่ย 5.69s ซึ่งส่วนใหญ่เป็น think time
- A2: ผ่านทุก threshold คือ home p(95) = 268.78ms (< 1000), recommendation p(95) = 372.06ms (< 1500), checks 100% และ failed 0% (243 requests, exit code 0) ในส่วน Overview ของ report จำนวน VUs ขึ้นจาก 1 เป็น 3 ภายใน 30s แล้วค้างไว้ request rate ขึ้นตามเป็นประมาณ 1.4–1.7 req/s ช่วงที่ค้าง 3 VUs ส่วน p(95) รายช่วง 10 วินาทีอยู่ระหว่าง 325–369ms ตลอดการรันและไม่ได้สูงขึ้นตามจำนวน VUs
- A3: ผ่านทุก threshold คือ home p(95) = 276.52ms, recommendation p(95) = 384.38ms, checks 100% และ failed 0% (393 requests, exit code 0)
  - เทียบกับ A2: request rate เพิ่มจาก 1.34 เป็น 2.16 req/s (ช่วงที่ค้าง VUs ไว้ เพิ่มจาก 1.57 เป็น 2.53 req/s) แต่ p(95) เพิ่มจาก 362.57ms เป็น 375.33ms แค่ 12.76ms (ประมาณ 3.5%) ซึ่งน้อยกว่าความต่างระหว่างการรันซ้ำด้วย load เท่าเดิม (Step 0.6 รันสองครั้งได้ p(95) 374.05ms กับ 391.85ms ต่างกัน 17.8ms)
  - Max สูงขึ้นเป็น 662.43ms แต่มาจาก request เดียวในช่วงวินาทีที่ 70–80 ซึ่ง p(95) ของช่วงนั้นยังเป็น 382.2ms จึงน่าจะเป็นความหน่วงของเครือข่ายชั่วขณะมากกว่าเป็นผลของ load
- A4: ผ่านทุก threshold คือ home p(95) = 281.56ms, recommendation p(95) = 386.57ms, checks 100% และ failed 0% (150 requests, exit code 0)
  - จุด spike ใน report อยู่ที่วินาทีที่ 30–35 ซึ่ง VUs ขึ้นจาก 1 เป็น 5 และ request rate ขึ้นจาก 0.5 เป็น 1.9–2.7 req/s ตลอดช่วงวินาทีที่ 30–70
  - response time ไม่ได้สูงขึ้นระหว่าง spike: p(95) รายช่วง 10 วินาทีระหว่าง spike อยู่ที่ 329–374ms ไม่สูงกว่าช่วงก่อน spike (359–375ms) และค่า max ของทั้งการรัน (394.02ms) ก็เกิดในช่วง 10 วินาทีแรกที่มีแค่ 1 VU
  - หลังลดกลับเป็น 1 VU (วินาทีที่ 65–70) ช่วง 10 วินาทีแรกหลังจากนั้นได้ p(95) = 348.4ms ซึ่งอยู่ในระดับปกติแล้ว จึงไม่มีช่วงฟื้นตัวให้เห็นในความละเอียด 10 วินาทีของ report และไม่มี request ที่ fail เลย
- A5: ผ่านทุก threshold คือ home p(95) = 273.93ms, recommendation p(95) = 381.68ms, checks 100% และ failed 0% (660 requests ใน 11 นาที, exit code 0)
  - เทียบ 3 นาทีแรกกับ 3 นาทีสุดท้ายของช่วงที่ค้าง 2 VUs ได้ p(95) 367.0ms กับ 364.4ms, avg 286.9ms กับ 287.9ms และ failed 0.00% ทั้งสองช่วง p(95) รายนาทีอยู่ระหว่าง 354–375ms ทุกนาที ยกเว้นนาทีที่ 6 จึงไม่มีแนวโน้มว่าช้าลงหรือ error เพิ่มขึ้นตามเวลา
  - ใน report มีช่วงสะดุดสั้น ๆ ราววินาทีที่ 350–370 ที่ p(95) รายช่วง 10 วินาทีขึ้นเป็น 540.3ms และ 791.0ms และเกิด max ของทั้งการรัน 1.06s (ส่วนใหญ่เป็น request ของ quotes) แต่ช่วงถัดไป (370–380) กลับมาที่ 310.6ms เองและไม่มี request ที่ fail จึงเป็นความผิดปกติชั่วขณะ (น่าจะมาจากเครือข่ายหรือ load ของคนอื่นบน server ที่ใช้ร่วมกัน) ไม่ใช่การเสื่อมลงตามเวลา

## Part B
1. ตารางผลอยู่ในหัวข้อ Part A results ด้านบน ทั้ง 5 test ผ่านทุก threshold (exit code 0) ไม่มี request ที่ fail (Failed 0.00%) checks ผ่าน 100% และ p(95) ของทุก test อยู่ในช่วงแคบ ๆ คือ 361.1–375.33ms แม้จำนวน VUs จะต่างกันตั้งแต่ 1 ถึง 5
2. VU ใน k6 ทำงานเป็นรอบ ๆ คือส่ง request แล้วรอจนได้ response จากนั้น sleep(1) แล้วจึงเริ่ม iteration ถัดไป VU หนึ่งตัวจึงมี request ค้างอยู่ได้ทีละ 1 ตัวเท่านั้น ใน Step 0.2 iteration หนึ่งใช้เวลาเฉลี่ย 1.35s (http_req_duration เฉลี่ย 314.75ms บวก sleep 1s และ iteration แรกต้องเปิด connection ใหม่) rate สูงสุดที่ได้จึงเท่ากับ จำนวน VUs ÷ เวลาต่อ iteration = 5 ÷ 1.35 ≈ 3.7 req/s ใกล้กับที่วัดได้ 3.65 req/s (ต่ำกว่านิดหน่อยเพราะ k6 คิด rate จากเวลาทั้งหมด 21.35s ซึ่งรวมช่วงท้ายที่รอ iteration สุดท้ายให้เสร็จ) จะได้ 5 req/s ก็ต่อเมื่อ response ใช้เวลา 0 วินาที ซึ่งเป็นไปไม่ได้
3. สูงขึ้นเล็กน้อยแต่ไม่มีนัยสำคัญ p(95) เพิ่มจาก 362.57ms (A2, 3 VUs) เป็น 375.33ms (A3, 5 VUs) คือเพิ่ม 12.76ms (3.5%) ขณะที่ request rate เพิ่มถึง 61% (1.34 → 2.16 req/s)
   - ความต่างนี้น้อยกว่าความต่างระหว่างสองรอบของ Step 0.6 ที่ใช้ load เท่ากัน (17.8ms) และหน้า home ที่ server แทบไม่ต้องประมวลผลอะไร (http_req_waiting ใกล้เคียงกับเวลา TCP connect) ก็ช้าลงพอ ๆ กัน (median ของ http_req_waiting 260.0 → 271.6ms) แสดงว่าความต่างส่วนใหญ่มาจากเครือข่ายในช่วงเวลานั้น ไม่ใช่เพราะ server รับ load ไม่ไหว
   - operational envelope: ระบบสัญญาว่าจะตอบเร็วตราบใดที่ load ยังไม่เกินขอบของ envelope แต่ 5 VUs ที่มี think time สร้าง load แค่ประมาณ 2.5 req/s ช่วงที่ค้าง VUs ซึ่งน้อยมากสำหรับ site ที่คนทั้งโลกใช้ฝึก K6 พร้อมกัน ทั้ง A2 และ A3 จึงอยู่ลึกเข้าไปใน envelope ซึ่งเป็นช่วงที่ response time ควรคงที่ตามที่เห็น
   - test ที่เราเรียกว่า stress จึงหนักกว่าแค่ load ปกติของเราเอง แต่ยังไปไม่ถึงขอบของ envelope จึงบอกไม่ได้ว่าขอบอยู่ตรงไหน และไม่ได้ทดสอบว่าเมื่อเกินขอบแล้วระบบจะช้าลงอย่างนุ่มนวล (graceful degradation) หรือล่ม ถ้าจะหาขอบจริงต้องทดสอบกับระบบของเราเอง
4. ไม่มีช่วงฟื้นตัวให้เห็น เพราะ response time ไม่ได้สูงขึ้นระหว่าง spike
   - p(95) รายช่วง 10 วินาทีระหว่าง spike (วินาทีที่ 30–70, 5 VUs) อยู่ที่ 329–374ms เทียบกับ 359–375ms ก่อน spike และช่วง 10 วินาทีแรกหลังลดกลับเหลือ 1 VU (วินาทีที่ 70–80) ได้ p(95) = 348.4ms ซึ่งเป็นระดับปกติแล้ว เวลาที่ใช้กลับสู่ปกติจึงน้อยกว่า 10 วินาทีซึ่งเป็นความละเอียดของ report
   - สิ่งที่เปลี่ยนชัดมีแค่ request rate ที่ขึ้นจาก 0.5 เป็นสูงสุด 2.7 req/s แล้วกลับมาที่ 0.5–0.6 req/s ภายใน 20 วินาทีหลังลด VUs (VU ที่ถูกลดต้องทำ iteration ที่ค้างอยู่ให้เสร็จก่อน) spike จาก 1 เป็น 5 VUs เล็กเกินกว่าจะกระทบ QuickPizza
5. ไม่แย่ลง เมื่อเทียบ 3 นาทีแรกกับ 3 นาทีสุดท้ายของช่วงที่ค้าง 2 VUs ได้ p(95) 367.0ms กับ 364.4ms และ avg 286.9ms กับ 287.9ms ส่วน error rate เป็น 0.00% ตลอดทั้ง 660 requests
   - p(95) รายนาทีอยู่ระหว่าง 354–375ms ทุกนาที ยกเว้นช่วงสะดุดราววินาทีที่ 350–370 (max 1.06s) ซึ่งหายไปเองภายในประมาณ 20 วินาที จึงเป็นความผิดปกติชั่วขณะ ไม่ใช่แนวโน้มที่แย่ลงตามเวลา แต่ 10 นาทีที่ 2 VUs ก็สั้นและเบาเกินกว่าจะเห็นปัญหาที่ต้องสะสมนานหลายชั่วโมง
   - defect ที่ soak test หลายชั่วโมงอาจเจอ: memory leak เช่น server เก็บข้อมูลของทุก request ไว้ใน memory โดยไม่เคยคืน memory จะโตขึ้นเรื่อย ๆ จน garbage collector ทำงานหนัก response ช้าลงทีละน้อย และสุดท้าย process ล่มเพราะ out of memory ซึ่ง test สั้น ๆ ไม่มีทางเห็น (ตัวอย่างอื่น: connection หรือ file handle รั่วจน pool หมด, ตารางในฐานข้อมูลหรือ log file โตจนช้าหรือ disk เต็ม)
6. ค่าเฉลี่ยซ่อน request ที่ช้าที่สุดเอาไว้ response time มักกระจายแบบหางยาวไปทางขวา request ที่ช้ามากไม่กี่ตัวจึงถูกเฉลี่ยกลืนไปกับ request เร็วจำนวนมาก
   - ตัวอย่างจาก A3: avg = 294.97ms แต่ p(95) = 375.33ms และ max = 662.43ms ถ้าดูแค่ avg จะไม่รู้เลยว่ามีผู้ใช้ราว 1 ใน 20 ที่ต้องรอนานกว่า 375ms
   - ถ้า 94% ของ requests ใช้ 0.3s และอีก 6% ใช้ 5s ค่าเฉลี่ยจะเป็น 0.58s ซึ่งดูดี แต่ p(95) จะเป็น 5s ซึ่งบอกความจริงว่าผู้ใช้กลุ่มหนึ่งรอนานมาก
   - p(95) แปลเป็นคำสัญญากับผู้ใช้ได้ตรง ๆ ว่า "95% ของ requests เสร็จภายใน X" และไม่แกว่งไปตาม outlier ตัวเดียวเหมือน max
7. เหตุผลสองข้อ
   - เครือข่าย: request ของเราวิ่งผ่าน internet จากไทยไปถึง server ของ QuickPizza เวลา TCP connect (ใช้ 1 round trip) วัดได้ 252–291ms และ http_req_waiting ของหน้า home มี median 260–275ms แปลว่า response time เกือบทั้งหมดคือเวลาเดินทางของข้อมูล ไม่ใช่เวลาที่ server ทำงาน ใน lab ที่ client กับ server อยู่ใน LAN เดียวกัน round trip ไม่ถึง 1ms ตัวเลขจึงต่ำกว่ามาก (home เหลือไม่กี่ ms และ recommendation เหลือราว 70ms ซึ่งเป็นเวลาที่ server ทำงานจริง) และแกว่งน้อยกว่า ส่วนบน internet เส้นทาง, Wi-Fi และความหนาแน่นของ ISP ทำให้ตัวเลขแกว่ง เช่น max 662.43ms ใน A3 ที่เกิดแค่ request เดียว
   - ระบบที่ใช้ร่วมกัน: QuickPizza มีคนทั้งโลกใช้ฝึก K6 พร้อมกัน load จากคนอื่นในเวลาเดียวกันจึงเปลี่ยนตัวเลขของเราได้ (เช่นช่วงสะดุดสั้น ๆ ใน A5) และ infrastructure ของ Grafana (จำนวน server, autoscaling, cache, load balancer, rate limit) ก็ต่างจากเครื่องใน lab ที่เรากำหนด hardware, config และ load ได้เองทั้งหมดโดยไม่มีใครมาแชร์ ผลใน lab จึงทำซ้ำได้มากกว่า
8. load หลายร้อย users ที่เจ้าของไม่ได้อนุญาตไม่ต่างจากการโจมตีแบบ DoS
   - ทำให้ site ช้าหรือล่มสำหรับผู้ใช้จริงและคนอื่นที่ใช้ร่วมกัน เช่นคนที่กำลังฝึก K6 บน QuickPizza
   - ทำให้เจ้าของเสียค่า cloud หรือ autoscaling เพิ่ม และอาจทำให้ระบบ security ส่ง alert หรือ block IP ของเรา
   - ผิดกฎหมาย เช่น พ.ร.บ.ว่าด้วยการกระทำความผิดเกี่ยวกับคอมพิวเตอร์ พ.ศ. 2550 มาตรา 10 ที่ห้ามทำให้ระบบคอมพิวเตอร์ของผู้อื่นถูกระงับ ชะลอ ขัดขวาง หรือรบกวนจนทำงานตามปกติไม่ได้ และผิดเงื่อนไขการใช้งานของ site ด้วย
   - การทดสอบแบบนี้ต้องทำกับระบบของเราเอง (เช่น staging) หรือมีหนังสืออนุญาตจากเจ้าของที่ระบุขอบเขตและช่วงเวลาไว้ชัดเจน
9. แอปสั่งอาหารที่ใช้ทุกวัน
   - requirement ที่คลุมเครือ: "แอปต้องค้นหาร้านอาหารได้เร็ว แม้ช่วงพักเที่ยงที่คนใช้เยอะ"
   - requirement ที่ทดสอบได้: "เมื่อมีผู้ใช้ค้นหาร้านพร้อมกัน 500 คนต่อเนื่อง 15 นาที (จำลองช่วงพักเที่ยง) API ค้นหาร้าน `GET /api/restaurants/search` ต้องตอบ 95% ของ requests ภายใน 800ms และ 99% ภายใน 1.5s, requests ที่ fail ต้องน้อยกว่า 0.5% และมากกว่า 99% ของการค้นหาต้องได้ร้านอย่างน้อย 1 ร้าน" (รันกับ staging ของทีมเอง)
   - K6 thresholds ที่ใช้ตรวจ:

   ```js
   import http from 'k6/http';
   import { check, sleep } from 'k6';

   const BASE_URL = 'https://staging.food-app.example';

   export const options = {
     stages: [
       { duration: '2m', target: 500 },
       { duration: '15m', target: 500 },
       { duration: '2m', target: 0 },
     ],
     thresholds: {
       'http_req_duration{name:search}': ['p(95)<800', 'p(99)<1500'],
       'http_req_failed{name:search}': ['rate<0.005'],
       checks: ['rate>0.99'],
     },
   };

   export default function () {
     const res = http.get(`${BASE_URL}/api/restaurants/search?q=pizza`, {
       tags: { name: 'search' },
     });
     check(res, { 'search has results': (r) => r.json('restaurants').length > 0 });
     sleep(1 + Math.random() * 4);
   }
   ```
