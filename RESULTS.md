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
- Step 0.8, what was wrong, and how the checks showed it:

## Part A results
| Step | Test type    | Users | Req/s | p(95) | Max | Failed | Checks |
| ---- | ------------ | ----- | ----- | ----- | --- | ------ | ------ |
| A1   | Smoke        | 1     |       |       |     |        |        |
| A2   | Average load | 3     |       |       |     |        |        |
| A3   | Stress       | 5     |       |       |     |        |        |
| A4   | Spike        | 5     |       |       |     |        |        |
| A5   | Soak         | 2     |       |       |     |        |        |

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
