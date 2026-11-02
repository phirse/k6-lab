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
- Step 0.6, exit code of the normal run and of the run with P95=300:
- Step 0.7, rules that passed, p(95), p(99), exit code after your change:
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
