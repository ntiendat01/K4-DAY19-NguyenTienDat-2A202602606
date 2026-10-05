# Thiết kế Ontology — Day 19

**Họ tên: Nguyễn Tiến Đạt** &#x20;

**MSSV:** 2A202602606

**Lựa chọn** (đánh dấu một):

- Dùng ontology gợi ý (có thể chỉnh nhỏ)
- Tự thiết kế (xét bonus +15, xem `SUBMISSION.md`)

> Hướng dẫn: `LAB_GUIDE.md` Bước 2. Dùng ontology gợi ý thì vẫn phải điền đủ các mục dưới đây bằng lời của bạn.

## 1. Sơ đồ

Vẽ bằng mermaid (hoặc chèn ảnh `report/img/ontology.png`). Đánh dấu rõ **node cầu nối**.

```javascript
flowchart LR
    P[Person] -- "INVOLVED_IN\nrole, charge, sentence" --> K[Case]
    K -- CHARGED_WITH --> C((Crime - bridge))
    K -- "INVOLVES\namount" --> S[Substance]
    K -- LOCATED_IN --> L[Location]
    A[Article] -- DEFINES --> C
    A -- HAS_CLAUSE --> CL[Clause]
    CL -- MENTIONS --> S
```

## 2. Entity types (node labels)

| Label       | Ý nghĩa                         | Khóa định danh (`MERGE` theo) | Properties                                  | Lấy từ KB nào | Trích bằng (regex / LLM / khác)              |
| ----------- | ------------------------------- | ----------------------------- | ------------------------------------------- | ------------- | -------------------------------------------- |
| `Article`   | Một điều luật                   | `id`                          | `title`, `law`, `doc_id`                    | Luật          | Regex                                        |
| `Clause`    | Một khoản trong điều luật       | `id`                          | `number`, `penalty`, `text`, `doc_id`       | Luật          | Regex                                        |
| `Crime`     | Tội danh chuẩn, là node cầu nối | `name`                        | `name`                                      | Cả hai        | Regex từ tiêu đề luật; LLM + liên kết từ tin |
| `Substance` | Chất ma túy                     | `name`                        | `name`                                      | Cả hai        | Danh sách chuẩn/LLM                          |
| `Case`      | Một vụ việc được bài báo mô tả  | `name`                        | `summary`, `date`, `source_title`, `doc_id` | Tin           | LLM JSON                                     |
| `Person`    | Người liên quan vụ án           | `name`                        | `aliases`                                   | Tin           | LLM JSON                                     |
| `Location`  | Địa điểm vụ án                  | `name`                        | `name`                                      | Tin           | LLM JSON                                     |

## 3. Relationships

| Type           | Từ → Đến               | Properties trên cạnh         | Ý nghĩa                          |
| -------------- | ---------------------- | ---------------------------- | -------------------------------- |
| `DEFINES`      | `Article` → `Crime`    | —                            | Điều luật quy định tội danh      |
| `HAS_CLAUSE`   | `Article` → `Clause`   | —                            | Điều luật gồm khoản              |
| `MENTIONS`     | `Clause` → `Substance` | —                            | Khoản nhắc tới chất ma túy       |
| `CHARGED_WITH` | `Case` → `Crime`       | —                            | Vụ việc bị truy tố/xét xử về tội |
| `INVOLVES`     | `Case` → `Substance`   | `amount`                     | Chất và khối lượng trong vụ      |
| `LOCATED_IN`   | `Case` → `Location`    | —                            | Địa điểm vụ việc                 |
| `INVOLVED_IN`  | `Person` → `Case`      | `role`, `charge`, `sentence` | Người và vai trò/mức án trong vụ |

## 4. Node cầu nối giữa 2 KB

- **Node nào:** `Crime`.
- **Vì sao chọn node này:** luật định nghĩa tội danh, còn tin mô tả vụ án bị truy tố/xét xử về tội đó. Đây là khái niệm cùng nghĩa và tạo đường đi ngắn `Case → Crime ← Article`.
- **Cách đảm bảo hai phía khớp tên** (chuẩn hóa, `link_entity`, danh sách chuẩn trong prompt…): `normalize_crime` hạ chữ hoa, bỏ tiền tố “Tội” và chuẩn hoá khoảng trắng ở cả hai phía. Prompt LLM đưa danh sách tội danh canonical; đầu ra vẫn được `link_entity` so khớp exact trước rồi fuzzy với ngưỡng 0,8. Node `Crime` luôn `MERGE` theo tên canonical từ luật.
- **Khi nào cầu gãy, và bạn xử lý thế nào:** cầu gãy khi bài báo nói về hành vi/tội không có trong các điều luật đã nạp hoặc LLM xuất sai quá khác. Khi đó không tạo `CHARGED_WITH`, an toàn hơn nối nhầm; có thể mở rộng corpus/danh sách tội danh hoặc rà prompt và đầu ra JSON.

