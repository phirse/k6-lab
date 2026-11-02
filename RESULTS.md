# K6 lab results

## Part 0
- Step 0.1, http_req_duration: avg=277.8ms min=277.8ms med=277.8ms max=277.8ms p(90)=277.8ms p(95)=277.8ms
  - ทุกค่าเท่ากันเพราะไม่ได้ใส่ option จึงรัน 1 VU แค่ 1 iteration มีเพียง 1 request (checks_succeeded 100.00% 1 out of 1, http_reqs = 1, iterations = 1)
  - iteration_duration = 1.92s ซึ่งนานกว่า sleep(1) บวกเวลาของ request เพราะ request แรกต้องเปิด connection (DNS, TCP, TLS) ก่อน และเวลาส่วนนี้ไม่ได้นับรวมใน http_req_duration
- Step 0.2, requests, rate and p(95), and why the rate stays below 5:
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
