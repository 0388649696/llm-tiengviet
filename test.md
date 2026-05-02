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

 Dưới đây là **bộ Evaluation Datasets + Auto Generator** để bạn có thể đo được **C, D, G** thật (không giả lập) và nuôi Validation Layer ổn định.

---

# 🧠 1. Cấu trúc dữ liệu

```bash
kg/data/
├── eval/
│   ├── paraphrase.jsonl        # gần nghĩa / không gần nghĩa (S)
│   ├── contrast.jsonl          # positive vs negative (D)
│   ├── logic.jsonl             # cause / condition / contrast (logic)
│   ├── role.jsonl              # entity/behavior/concept (role nhẹ)
│   └── generalization.jsonl    # áp dụng pattern sang case mới (G)
│
├── seeds/
│   ├── patterns.json           # seed pattern
│   ├── templates.json          # template sinh câu
│   └── lexicon.json            # danh sách entity/verb/object theo domain
│
└── generated/
    ├── train_cases.jsonl
    └── eval_generated.jsonl
```

> Dùng `.jsonl` (mỗi dòng 1 JSON) để dễ stream/append.

---

# 🧩 2. Schema từng tập

## 2.1 Paraphrase (S – Consistency)

```json
{"id":"p_0001",
 "a":"LLM hiểu dữ liệu lớn",
 "b":"AI xử lý dữ liệu quy mô lớn",
 "label":1,
 "domain":"ai"}
```

```json
{"id":"p_0002",
 "a":"LLM hiểu dữ liệu lớn",
 "b":"Docker chạy container",
 "label":0,
 "domain":"ai"}
```

---

## 2.2 Contrast (D – phân biệt)

```json
{"id":"c_0001",
 "positive":"AI xử lý ngôn ngữ tự nhiên",
 "negative":"Hệ thống lưu trữ dữ liệu",
 "pattern":"X hiểu/ xử lý Y",
 "label":1}
```

---

## 2.3 Logic (nhân quả/điều kiện/nhượng bộ)

```json
{"id":"l_0001",
 "text":"Vì dữ liệu ít nên mô hình học kém",
 "type":"cause_effect",
 "spans":{"cause":"dữ liệu ít","effect":"mô hình học kém"}}
```

```json
{"id":"l_0002",
 "text":"Nếu dữ liệu đủ thì mô hình học tốt",
 "type":"condition",
 "spans":{"cond":"dữ liệu đủ","result":"mô hình học tốt"}}
```

```json
{"id":"l_0003",
 "text":"Mặc dù dữ liệu ít nhưng mô hình vẫn học tốt",
 "type":"contrast",
 "spans":{"a":"dữ liệu ít","b":"mô hình vẫn học tốt"}}
```

---

## 2.4 Role (nhẹ, không hard rule)

```json
{"id":"r_0001",
 "text":"LLM hiểu dữ liệu lớn",
 "roles":[
   {"span":"LLM","role":"entity_like"},
   {"span":"hiểu","role":"behavior_like"},
   {"span":"dữ liệu lớn","role":"concept_like"}
 ]}
```

---

## 2.5 Generalization (G)

```json
{"id":"g_0001",
 "pattern":"X hiểu Y",
 "train_examples":[
   "LLM hiểu dữ liệu lớn",
   "AI hiểu ngôn ngữ tự nhiên"
 ],
 "test_case":"Robot hiểu tín hiệu cảm biến",
 "expect_match":true}
```

---

# 🔧 3. Seeds để sinh dữ liệu

## 3.1 patterns.json

```json
{
  "patterns": [
    "X hiểu Y",
    "X xử lý Y",
    "X học từ Y",
    "vì X nên Y",
    "nếu X thì Y",
    "mặc dù X nhưng Y"
  ]
}
```

## 3.2 templates.json

```json
{
  "simple": [
    "{X} hiểu {Y}",
    "{X} xử lý {Y}",
    "{X} học từ {Y}"
  ],
  "logic": [
    "vì {X} nên {Y}",
    "nếu {X} thì {Y}",
    "mặc dù {X} nhưng {Y}"
  ]
}
```

## 3.3 lexicon.json (domain AI/dev ví dụ)

```json
{
  "entities": ["LLM","AI","Mô hình","Hệ thống","Robot"],
  "verbs": ["hiểu","xử lý","học","lưu trữ"],
  "objects": [
    "dữ liệu lớn",
    "ngôn ngữ tự nhiên",
    "tín hiệu cảm biến",
    "thông tin người dùng"
  ],
  "conditions": [
    "dữ liệu ít",
    "dữ liệu đủ",
    "tài nguyên hạn chế"
  ],
  "results": [
    "mô hình học kém",
    "mô hình học tốt",
    "hiệu suất giảm",
    "kết quả chính xác"
  ]
}
```

---

# ⚙️ 4. Auto Generator (Python)

