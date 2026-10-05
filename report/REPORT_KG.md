# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Hoàng Văn Sơn  **MSSV:** 2A202602375  **Ngày:** 2026-10-05

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    217.9
graph       196     34619     5581   0.00569    332.0

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.33      696       73   0.00010     2.11
graph       0.89   1.67     5704      169   0.00064     2.71
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | 0.00000 | 0.00569 | ∞ |
| Indexing giây | 217.9 | 332.0 | ×1.5 |
| Mỗi câu: USD | 0.00010 | 0.00064 | ×6.4 |
| Mỗi câu: giây | 2.11 | 2.71 | ×1.3 |
| Mỗi câu: in_tok | 696 | 5704 | ×8.2 |

**Chi phí tăng thêm đến từ đâu?** (2–3 câu)
> Chi phí Indexing tăng là do phải tốn thêm nhiều lượt gọi LLM để đọc, trích xuất dữ kiện (entity, relationship) từ các bài báo (khoảng 196 calls). Chi phí Querying tăng (cả thời gian, lượng token và tiền) là do hàm truy vấn GraphRAG cần lôi thêm một lượng lớn ngữ cảnh (facts) từ KG (Graph Context) nhồi vào prompt, làm tăng số `in_tok` (từ 696 lên 5704 tokens mỗi câu).

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Câu hỏi tra cứu định nghĩa luật đơn giản, có sẵn trong 1 chunk nên Flat tìm ra ngay. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Tên người bị bắt nằm chung trong một bài báo ngắn, Vector Search của Flat đủ sức lấy được. |
| Q3 | cross-kb | 0.33 / 1 | 0.67 / 1 | Graph | GraphRAG dò từ Vụ Án (News) qua Tội Danh để nối sang Điều Luật dễ dàng, trong khi Flat bị đứt đoạn vì không có keyword chung. |
| Q4 | cross-kb | 0.33 / 1 | 0.67 / 1 | Graph | Flat RAG thất bại do không biết móc nối hành vi với điều khoản, GraphRAG đi dọc qua quan hệ CHARGED_WITH và DEFINES. |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | Graph | GraphRAG có khả năng đi qua nhiều node (Vụ án -> Chất -> Khối lượng -> Điều -> Khoản) nên mới tìm được khung hình phạt. |
| Q6 | aggregation | 0.00 / 1 | 1.00 / 2 | Graph | Câu hỏi tổng hợp đòi hỏi liệt kê tất cả vụ án chứa chất MDMA; Graph đếm số cạnh nối tới Node MDMA nhanh gọn, Flat bị trôi chunk. |

## 3. Phân tích lỗi (20 điểm)

Chọn ít nhất 2 nhóm lỗi trong E1–E6 (`LAB_GUIDE.md` Bước 8.4). Sao chép khung dưới đây cho mỗi lỗi.

### Lỗi E1: Lỗi tách thực thể (Entity Resolution)

- **Hiện tượng:** Cùng một tội danh hoặc chất nhưng bị LLM ghi ra dưới 2 cái tên hơi khác nhau (Ví dụ: "mua bán trái phép chất ma túy" và "tội mua bán trái phép chất ma tuý").
- **Bằng chứng:** Trong hàm KG-1, chúng ta đã phải dùng hàm `difflib.get_close_matches` với `cutoff=0.8` và chuẩn hóa chuỗi `normalize_crime` để ép chúng về làm 1.

```cypher
MATCH (c:Crime) RETURN c.name
```

```
kết quả: "tội mua bán trái phép chất ma túy"
```

- **Nguyên nhân:** Nhà báo viết tắt hoặc viết sai chính tả một vài từ so với văn bản Luật.
- **Đề xuất sửa:** Viết thêm hàm chuẩn hóa mạnh hơn hoặc dùng một prompt mồi LLM kiểm tra kỹ danh sách `known_entities`.

### Lỗi E5: Ngữ cảnh bị tràn (Context Overflow)

- **Hiện tượng:** Ở câu Q6 (câu hỏi tổng hợp), Graph ném về quá nhiều fact (Điều 249, 250, 251, 252...) khiến số lượng token tăng vọt lên 5704 tokens.
- **Bằng chứng:** Ở bảng thống kê trung bình, Graph in_tok = 5704, trong khi Flat in_tok = 696.

```cypher
MATCH (a:Article)-[:HAS_CLAUSE]->(cl:Clause)-[:MENTIONS]->(s:Substance {name: "mdma"})
RETURN COUNT(cl)
```

```
kết quả: Trả về hàng chục Clause.
```

- **Nguyên nhân:** Câu hỏi chạm vào "siêu nút" (supernode) là chất MDMA, chất này được kết nối với quá nhiều vụ án và vô vàn khung hình phạt trong luật.
- **Đề xuất sửa:** Cần giới hạn `max_facts` hợp lý hoặc cho điểm (score) các fact bằng Vector Search trước khi đưa vào context để lọc bớt.

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2.
> - **Nên dùng Flat RAG khi:** Hệ thống trả lời các câu hỏi tra cứu thông tin trực tiếp (Q1, Q2) với chi phí thấp ($0.00010/câu), tốc độ nhanh, ít token.
> - **Nên dùng GraphRAG khi:** Cần trả lời các câu hỏi phức tạp đòi hỏi phải xâu chuỗi thông tin (Q3, Q4, Q5) hoặc câu hỏi tổng hợp (Q6) (Graph đạt recall 1.00 ở Q5, Q6). Tuy nhiên, phải chấp nhận chi phí cao gấp 6 lần và hệ thống Indexing phức tạp hơn.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                                                                       [100%]
48 passed in 0.27s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 23 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00055. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: Cái Quang Huy

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: Không có. Mọi lỗi (Quota Limit, Unicode) đã được xử lý bằng cách thêm `time.sleep` và đổi Encoding sang `utf-8`.
> Không có.
