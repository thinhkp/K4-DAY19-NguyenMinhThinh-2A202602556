# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Ngày chạy:** 05-10-2026
**Thiết lập:** OpenRouter `openai/gpt-4o-mini` (chat), `openai/text-embedding-3-small` (embedding), `top_k=3`, `chunk_size=800`, 176 chunks. Neo4j chứa 199 node và 380 quan hệ.

Đã dùng inference API key OpenRouter hợp lệ trong `.env`; benchmark và extraction thật chạy thành công. Key chỉ được đọc từ môi trường, không ghi vào báo cáo. Trước đó key lấy từ trang Management API Keys bị 401 `User not found`; sau khi thay bằng inference key, cả `--check` và `--judge` đều hoàn tất. Không đưa `.env`/key vào repo.

## 1. Chi phí và hiệu năng

Số dưới đây được chép từ `ket_qua_benchmark_kg.txt` do `python bench_kg.py --judge` sinh ra. Graph/Flat được tính từ số trung bình đã in (làm tròn).

### Indexing (one-off)

| Pipeline | Calls | Input tokens | Output tokens | USD | Giây | Graph / Flat |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Flat | 176 | 56,072 | 0 | 0.00112 | 101.2 | 1× |
| Graph | 196 | 91,958 | 4,594 | 0.00926 | 196.5 | 8.27× chi phí; 1.94× thời gian |

### Querying (trung bình mỗi câu)

| Pipeline | Recall | Judge | Input tokens | Output tokens | USD | Giây | Graph / Flat |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Flat | 0.43 | 1.00 | 694 | 47 | 0.00013 | 2.55 | 1× |
| Graph | 0.89 | 1.83 | 5,855 | 77 | 0.00092 | 3.40 | 7.08× chi phí; 1.33× thời gian |

Phần tăng chi phí indexing chủ yếu đến từ 20 lần gọi LLM bổ sung để trích xuất graph: 4,594 output tokens và tổng input token tăng 35,886. Embedding 176 chunks dùng chung. Khi truy vấn, Graph gửi trung bình nhiều hơn 5,161 input tokens để đưa dữ kiện graph vào prompt. Graph cải thiện recall trung bình 0.46 và judge 0.83 điểm, đổi lại tốn thêm khoảng $0.00079/câu. Chỉ xét chi phí query, phần tăng thêm indexing $0.00814 cần khoảng 11 câu hỏi để bù (0.00814 / 0.00079); phép tính này giả định mỗi câu Graph thay một câu Flat và không tính khác biệt chi phí xây dựng/vận hành Neo4j.

## 2. Kết quả từng câu

| Câu | Loại | Flat recall / judge | Graph recall / judge | Kết quả | Bằng chứng/tóm tắt |
| --- | --- | ---: | ---: | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai nêu đúng định nghĩa tiền chất; Graph chỉ rõ Điều 2 khoản 4 Luật PCMT. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai tìm đúng Trần Thanh Tuấn và Trần Minh Tâm nhận án tử hình. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Flat trả “Không đủ thông tin”; Graph nối Lê Minh Thành → mua bán trái phép → Điều 251 và khung khoản 1. |
| Q4 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Flat trả “Không đủ thông tin”; Graph nêu Điều 255 khoản 4, tối đa 20 năm hoặc chung thân. |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 1.00 / 2 | Graph | Flat tìm đúng Huy/MDMA nhưng ghi “khoản b)”; Graph xác định Điều 250 khoản 4 và hình phạt. |
| Q6 | aggregation | 0.00 / 1 | 0.33 / 1 | Graph một phần | Cả hai trả lời một phần; Graph liệt kê bốn vụ, nhưng bỏ sót một vụ MDMA có liên kết trong KG (xem E5). |

Graph vượt Flat rõ ở các câu cần nối nhiều tài liệu (Q3–Q5). Hai câu đơn lẻ Q1–Q2 hòa. Q6 cho thấy có graph không tự đảm bảo câu trả lời tổng hợp đầy đủ: bước chọn lọc/tóm tắt của agent vẫn có thể bỏ sót node.

## 3. Phân tích lỗi