```python
import json, random, uuid, itertools
from pathlib import Path

ROOT = Path("kg/data")
SEEDS = ROOT / "seeds"
OUT_GEN = ROOT / "generated"

def load_json(p): return json.loads(Path(p).read_text(encoding="utf-8"))

def rid(prefix): return f"{prefix}_{uuid.uuid4().hex[:8]}"

def gen_simple(templates, lex, n=200):
    rows = []
    for _ in range(n):
        t = random.choice(templates["simple"])
        X = random.choice(lex["entities"])
        Y = random.choice(lex["objects"])
        text = t.format(X=X, Y=Y)
        rows.append({
            "id": rid("s"),
            "text": text,
            "pattern_hint": t,
            "domain": "ai"
        })
    return rows

def gen_logic(templates, lex, n=200):
    rows = []
    for _ in range(n):
        t = random.choice(templates["logic"])
        X = random.choice(lex["conditions"])
        Y = random.choice(lex["results"])
        text = t.format(X=X, Y=Y)
        rows.append({
            "id": rid("l"),
            "text": text,
            "type_hint": t.split()[0],  # vì / nếu / mặc dù
            "domain": "ai"
        })
    return rows

def gen_paraphrase(simple_rows, n=200):
    # tạo cặp gần nghĩa (thay verb tương đương nhẹ)
    verb_map = {"hiểu":"xử lý", "xử lý":"hiểu", "học":"hiểu"}
    rows = []
    for _ in range(n):
        a = random.choice(simple_rows)["text"]
        parts = a.split()
        if len(parts) >= 3 and parts[1] in verb_map:
            b = a.replace(parts[1], verb_map[parts[1]])
            rows.append({"id": rid("p"), "a": a, "b": b, "label": 1})
        # negative
        c = random.choice(simple_rows)["text"]
        rows.append({"id": rid("p"), "a": a, "b": c, "label": 0})
    return rows

def gen_contrast(simple_rows, n=200):
    rows = []
    for _ in range(n):
        pos = random.choice(simple_rows)["text"]
        neg = random.choice(simple_rows)["text"]
        rows.append({
            "id": rid("c"),
            "positive": pos,
            "negative": neg,
            "pattern": "X hiểu/ xử lý Y",
            "label": 1
        })
    return rows

def gen_generalization(simple_rows, n=200):
    rows = []
    for _ in range(n):
        train = random.sample(simple_rows, k=min(2, len(simple_rows)))
        test = random.choice(simple_rows)["text"]
        rows.append({
            "id": rid("g"),
            "pattern": "X hiểu Y",
            "train_examples": [t["text"] for t in train],
            "test_case": test,
            "expect_match": True
        })
    return rows

def save_jsonl(path, rows):
    path.parent.mkdir(parents=True, exist_ok=True)
    with open(path, "w", encoding="utf-8") as f:
        for r in rows:
            f.write(json.dumps(r, ensure_ascii=False) + "\n")

def main():
    templates = load_json(SEEDS/"templates.json")
    lex = load_json(SEEDS/"lexicon.json")

    simple = gen_simple(templates, lex, n=500)
    logic = gen_logic(templates, lex, n=300)

    paraphrase = gen_paraphrase(simple, n=400)
    contrast = gen_contrast(simple, n=400)
    generalization = gen_generalization(simple, n=300)

    save_jsonl(OUT_GEN/"train_cases.jsonl", simple + logic)
    save_jsonl(ROOT/"eval/paraphrase.jsonl", paraphrase)
    save_jsonl(ROOT/"eval/contrast.jsonl", contrast)
    save_jsonl(ROOT/"eval/generalization.jsonl", generalization)

    # logic eval có thể tách riêng từ logic rows
    save_jsonl(ROOT/"eval/logic.jsonl", logic)

if __name__ == "__main__":
    main()
```

---

# 🧪 5. Gắn vào Validation Layer

* **S (Consistency)**: dùng `eval/paraphrase.jsonl` → accuracy ≥ 0.8
* **D (Contrast)**: dùng `eval/contrast.jsonl` → accuracy ≥ 0.75
* **G (Generalization)**: dùng `eval/generalization.jsonl` → success ≥ 0.7
* **Logic**: dùng `eval/logic.jsonl` → accuracy ≥ 0.6 (ban đầu)

---

# 📏 6. Khối lượng khuyến nghị

* Starter: 1–2k dòng (tự sinh)
* Usable: 5–10k dòng (mix tự sinh + thủ công 20–30%)
* Mỗi pattern: ~150–250 câu (đủ lên HIỂU/THẤU)

---

# ⚠️ 7. Lưu ý quan trọng

* Tự sinh phải **đa dạng template** (tránh overfit cấu trúc)
* Trộn **20–30% dữ liệu người viết** vào eval (chống bias AI)
* Giữ **eval cố định** (đo drift theo thời gian)

---

# 🎯 1 câu chốt

```text
Dataset = “bài kiểm tra” của hệ học.
Không có bài kiểm tra tốt → không có “HIỂU” thật.
```
