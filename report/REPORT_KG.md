# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Nguyễn Tiến Đạt&#x20;

**MSSV:** 2A202602606 &#x20;

**Ngày:** 2026-10-05

## 1. Chi phí

Số liệu dưới đây được giữ nguyên từ `ket_qua_benchmark_kg.txt` (OpenAI `gpt-4o-mini`, embedding `text-embedding-3-small`, `top_k=3`, `chunk_size=800`, 176 chunks, graph 204 nodes / 383 relationships).

### Indexing (one-off)

```text
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     54.3
graph       196     91958     4790   0.00938    122.4
```

### Querying (mean per question)

```text
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00       694       47   0.00013     1.55
graph       0.83   1.67      4529       92   0.00073     2.24
```

| Chỉ số           | Flat    | Graph   | Graph / Flat |
| ---------------- | ------- | ------- | ------------ |
| Indexing USD     | 0.00112 | 0.00938 | ×8.38        |
| Indexing giây    | 54.3    | 122.4   | ×2.25        |
| Mỗi câu: USD     | 0.00013 | 0.00073 | ×5.62        |
| Mỗi câu: giây    | 1.55    | 2.24    | ×1.45        |
| Mỗi câu: in\_tok | 694     | 4529    | ×6.53        |

Chi phí indexing tăng do GraphRAG gọi LLM để trích xuất thực thể/vụ án từ 20 bài báo: 196 calls so với 176 calls của Flat RAG, thêm 4.790 output tokens. Khi hỏi, context graph dài hơn làm input tăng từ 694 lên 4.529 tokens/câu; đổi lại GraphRAG đưa được dữ kiện xuyên hai KB.

## 2. Từng câu hỏi

| Câu | Loại               | Flat recall / judge | Graph recall / judge | Thắng                        | Vì sao                                                                                                               |
| --- | ------------------ | ------------------- | -------------------- | ---------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| Q1  | single-hop-law     | 1.00 / 2            | 1.00 / 2             | Hòa                          | Cả hai đều lấy đúng định nghĩa tiền chất; graph bổ sung nhãn Điều 2 khoản 4.                                         |
| Q2  | single-hop-news    | 1.00 / 2            | 1.00 / 2             | Hòa                          | Cả hai đều nêu đúng Trần Thanh Tuấn và Trần Minh Tâm.                                                                |
| Q3  | cross-kb           | 0.00 / 0            | 1.00 / 2             | Graph                        | Graph nối Lê Minh Thành với Điều 251 và khoản 1; Flat không đủ dữ kiện liên KB.                                      |
| Q4  | cross-kb           | 0.00 / 0            | 0.67 / 1             | Graph                        | Graph nhận diện đúng hành vi nhưng chỉ lấy khung khoản 1, thiếu mức tối đa.                                          |
| Q5  | cross-kb-multi-hop | 0.60 / 1            | 1.00 / 2             | Graph                        | Graph nối được Cái Quang Huy, MDMA, Điều 250 khoản 4 và tử hình; Flat thiếu Ketamine và gọi sai “khoản b”.           |
| Q6  | aggregation        | 0.00 / 1            | 0.33 / 1             | Graph theo recall; judge hòa | Graph tìm được một phần vụ MDMA nhưng bỏ sót vụ Pháp y; Flat nêu ba vụ nhưng tên thực thể không khớp `must_include`. |

GraphRAG đạt recall trung bình 0.83 so với 0.43 và judge trung bình 1.67 so với 1.00. Lợi ích tập trung ở các câu cross‑KB (Q3–Q5); hai pipeline ngang nhau ở câu hỏi single-hop (Q1–Q2).

## 3. Phân tích lỗi

### Lỗi E2: Thiếu ngữ cảnh luật cho câu hỏi về mức phạt tối đa