### E5 — Câu tổng hợp MDMA bỏ sót một vụ đã có trong graph

- **Hiện tượng:** Graph trả lời Q6 liệt kê Sầm Sơn, Cái Quang Huy, Hà Nội và vụ vận chuyển từ Đức; bỏ sót vụ tại Viện Pháp y tâm thần Trung ương. Recall Q6 chỉ 0.33 và judge 1.
- **Bằng chứng:** mục `Q6 [aggregation] graph` trong benchmark liệt kê bốn vụ kể trên. Truy vấn trên graph đầy đủ:

```cypher
MATCH (k:Case)-[:INVOLVES]->(s:Substance)
WHERE toLower(s.name) CONTAINS 'mdma'
RETURN k.name AS case, s.name AS substance ORDER BY case;
```

Trả về năm hàng: `Vụ góp tiền mua ma túy tại Hà Nội | MDMA`; `Vụ tổ chức sử dụng ma túy tại Sầm Sơn | MDMA`; `Vụ vận chuyển ma túy của Cái Quang Huy | MDMA`; `Vụ vận chuyển ma túy từ Đức về Việt Nam | MDMA`; `Vụ án tại Viện Pháp y tâm thần Trung ương | MDMA`. Hàng Viện Pháp y là case có cạnh `INVOLVES` trong KG nhưng bị thiếu khỏi danh sách trả lời Q6.
- **Nguyên nhân:** Agent đưa dữ kiện truy xuất vào prompt rồi sinh câu trả lời tự do; prompt/ranking không buộc liệt kê đầy đủ mọi Case. Kết quả cho thấy khâu tổng hợp/grounding chưa bảo toàn tính exhaustive, dù graph có cạnh liên quan.
- **Đề xuất sửa:** Với câu hỏi dạng “liệt kê tất cả”, chạy truy vấn Cypher trực tiếp từ Substance sang toàn bộ Case, trả danh sách có kiểm tra số hàng; yêu cầu LLM chỉ diễn đạt kết quả và kiểm tra mọi tên Case trước khi hoàn tất.

### E6 — `charge` rỗng lẫn trường hợp “không có tội danh ma túy” và “không trích xuất được”

- **Hiện tượng:** Có sáu quan hệ `INVOLVED_IN` với `charge=''` trong graph đầy đủ; chỉ nhìn thuộc tính này không biết đó là thiếu trích xuất hay người được nêu trong bài nhưng không bị cáo buộc tội ma túy.
- **Bằng chứng Cypher và kết quả:**

```cypher
MATCH (p:Person)-[r:INVOLVED_IN]->(k:Case)
WHERE r.charge = ''
RETURN p.name, r.role, k.name ORDER BY k.name;
```

Trả về sáu người: Cao Thị Bích Hằng, Ngô Việt Dũng, Trần Quốc An (vụ Sầm Sơn), cùng Nguyễn Văn Quang, Ngô Văn Vinh, Trần Văn Trường (vụ Viện Pháp y). Bài nguồn `data/drug_news/news-100260924105118645.md` nêu các cáo buộc hối lộ/đánh bạc/tham nhũng đối với nhóm Viện Pháp y, nhưng đây không phải tội ma túy trong tập 18 điều luật của bài lab. Vì vậy, các chuỗi rỗng này không thể kết luận đều là lỗi; một phần đến từ phạm vi schema chỉ chuẩn hóa tội ma túy.
- **Nguyên nhân:** Pipeline dùng chuỗi rỗng cho nhiều trạng thái khác nhau; ontology chỉ nối `Crime` đã chuẩn hóa từ kho luật ma túy. Charge ngoài tập này không có node chuẩn, còn “không nêu” và “không trích được” chưa được phân biệt.
- **Đề xuất sửa:** Lưu `charge_raw` theo văn bản nguồn; tách `charge_status` (nêu rõ/không nêu/không trích xuất/ngoài phạm vi) và chỉ tạo cạnh tới Crime chuẩn khi có căn cứ. Có thể mở rộng ontology bằng nguồn luật bổ sung nếu lab cần tra cứu các cáo buộc ngoài ma túy.

### E2 — Truy vấn mức hình phạt tối đa từng bỏ khoản liên quan, đã sửa