## 5. Competency questions

Với mỗi câu trong `data/benchmark_kg.json`, ghi đường đi trên graph dùng để trả lời. Câu nào không trả lời được thì ghi rõ lý do.

| Câu | Đường đi (Cypher pattern)                                                                                                                                                                    | Trả lời được?                                                                        |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| Q1  | `(:Article)-[:HAS_CLAUSE]->(:Clause)`; truy hồi bài luật Phòng, chống ma túy từ vector rồi đọc khoản liên quan                                                                               | Có (qua chunk luật; graph không tách riêng định nghĩa ngoài cấu trúc Article/Clause) |
| Q2  | `(:Person)-[:INVOLVED_IN]->(:Case)`                                                                                                                                                          | Có                                                                                   |
| Q3  | `(:Person)-[:INVOLVED_IN]->(:Case)-[:CHARGED_WITH]->(:Crime)<-[:DEFINES]-(:Article)-[:HAS_CLAUSE]->(:Clause {number:1})`                                                                     | Có                                                                                   |
| Q4  | `(:Person {aliases})-[:INVOLVED_IN]->(:Case)-[:CHARGED_WITH]->(:Crime)<-[:DEFINES]-(:Article)-[:HAS_CLAUSE]->(:Clause)`                                                                      | Có                                                                                   |
| Q5  | `(:Person)-[:INVOLVED_IN]->(:Case)-[:INVOLVES]->(:Substance {name:'MDMA'})` và `(:Case)-[:CHARGED_WITH]->(:Crime)<-[:DEFINES]-(:Article)-[:HAS_CLAUSE]->(:Clause)-[:MENTIONS]->(:Substance)` | Có                                                                                   |
| Q6  | `(:Case)-[:INVOLVES]->(:Substance {name:'MDMA'})` (và người/case kề nó để đặt tên vụ)                                                                                                        | Có                                                                                   |

## 6. Quyết định thiết kế và đánh đổi

Ít nhất 3 quyết định. Mỗi quyết định ghi: đã chọn gì, phương án khác là gì, vì sao chọn.

1. Tách `Article` và `Clause` thay vì lưu toàn bộ điều trong một node. Phương án ít node hơn là để khoản trong property mảng; tách node giúp truy hồi đúng khoản và nối từng khoản với chất ma túy, đổi lại graph có nhiều cạnh hơn.
2. Chọn `Crime` làm bridge thay vì nối trực tiếp `Case` với `Article`. Tội danh ổn định và có ý nghĩa pháp lý hơn số điều thường không xuất hiện trong tin; đổi lại phải chuẩn hoá/link tên để tránh cầu gãy.
3. Dùng regex cho luật, LLM JSON cho tin. Regex rẻ, tái lập được với văn bản luật đều cấu trúc; LLM linh hoạt với báo nhưng có chi phí và có thể thiếu/trùng thực thể.
4. Chất, người và địa điểm `MERGE` theo tên để tái dùng liên tài liệu. Cách này giảm trùng nhưng có rủi ro đồng danh; `Case`, `Article`, `Clause` mang `doc_id` nhằm truy ngược văn bản nguồn.

## 7. So với ontology gợi ý (bắt buộc nếu xét bonus)

| Điểm khác                                       | Gợi ý làm gì                        | Bạn làm gì                                           | Vấn đề nó giải quyết                                                   | Bằng chứng (Cypher, hoặc số liệu benchmark)                                                          |
| ----------------------------------------------- | ----------------------------------- | ---------------------------------------------------- | ---------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| Không xét bonus; triển khai đúng ontology gợi ý | Các label/quan hệ trong sơ đồ gợi ý | Giữ nguyên để ưu tiên xây dựng và kiểm thử KG-1/KG-2 | Có bridge `Crime` rõ ràng, phép kiểm có thể tìm đường từ tin sang luật | Chạy `python bench_kg.py --build --limit 2`, sau đó truy vấn đường `Person → Case → Crime ← Article` |

## 8. Hạn chế còn lại

- `Case` và `Person` định danh bằng tên do LLM tạo/trích xuất nên vẫn có thể trùng hoặc tách cùng một người/vụ.
- `Substance` chưa hợp nhất đầy đủ đồng nghĩa/tên lóng (ví dụ “kẹo”); chỉ các tên chuẩn được nối tốt.
- Khối lượng đang là chuỗi trên cạnh `INVOLVES`, chưa được chuẩn hoá đơn vị/số để tự suy luận chính xác khoản luật.
- Tin không nêu rõ tội danh hoặc tội nằm ngoài 18 điều luật nạp sẽ không có cạnh `CHARGED_WITH`, chủ ý tránh suy đoán sai.