- **Hiện tượng:** Q4 hỏi mức phạt tù tối đa. GraphRAG trả lời “khung cao nhất … 7 năm theo Điều 255 khoản 1”, trong khi đáp án chuẩn là “20 năm hoặc tù chung thân”. Vì vậy Q4 chỉ đạt recall `0.67`, judge `1`.
- **Bằng chứng:** trích từ `ket_qua_benchmark_kg.txt`:

```text
--- Q4 [cross-kb] graph recall=0.67 judge=1
Giang hồ 'Hoàng Nato' bị bắt về hành vi tổ chức sử dụng trái phép chất ma túy. Hành vi này có thể bị phạt tù tối đa 7 năm theo Điều 255 BLHS khoản 1.
```

Đường truy hồi hiện tại của KG‑3 giữ khoản 1 và các khoản `MENTIONS` chất mà vụ án `INVOLVES`. Với câu hỏi Q4 không nêu chất cụ thể, các khoản khung cao hơn không được đưa vào prompt.

- **Nguyên nhân:** lỗi ở chiến lược lọc Cypher của KG‑3, không phải ở `link_entity`. Quy tắc “khoản 1 + khoản có chất liên quan” phù hợp câu hỏi khung cơ bản nhưng không đủ cho từ khóa “tối đa”, “cao nhất”.
- **Đề xuất sửa:** phát hiện các từ khóa `tối đa`, `cao nhất`, `khung cao nhất`; khi có thì lấy toàn bộ khoản có `penalty`, hoặc ít nhất khoản có số lớn nhất của Điều luật. Đánh đổi là thêm facts, input tokens và latency cho các câu hỏi về mức phạt.

### Lỗi E4: Phép đo keyword recall thấp hơn chất lượng thực tế

- **Hiện tượng:** Q6 Flat có recall `0.00` nhưng judge `1`; Graph có recall `0.33` và cũng judge `1`. Cả hai câu trả lời đều có một phần thông tin đúng, nhưng bộ `must_include` yêu cầu chuỗi tên chính xác.
- **Bằng chứng:** Flat trả lời:

```text
Q6 flat recall=0.00 judge=1
Cả ba vụ việc trong tin tức đều có liên quan đến ma túy MDMA. Cụ thể:
1. Vụ việc của Đức liên quan đến số viên nén hình tam giác màu hồng - xám được xác định là MDMA.
2. Vụ việc của Thành liên quan đến 5 viên nén màu trắng được xác định là ma túy MDMA.
3. Vụ việc của Đông liên quan đến 0,686g ma túy MDMA được thu giữ trong buồng chữa bệnh.
```

Trong khi `data/benchmark_kg.json` yêu cầu các chuỗi `Cái Quang Huy`, `Lê Minh Thành`, `Pháp y tâm thần`. Câu trả lời dùng cách gọi rút gọn “Thành”, “Đông” nên recall máy móc bằng 0 dù judge ghi nhận trả lời đúng một phần.

- **Nguyên nhân:** phép đo `keyword_recall()` chỉ kiểm tra substring, không hiểu alias, tên rút gọn hay quan hệ đồng tham chiếu. Judge LLM đánh giá ngữ nghĩa nên hai thước đo có thể mâu thuẫn.
- **Đề xuất sửa:** giữ recall hiện tại để có chỉ số rẻ, tái lập; bổ sung entity-aware recall với alias canonical (ví dụ “Thành” → “Lê Minh Thành”) và chuẩn hóa “Pháp y tâm thần”. Báo cáo nên luôn đọc thêm câu trả lời và judge, không dùng recall đơn độc.

### Lỗi E5: GraphRAG chưa phù hợp với aggregation toàn corpus

- **Hiện tượng:** Q6 hỏi tất cả vụ việc liên quan MDMA, nhưng GraphRAG chỉ trả về hai vụ chính (Cái Quang Huy và vụ Thành), bỏ sót vụ Viện Pháp y tâm thần; đồng thời thêm danh sách Điều 249–252 không phải tên vụ việc.
- **Bằng chứng:**

