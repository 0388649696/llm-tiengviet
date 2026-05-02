OK, đây là **Validation Layer + Criteria** ở mức **production-ready** cho hệ hybrid của bạn. Mục tiêu:
**Learning tạo candidate → Validation quyết định → KG chỉ nhận cái “đủ chín”**.

---

# 🧠 1. Vai trò Validation Layer

```text
Input: candidate (pattern / concept / relation) + evidence
Output: verdict {REJECT | QUARANTINE | ACCEPT} + score + reason
```

---

# 🧩 2. Chuẩn dữ liệu đầu vào (Candidate)

```json
{
  "type": "pattern",          // pattern | concept | relation
  "value": "X hiểu Y",
  "examples": [
    "LLM hiểu dữ liệu lớn",
    "AI hiểu ngôn ngữ tự nhiên"
  ],
  "source": ["human","ai"],
  "teacher_signals": {
    "vncorenlp": {...},
    "underthesea": {...},
    "embedding": {...}
  },
  "stats": {
    "freq": 18,
    "unique_contexts": 7,
    "age": 120               // số lần update đã sống
  }
}
```

---

# 🔥 3. Bộ tiêu chí (Criteria)

## 3.1 Coverage (độ phủ)

```text
C = min(1, freq / FREQ_TARGET)
```

* `FREQ_TARGET` mặc định: 20
* Ý nghĩa: xuất hiện đủ nhiều chưa?

---

## 3.2 Consistency (ổn định ngữ cảnh)

Dùng embedding (cosine) giữa các ví dụ:

```text
S = average_pairwise_similarity(examples)
```

* tốt: ≥ 0.7
* cảnh báo: 0.5–0.7
* xấu: < 0.5

---

## 3.3 Contrastiveness (khả năng phân biệt)

So với negative set (tự sinh):

```text
D = 1 - average_similarity(positive, negative)
```

* tốt: ≥ 0.6
* thấp → pattern mơ hồ

---

## 3.4 Teacher Agreement (đồng thuận “giáo viên”)

So khớp tín hiệu từ VnCoreNLP và underthesea:

```text
T = weighted_agreement(vncorenlp, underthesea)
```

* ví dụ: vị trí “động từ” trùng nhau, chunk tương tự
* tốt: ≥ 0.7

---

## 3.5 Generalization (khả năng áp dụng mới)

Sinh câu mới (AI hoặc template), đo match:

```text
G = success_rate_on_generated_cases
```

* tốt: ≥ 0.7

---

## 3.6 Noise / Anomaly (phát hiện bẩn)

Rule nhẹ để bắt “rác”:

* verb chứa token object phổ biến (vd: “hiểu dữ”)
* span quá dài/ngắn bất thường
* ký tự lạ

```text
N = anomaly_score ∈ [0,1]  (càng cao càng xấu)
```

* chấp nhận: ≤ 0.3

---

## 3.7 Temporal Stability (ổn định theo thời gian)

```text
H = stability_over_updates
```

* đo độ biến thiên của S, D theo các batch gần nhất
* tốt: ≥ 0.6

---

# 🧮 4. Hàm chấm điểm tổng

Chuẩn hoá về [0,1]:

```text
score =
  0.20*C +
  0.20*S +
  0.15*D +
  0.15*T +
  0.15*G +
  0.10*H -
  0.15*N
```

---

# 🎯 5. Ngưỡng quyết định

| Verdict        | Điều kiện                |
| -------------- | ------------------------ |
| **ACCEPT**     | score ≥ 0.75 AND N ≤ 0.3 |
| **QUARANTINE** | 0.5 ≤ score < 0.75       |
| **REJECT**     | score < 0.5 OR N > 0.6   |

---

# 🧠 6. Trạng thái (State Machine gắn với Validation)

```text
BIẾT   → nếu C ≥ 0.3
THÔNG  → nếu C≥0.6 & S≥0.65 & D≥0.55
HIỂU   → nếu S≥0.7 & D≥0.6 & T≥0.7 & G≥0.6
THẤU   → nếu G≥0.75 & H≥0.6 & score≥0.8
```

---

# 🧪 7. Ví dụ đánh giá

```json
{
  "pattern": "X hiểu Y",
  "metrics": {
    "C": 0.9,
    "S": 0.78,
    "D": 0.64,
    "T": 0.72,
    "G": 0.70,
    "H": 0.68,
    "N": 0.1
  },
  "score": 0.79,
  "verdict": "ACCEPT",
  "state": "HIỂU"
}
```

---

# 🔧 8. Code khung

```python
def validate(candidate):
    C = coverage(candidate)
    S = consistency(candidate)
    D = contrastiveness(candidate)
    T = teacher_agreement(candidate)
    G = generalization(candidate)
    H = temporal_stability(candidate)
    N = anomaly_score(candidate)

    score = (
        0.20*C + 0.20*S + 0.15*D +
        0.15*T + 0.15*G + 0.10*H -
        0.15*N
    )

    if score >= 0.75 and N <= 0.3:
        verdict = "ACCEPT"
    elif score >= 0.5:
        verdict = "QUARANTINE"
    else:
        verdict = "REJECT"

    return {
        "score": round(score, 3),
        "verdict": verdict,
        "metrics": {"C":C,"S":S,"D":D,"T":T,"G":G,"H":H,"N":N}
    }
```

---

# 🔁 9. Luồng tích hợp

```text
Learning Core
  → sinh candidate
  → validate()
      → REJECT      → bỏ
      → QUARANTINE  → giữ theo dõi
      → ACCEPT      → ghi vào KG
KG
  → dùng cho inference
  → phản hồi ngược (feedback) cho Learning
```

---

# ⚠️ 10. Lưu ý vận hành

* Không hạ ngưỡng quá sớm (tránh “nhiễm bẩn KG”)
* Quarantine phải có TTL (vd: 3–5 vòng update, không cải thiện thì loại)
* Eval set cố định để đo drift
* Teacher chỉ là “phiếu tham khảo”, không phải chân lý

---

# 🎯 1 câu chốt

```text
Validation Layer = nơi quyết định cái gì được gọi là “tri thức”.
```

---

Nếu bạn muốn, bước tiếp theo nên là:

👉 `build evaluation datasets + auto generator` (để có C, D, G đo được thật, không giả lập)
