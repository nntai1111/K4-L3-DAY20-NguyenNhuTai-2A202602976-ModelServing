# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Như Tài
**MSSV:** 2A202602976
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 Home Single Language (AMD64)
- **CPU:** AMD Ryzen 7 5800H with Radeon Graphics
- **Cores:** 8 physical / 16 logical
- **CPU extensions:** AVX2 (Zen 3, không có AVX-512)
- **RAM:** 13.9 GB usable (2 × 8 GB DDR4-3200, dual channel ≈ 51.2 GB/s lý thuyết)
- **Accelerator:** NVIDIA GeForce RTX 3050 Ti Laptop GPU, 4 GB GDDR6 (≈ 192 GB/s) — chạy qua **Vulkan**
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-vulkan-x64.zip`
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL (primary) + UD-Q2_K_XL (compare) (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi (không dùng Colab/Kaggle).

**Setup story** (≤ 80 chữ): Driver NVIDIA 12.3 quá cũ cho bản CUDA prebuilt nên
runtime tự chuyển sang bản Vulkan (vẫn offload 100% lên GPU, `ngl=99`). Phải sửa ba
chỗ để lab chạy: `lab.ps1` lỗi parse trên PowerShell 5.1 (UTF-8 thiếu BOM); `serve.py`
dùng `os.execv` làm vỡ đường dẫn có dấu cách `D:\vinuni AI\…` → đổi sang
`subprocess` trên Windows; thêm `PYTHONUTF8=1` để report không bị lỗi ký tự.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 9403 | 303 / 915 | 14.6 / 14.7 | 1210 / 1809 / 1809 | 68.3 |
| UD-Q2_K_XL | 2.24 | 6985 | 312 / 1166 | 15.8 / 16.2 | 1291 / 2150 / 2150 | 63.1 |

**Quan sát** (≤ 60 chữ): 2-bit nhỏ hơn 25% nhưng **chậm hơn 1.08×** (TPOT 15.8 vs
14.6 ms; lần chạy trước cũng chậm hơn 1.04×): model nằm gọn trong VRAM, decode ~15 ms/token chủ yếu là overhead cố định
mỗi bước chứ không phải băng thông, còn dequant Q2_K tốn thêm. Hỏi cùng 3 câu: cả hai
đúng, 4-bit chặt chẽ hơn, 2-bit lan man. **Không đáng** — giữ 4-bit.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 1.38 | 3700 | 24000 | 26000 | 8.4 | 0 |
| 50 | 2.27 | 20000 | 22000 | 22000 | 39.4 | 0 |

- **Offered load tăng 5×, throughput thực tăng:** 1.64× (steady-state: ~1.0×, xem dưới)
- **P95 tăng:** 0.92× (P95 của 10 users bị cold start đẩy lên — xem dưới)
- **Effective concurrency ở 50 users:** 39.4 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.96 / 4 slots (`requests_deferred` peak 45)

**Saturation reading** (≤ 80 chữ): Bão hoà **ngay từ 10 users**: theo
`stats_history`, RPS steady-state đi ngang ~2.3–2.4 ở **cả** 10 và 50 users; 1.64× chỉ
do ~15 request đầu của run 10-user mất 23–26 s (cold start Vulkan cho batch nhiều
slot) — cũng là nguồn của P95 24 s. Little's Law: 4 slot / 2.35 req/s ≈ 1.7 s compute,
P50 ~20 s ⇒ ~90% là **queue time** (khớp `deferred` = 45). Knob đầu tiên:
`--parallel 8` — thêm sequence vào cùng một bước decode dùng lại cùng lượt đọc weights.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Không có — chạy hoàn toàn trên laptop | stub |
| N17 Data pipeline | `TOY_DOCS` hard-code trong `pipeline.py` | stub |
| N18 Lakehouse | Không có storage layer, docs nằm trong RAM | stub |
| N19 Vector + features | `retrieve()` keyword overlap, không có embedding server | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 715.6 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): Lần chạy đầu llm = 2873 ms dù server chỉ tốn ~0.6 s: trên
Windows `localhost` thử `::1` trước, mất ~2 s mỗi kết nối mới (đo: 2030 ms vs 1 ms với
`127.0.0.1`). Đổi `--base-url` → 716 ms (4.0×). Phần còn lại là decode ~14 ms/token,
nên muốn giảm 2× nữa: giới hạn `max_tokens`, giữ prefix prompt ổn định để prefill
được cache.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** đưa toàn bộ model lên GPU — `LAB_N_GPU_LAYERS=0` (CPU, `-t 8`) →
`ngl=99` (RTX 3050 Ti qua Vulkan, `-t 8`). Cùng model UD-Q4_K_XL, cùng `llama-bench tg128`.

```
before:  12.3 tok/s   (ngl=0,  -t 8, decode tg128)
after:   74.9 tok/s   (ngl=99, -t 8, decode tg128)
speedup: 6.1×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Decode sinh từng token một: mỗi token phải đọc lại **toàn bộ weights** từ bộ nhớ,
nhưng mỗi byte chỉ dùng cho vài phép nhân-cộng. Nên tốc độ decode ≈ băng thông bộ nhớ
÷ số byte đọc mỗi token, không phải FLOPs. Máy tôi có hai loại bộ nhớ rất khác nhau:
DDR4-3200 dual channel ≈ 51.2 GB/s cho CPU, và GDDR6 12 Gbps × 128-bit ≈ 192 GB/s
cho GPU — chênh **3.75×**. Thread sweep CPU-only xác nhận CPU đang chạm trần băng thông:
tốc độ tăng 1 → 4 thread (8.9 → ~12.6 tok/s) rồi **đi ngang** ở 4–8 thread
(12.2–12.8, đo lại 3 lần), tức thêm core không thêm byte/giây nào; quá 8 core vật lý thì
**sụp** (16 thread: 5.2, 32 thread: 2.9) vì SMT siblings tranh cùng L1/L2 và llama.cpp
đồng bộ mọi thread ở barrier sau mỗi op — một thread bị OS preempt làm cả bước chờ.