```text
--- Q6 [aggregation] graph recall=0.33 judge=1
Các vụ việc trong tin tức có liên quan đến ma túy MDMA bao gồm:
1. Vụ vận chuyển ma túy từ Đức về Việt Nam: Cái Quang Huy bị cáo buộc vận chuyển hơn 9,6kg MDMA và Nguyễn Tiến Đạt bị truy cứu trách nhiệm hình sự về gần 4,3kg MDMA.
2. Vụ việc liên quan đến Thành, người đã bị bắt quả tang khi mang 5 viên ma túy MDMA đi bán.
Ngoài ra, trong các điều luật liên quan đến MDMA, có thể tham khảo các điều luật sau: Điều 249, Điều 250, Điều 251, Điều 252.
```

Truy vấn kiểm chứng cho aggregation là:

```javascript
MATCH (k:Case)-[:INVOLVES]->(s:Substance {name:'MDMA'})
RETURN k.name, k.doc_id
ORDER BY k.name;
```

Truy vấn này phải được chạy trên toàn graph; còn GraphRAG hiện chỉ mở rộng các `doc_id` do top‑k vector search trả về. Vì vậy một vụ MDMA ngoài top‑k không đi vào context.

- **Nguyên nhân:** thiết kế KG‑3 ưu tiên “vector hits + graph expansion” để GraphRAG không kém Flat RAG, nhưng aggregation cần truy vấn toàn cục theo node `Substance`, không chỉ các tài liệu seed. Prompt cũng không ràng buộc chặt rằng chỉ liệt kê `Case`.
- **Đề xuất sửa:** nhận diện câu hỏi aggregation và chạy truy vấn toàn graph theo `(:Case)-[:INVOLVES]->(:Substance {name:'MDMA'})`, sau đó chỉ đưa tên Case/người vào prompt. Đánh đổi là context có thể lớn; nên giới hạn số Case, hoặc dùng truy vấn thống kê trước rồi lấy chi tiết top kết quả.

## 4. Kết luận

Flat RAG đủ tốt cho câu hỏi single-hop: Q1 và Q2 đều đạt `1.00 / 2` ở cả hai pipeline, trong khi GraphRAG tốn khoảng `×5.62` USD mỗi câu và chậm `×1.45`. GraphRAG đáng dùng cho câu hỏi cần nối tin tức với luật: Q3–Q5 tăng từ Flat `0.00–0.60` lên Graph `0.67–1.00`, và judge trung bình toàn bộ tăng từ `1.00` lên `1.67`. Với aggregation như Q6, cần bổ sung chế độ truy vấn toàn graph; nếu không, Flat/Graph đều chỉ trả lời một phần.

## 5. Tự kiểm

```text
$ .venv/bin/pytest tests/ -q
................................................                         [100%]
48 passed in 0.21s

$ .venv/bin/python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openai:gpt-4o-mini | embedding = openai:text-embedding-3-small
[OK] KG-2 build_graph: 146 node / 289 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 13 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00064. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j đã nộp:

- `report/img/kg_count.png`: Q‑A, 7 label; tổng 204 node (`Clause` 99, `Person` 36, `Article` 18, `Substance` 17, `Case` 14, `Crime` 13, `Location` 7).
- `report/img/kg_cross_kb.png`: Q‑B, đường qua `Crime`; Results overview có `Article`, `Case`, `Crime`, `Person` và các quan hệ `CHARGED_WITH`, `DEFINES`, `INVOLVED_IN`.
- `report/img/kg_my_case.png`: Q‑D với người **Lê Văn Đông**; Results overview có `Person`, `Case`, `Crime`, `Article`, `Substance`, `Location`.

## Vấn đề gặp phải

Không còn lỗi chưa giải quyết trong phạm vi KG‑1 đến KG‑4. Hạn chế đã biết và hướng sửa được ghi ở các mục E2, E4 và E5; graph hiện tại sau `--check` là graph nhỏ, còn benchmark đầy đủ trong `ket_qua_benchmark_kg.txt` được tạo trước đó với 20 bài báo.
