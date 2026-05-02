# llm-tiengviet
## Giới thiệu mô hình
 Phase 1: train tokenizer riêng tiếng Việt từ raw corpus. Không dùng VnCoreNLP/underthesea.

Phase 2: train sentence encoder nhỏ bằng contrastive learning.

Phase 3: build evaluator: 100–300 cặp câu Việt do bạn tự viết để đo “gần nghĩa / khác nghĩa / nhân quả”.

Phase 4: dùng semantic clusters để sinh KG.

Phase 5: nếu cần sinh câu, mới gắn một decoder nhỏ hoặc dùng LLM ngoài làm verbalizer.
##### 📌 Quy trình hoạt động:
```mermaid
graph LR
A(Text)
A --> B[Weak Parser - Rule nhẹ]
A --> C[Experience Memory]

C --> D[Pattern Discovery]
D --> E[Role Emergence]

B --> E

E --> F[Concept Formation]
E --> G[Behavior Emergence]

F --> H[Relation Induction]
G --> H

H --> I[(Knowledge Graph)]
```
- Mô hình đang ở mức logic phục vụ cho Knowledge Graph
- 
#### So sánh với KG Engine truyền thống
| Tiêu chí             | Mô hình này (Experience-based) | KG Engine truyền thống |
| -------------------- | ------------------------------ | ---------------------- |
| Triết lý             | Từ trải nghiệm → tri thức      | Từ schema → tri thức   |
| Điểm xuất phát       | Text tự nhiên                  | Ontology / database    |
| Cách hiểu            | Pattern + ngữ cảnh             | Mapping cố định        |
| Tạo concept          | Emergent (tự hình thành)       | Định nghĩa trước       |
| Tạo relation         | Induction từ pattern           | Rule + schema          |
| Mức độ tự động       | ✔ Cao                          | ❌ Thấp                 |
| Phụ thuộc con người  | ✔ Ít                           | ❌ Nhiều                |
| Khả năng học         | ✔ Có (tích lũy)                | ❌ Gần như không        |
| Độ chính xác ban đầu | ⚠️ Thấp                        | ✔ Cao                  |
| Độ thích nghi        | ✔ Rất cao                      | ❌ Thấp                 |
| Giải thích           | ✔ Rõ                           | ✔ Rõ                   |


| Ưu/nhược           | Mô hình của tôi          | KG Engine truyền thống        |
| ------------------- | ------------------------ | ----------------------------- |


#### Version:
v1: Knowledge Graph Engine kiểu mới: Mô hình đang ở mức logic phục vụ cho Knowledge Graph, chưa Query Engine, Reasoning, Learning, Hybrid

# 1. Kiến trúc tổng quát
## A. LỚP CẤU TRÚC LOGIC:
### Text → Input
```
- Văn bản tự nhiên (raw text)
- Không yêu cầu schema, không preprocessing nặng
- CNN không hiểu nếu dữ liệu ít
```
### Weak Parser (Rule nhẹ)
```
👉 Vai trò:
KHÔNG hiểu ngữ nghĩa
Chỉ chia câu thành các phần thô

Ví dụ:
[cnn] [không hiểu] [nếu dữ liệu ít]

👉 Đặc điểm:
Không cần danh sách verb đầy đủ
Không phụ thuộc lexicon cứng
Chỉ là “gợi ý cấu trúc ban đầu”
```
### Experience Memory
```
👉 Lưu toàn bộ trải nghiệm:

{
  "sentence": "cnn không hiểu nếu dữ liệu ít",
  "tokens": [...],
  "l1_guess": {...}
}

👉 Đây là “trí nhớ” của hệ
```

### Pattern Discovery

```
👉 Tìm cái lặp lại giữa nhiều câu

Ví dụ:

cnn hiểu dữ liệu lớn
cnn hiểu dữ liệu tốt

→ phát hiện:

X hiểu Y
```

### Role Emergence (cốt lõi)
```
Máy tự suy ra vai trò:
| Thành phần | Vai trò             |
| ---------- | ------------------- |
| cnn        | biến (subject-like) |
| hiểu       | trung tâm           |
| dữ liệu    | biến (object-like)  |
KHÔNG cần định nghĩa trước
```
 
### Concept Formation
```
👉 Từ pattern:

X là Y

→ Y xuất hiện nhiều → trở thành khái niệm

Ví dụ:

mô hình học sâu
hệ thống
thuật toán
```

### Behavior Emergence
```
Từ nhiều pattern:

X là Y
X hiểu Y
X dẫn đến Y

→ nhóm lại:

Pattern	Behavior
X là Y	định danh
X hiểu Y	nhận thức
X dẫn đến Y	nguyên nhân

👉 Behavior không định nghĩa trước – mà tự xuất hiện
```

### Relation Induction
```
Từ pattern → quan hệ
| Pattern       | Relation |
| ------------- | -------- |
| X là Y        | IS_A     |
| X dẫn đến Y   | CAUSES   |
| X ảnh hưởng Y | AFFECTS  |
```
### Knowledge Graph
```
Kết quả cuối:

cnn --[không hiểu]--> dữ liệu ít
```
 
||===> Knowledge Graph cơ bản

### Demo
```
emty
```


## B. Thành phần nâng cao: (optinal)

# 2. Chương trình
#### 📂Thư mục myapp: 
```
kg/
├── data/
│   ├── train_cases.json
│   ├── eval_cases.json
│   └── generated_cases.json
│
├── teachers/
│   ├── vncorenlp_teacher.py
│   ├── underthesea_teacher.py
│   ├── embedding_teacher.py
│   └── ai_generator.py
│
├── memory/
│   └── memory_store.py
│
├── pattern/
│   └── pattern_learner.py
│
├── evaluation/
│   └── evaluation_gate.py
│
├── state/
│   └── state_machine.py
│
├── knowledge/
│   └── knowledge_store.py
│
└── main.py
```

#### Giai đoạn hiện tại:
v1
- Đang ở bước test 
- Đang đến step1
```
🔥 STEP 1 (quan trọng nhất)

👉 normalize pattern

X hiểu dữ liệu X → X hiểu Y
🔥 STEP 2

👉 detect phrase (chunking)

"dữ liệu lớn" = 1 unit
🔥 STEP 3

👉 clustering verb

hiểu ~ xử lý ~ phân tích
🔥 STEP 4

👉 logic (if-then)
```
