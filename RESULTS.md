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
| A3   | Stress       | 5     |       |       |     |        |        |
| A4   | Spike        | 5     |       |       |     |        |        |
| A5   | Soak         | 2     |       |       |     |        |        |

หมายเหตุของแต่ละ step
- A1: checks ผ่าน 24 out of 24 และ thresholds ทั้ง 2 ข้อผ่าน (exit code 0) แสดงว่า script ใช้งานได้และพร้อมรัน test ที่ใหญ่กว่า ใน 30s มี 1 VU ทำได้ 6 iterations (18 requests) เพราะแต่ละ iteration ใช้เวลาเฉลี่ย 5.69s ซึ่งส่วนใหญ่เป็น think time
- A2: ผ่านทุก threshold คือ home p(95) = 268.78ms (< 1000), recommendation p(95) = 372.06ms (< 1500), checks 100% และ failed 0% (243 requests, exit code 0) ในส่วน Overview ของ report จำนวน VUs ขึ้นจาก 1 เป็น 3 ภายใน 30s แล้วค้างไว้ request rate ขึ้นตามเป็นประมาณ 1.4–1.7 req/s ช่วงที่ค้าง 3 VUs ส่วน p(95) รายช่วง 10 วินาทีอยู่ระหว่าง 325–369ms ตลอดการรันและไม่ได้สูงขึ้นตามจำนวน VUs

## Part B
1.
2.
3.
4.
5.
6.
7.
8.
9.