Speedup đo được (6.1×) **lớn hơn** tỉ lệ băng thông lý thuyết (3.75×) — khác với kỳ
vọng "speedup = tỉ lệ bandwidth". Giải thích của tôi: CPU không dùng được hết 51.2 GB/s
(thực tế dual-channel DDR4 thường chỉ đạt ~60–70%, và CPU còn phải tự dequantize Q4_K
bằng AVX2 trên 8 core, chiếm thêm thời gian), còn GPU dequantize trong shader gần như
miễn phí và giữ bus GDDR6 bận liên tục. Nói cách khác, tỉ lệ = 3.75× (bandwidth) × ~1.6×
(hiệu suất khai thác bandwidth). Hệ quả thứ hai đáng chú ý: khi đã lên GPU, **thread
count gần như không còn ý nghĩa** (sweep `ngl=99`: 70.0 → 74.9 tok/s, chênh 1.07×), và
**2-bit không còn nhanh hơn 4-bit** (§2) — vì cổ chai đã chuyển từ băng thông sang overhead
cố định mỗi bước decode. Một knob chỉ có tác dụng khi nó đánh vào đúng cổ chai hiện tại.

Ghi chú trung thực: lần sweep CPU đầu tiên cho `-t 4` = 18.4 tok/s, nhưng đo lại
(`llama-bench -t 2,4,6,8 -r 3`) chỉ được 12.6 ± 0.8 — tôi coi 18.4 là outlier do boost
nhiệt khi máy còn mát, nên dùng `-t 8` = 12.3 tok/s (mặc định của lab) làm "before".

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Cải thiện latency lớn thứ hai của tôi không liên quan đến model: đổi `localhost` thành
`127.0.0.1` làm stage llm của pipeline nhanh **4.0×** (2873 → 716 ms), vì Windows thử
IPv6 `::1` trước và chờ ~2 s mỗi kết nối mới. Không đo tách server timing với client
timing thì sẽ đổ lỗi nhầm cho GPU.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của mình
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [ ] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Claude Code (Anthropic): chạy các lệnh lab trên máy tôi (bench, tune, load-report,
pipeline), chẩn đoán và sửa 3 lỗi chạy trên Windows (BOM của `lab.ps1`, `os.execv` với
đường dẫn có dấu cách, encoding report), phát hiện độ trễ 2 s của `localhost`, và soạn
nháp các phần nhận xét / REFLECTION từ số đo thật. Tôi tự chạy serve, smoke, load-10,
load-50, metrics và chụp screenshot; tôi đã đọc và kiểm tra lại các con số và lập luận.