Trước khi sửa, context lấy khoản 1 và khoản gắn với chất trong vụ; với câu hỏi tối đa cho vụ không có Substance, khoản có mức tối đa cao hơn có thể bị bỏ. Đã xác minh trên Neo4j fixture: hỏi “mức hình phạt cao nhất” trả các khoản Điều 255 `[1, 2, 3, 4, 5]`, trong đó khoản 4 có 20 năm hoặc chung thân; hỏi “khung hình phạt cơ bản” vẫn chỉ trả khoản `[1]`. Đã sửa `context()` để nhận diện câu hỏi tối đa/cao nhất và lấy toàn bộ khoản của Article liên quan. Đây là kiểm tra Cypher/context, không phải kết luận pháp lý hay so sánh mọi khung hình phạt trong toàn bộ pháp luật.

## 4. Kết luận

Trong bộ sáu câu này, GraphRAG có recall trung bình **0.89 so với 0.43** và judge **1.83 so với 1.00**, chủ yếu nhờ ba câu hỏi cross-KB. Với câu đơn lẻ Q1–Q2, Flat ngang điểm mà rẻ và nhanh hơn. Graph phù hợp khi bộ câu hỏi thường xuyên cần nối người–vụ–tội danh–điều luật, hoặc khi cùng một graph phục vụ đủ nhiều lượt hỏi để bù chi phí dựng. Với chỉ sáu câu như phép đo này, chi phí indexing Graph cao hơn 0.00814 USD; mức tiết kiệm/chi phí tăng thêm ở query cho điểm hòa vốn khoảng 11 câu, trong giả định nêu ở mục 1. Với yêu cầu liệt kê đầy đủ như Q6, cần truy vấn tổng hợp có kiểm đếm thay vì chỉ dựa vào câu trả lời sinh tự do.

## 5. Tự kiểm

```text
> python -m pytest tests/ -q
................................................                         [100%]
48 passed

> python -m compileall -q src
Hoàn tất, không có lỗi cú pháp.

> python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openrouter:openai/gpt-4o-mini | embedding = openrouter:openai/text-embedding-3-small
[OK] KG-2 build_graph: 146 node / 289 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 13 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00064. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

`python bench_kg.py --judge` cũng đã chạy thành công; benchmark đầy đủ nằm tại `ket_qua_benchmark_kg.txt`. Graph thật có 7 label (`Clause`, `Person`, `Article`, `Substance`, `Case`, `Crime`, `Location`) và 7 kiểu quan hệ (`CHARGED_WITH`, `DEFINES`, `HAS_CLAUSE`, `INVOLVED_IN`, `INVOLVES`, `LOCATED_IN`, `MENTIONS`), khớp `ONTOLOGY.md`.

### Ảnh Neo4j cần bổ sung

Chưa lưu được ba ảnh đúng quy cách vào `report/img/`: yêu cầu bài lab cần thấy ô truy vấn và Results overview. Ảnh xem trong browser không thể xuất thành file bằng công cụ hiện có; cần chụp màn hình trực tiếp và đặt tên `kg_count.png`, `kg_cross_kb.png`, `kg_my_case.png`. Tên người tự chọn cho Q-D: Cái Quang Huy. Neo4j hiện vẫn chạy để có thể chụp các kết quả.

## Tự chấm

**Ước lượng: 95/100** theo thang `SUBMISSION.md`: code/tests 30/30, ontology 15/15, benchmark 5/5, báo cáo chi phí 10/10, từng câu 10/10, hai lỗi có bằng chứng 20/20, kết luận 5/5, ảnh 0/5. Đây là tự chấm theo tiêu chí trong repo, không phải điểm chính thức; ba ảnh còn thiếu có thể làm mất tối đa 5 điểm.

## Trạng thái nộp

Đã hoàn tất code, ontology, `--check`, benchmark và báo cáo; các file hiện có đã được push lên repo cá nhân [K4-DAY19-NguyenMinhThinh-2A202602556](https://github.com/thinhkp/K4-DAY19-NguyenMinhThinh-2A202602556). Chưa có ảnh Neo4j lưu trong `report/img/` và chưa nộp link lên VLearn.
