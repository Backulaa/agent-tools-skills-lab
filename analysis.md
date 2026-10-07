# D05 — Bài luyện tập 01: Tra cứu chính sách đúng phiên bản

## 1. Phạm vi thực hiện

- Làm trên bản sao `d05-exercise-01`, giữ nguyên project mẫu.
- Stage 00 dùng để xác định giới hạn khi agent chưa có tool.
- Stage 01 và stage 02 bổ sung `list_files` trong `tools/files.py`, export trong `tools/__init__.py` và đăng ký trong `agent.py`.
- Danh sách tool gửi model gồm `list_files`, `read_file`, `write_file`.
- Stage 02 bổ sung `workspace/skills/refund-policy/SKILL.md` và `references/answer-template.md`.
- Chính sách nằm trong `workspace/data/policies/`; đường dẫn agent dùng là `data/policies/`.
- Không đưa tên file chính sách, nội dung chính sách hay đáp án cố định vào mã nguồn tool hoặc system prompt.
- Môi trường đã chạy được dependencies qua `uv`; model trong các trace là `cx/gpt-6.1-sol`.

## 2. Thiết kế tool và skill

`list_files(path: str)` liệt kê các mục trực tiếp, không duyệt đệ quy. Kết quả thành công có `ok`, `path`, `entries`; mỗi entry có `name`, `path` tương đối workspace và `type` là `file` hoặc `directory`. Các entry được sắp xếp theo tên.

Tool tái sử dụng `_resolve` và `_error` của tool file hiện có. Đường dẫn được resolve và đối chiếu với workspace; không chấp nhận đường dẫn tuyệt đối hoặc đường dẫn thoát workspace. Các mục trong thư mục cũng được kiểm tra qua `_resolve` để xử lý symlink dẫn ra ngoài. Thư mục không tồn tại và đường dẫn trỏ tới file trả lỗi riêng, không trả danh sách rỗng.

Skill có frontmatter `name` và `description`. Khi câu hỏi liên quan đến hoàn tiền, agent đọc skill; nếu thiếu ngày mua, ngày yêu cầu hoặc trạng thái kích hoạt thì hỏi lại trước khi kết luận. Khi đủ dữ liệu, agent đọc reference, liệt kê `data/policies/`, đọc tài liệu tìm được và chọn phiên bản theo phạm vi hiệu lực chứa **ngày mua**. Skill không yêu cầu Bash hay script.

Số ngày là chênh lệch ngày lịch giữa ngày yêu cầu và ngày mua. Bằng đúng giới hạn vẫn đáp ứng điều kiện thời gian. Reference yêu cầu nêu chính sách áp dụng, số ngày, kết luận, lý do, phí khi đủ điều kiện và đường dẫn tài liệu đã đọc thành công.

## 3. Kết quả kiểm tra tool trực tiếp

Các kết quả sau được quan sát khi gọi `list_files.invoke` bằng terminal tại stage 01:

| Đầu vào | Kết quả quan sát | Đánh giá |
| --- | --- | --- |
| `data/policies` | `ok: true`; hai entry là `policy-before-oct.md` và `policy-from-oct.md`, đúng thứ tự tên, có đường dẫn tương đối và `type: file` | Đạt |
| `data/policies/policy-before-oct.md` | `ok: false`, mã `NOT_A_DIRECTORY` | Đạt |
| `data/does-not-exist` | `ok: false`, mã `DIRECTORY_NOT_FOUND` | Đạt |
| `../dummy-outside` | `ok: false`, mã `PATH_OUTSIDE_WORKSPACE` | Đạt |
| Đường dẫn tuyệt đối lấy từ `paths.WORKSPACE_DIR / 'data'` | `ok: false`, mã `PATH_OUTSIDE_WORKSPACE` | Đạt |

Lệnh kiểm tra đăng ký tại stage 01 trả `['list_files', 'read_file', 'write_file']`. Các snapshot `model_request` tại stage 02 cũng chứa schema của cả ba tool.

Giới hạn kiểm chứng: chưa chạy kiểm thử trực tiếp symlink/junction trên Windows. Các kết quả terminal trên không được ghi là sự kiện trong trace hội thoại.

## 4. Stage 00 và stage 01

### Stage 00 — trường hợp A

- Câu hỏi: mua 28/09/2026, yêu cầu hoàn 06/10/2026, chưa kích hoạt.
- Conversation ID: `948f9fe4faa745c0a8cda5d114b29033`.
- Một lần gọi model, không có tool call. Snapshot tại sự kiện `sequence: 2` có `tools: []`.
- Agent tính được 8 ngày nhưng chưa thể kết luận, yêu cầu cung cấp chính sách bên bán. Các mốc thời gian giả định trong câu trả lời không phải kết quả đọc tài liệu.
- Khả năng còn thiếu: tìm tài liệu trong workspace, đọc chính sách thực tế và xác định phiên bản áp dụng.

### Stage 01 — trường hợp A

- Conversation ID: `0c233b59a77149ffa3a9476c1ec41daa`.
- Sáu lần gọi model, năm tool call. Agent liệt kê lần lượt `.`, `data`, `data/policies`, rồi đọc hai chính sách.
- Bằng chứng: các `tool_finished` tại sequence 5, 9, 13 là kết quả liệt kê; sequence 17 đọc chính sách cũ, sequence 21 đọc chính sách mới, đều thành công.
- Kết quả: chọn chính sách cũ, tính 8 ngày và kết luận không đủ điều kiện vì vượt thời hạn 7 ngày.
- Điểm còn thiếu trong câu trả lời của lần chạy này: chưa dẫn đường dẫn tài liệu. Stage 02 bổ sung yêu cầu dẫn nguồn trong reference và đã đáp ứng ở các lần chạy bên dưới.

## 5. Các trace stage 02

Các file sau nằm dưới `stage-02-skills/traces/`:

| Trường hợp | File JSONL | Conversation ID |
| --- | --- | --- |
| A, trước đổi tên | `20261007-230438_852752a4_turn01_dbfa1e69.jsonl` | `852752a47b40419eb30ec3e6be954fc2` |
| B, trước đổi tên | `20261007-230916_e54b754f_turn01_114400c0.jsonl` | `e54b754fe0c149f1a3ffc578242eeb83` |
| A, sau đổi tên | `20261007-231203_5ae300a9_turn01_01bac360.jsonl` | `5ae300a942f7431e9eaf26587f9b925d` |
| B, sau đổi tên | `20261007-231504_00a7b051_turn01_7952c67d.jsonl` | `00a7b051d8934cba8f33657c024d0bc4` |
| Thiếu trạng thái kích hoạt | `20261007-231750_274af3e6_turn01_4cab4578.jsonl` | `274af3e6c4da4f309c4beccda2f7202e` |

JSONL dùng để đối chiếu các sự kiện gọi tool và snapshot. Câu trả lời cuối được kiểm tra trong **Conversation JSON xuất từ cùng Conversation ID**, ở message assistant cuối cùng (`messages[-1].content`). JSONL của lab không lưu toàn văn câu trả lời cuối; sự kiện `run_completed` chỉ có số ký tự của câu trả lời.

### Vị trí bằng chứng chung cho bốn lần chạy A/B

Bốn trace A/B ở bảng trên đều có sáu lần gọi model, năm tool call và 24 sự kiện. Tra theo trường `sequence` và `type`, không nhầm với chỉ số dòng bắt đầu từ 0:

| Sequence | Bằng chứng |
| --- | --- |
| 2 | `model_request` đầu tiên có `snapshot.message_count: 1`, chỉ có câu hỏi mới, không có kết quả đọc file từ hội thoại cũ |
| 5 | `tool_finished`: `read_file` đọc `skills/refund-policy/SKILL.md`, `result.ok: true` |
| 6 | `model_request`: nội dung skill đã có trong lịch sử gửi model |
| 9 | `tool_finished`: đọc `skills/refund-policy/references/answer-template.md`, `result.ok: true` |
| 10 | `model_request`: nội dung reference đã có trong lịch sử gửi model |
| 13 | `tool_finished`: `list_files` liệt kê `data/policies`, trả các tên file thực tế |
| 17 | `tool_finished`: đọc tài liệu chính sách cũ thành công |
| 21 | `tool_finished`: đọc tài liệu chính sách mới thành công |
| 22 | `model_request`: có nội dung skill, reference và các tài liệu vừa đọc trước khi model trả lời cuối |

## 6. Kết quả các trường hợp stage 02

| Trường hợp | Kết quả quan sát | Tài liệu được dẫn trong câu trả lời | Đánh giá |
| --- | --- | --- | --- |
| A: mua 28/09/2026, yêu cầu 06/10/2026, chưa kích hoạt | Chính sách cũ; 8 ngày; vượt giới hạn 7 ngày; không đủ điều kiện; phí không áp dụng | `data/policies/policy-before-oct.md` | Đạt |
| B: mua 02/10/2026, yêu cầu 12/10/2026, chưa kích hoạt | Chính sách mới; 10 ngày; trong giới hạn 14 ngày; đủ điều kiện; không thu phí | `data/policies/policy-from-oct.md` | Đạt |
| A sau đổi tên | Vẫn chọn chính sách cũ; 8 ngày; không đủ điều kiện | `data/policies/archive-a.md` | Đạt |
| B sau đổi tên | Vẫn chọn chính sách mới; 10 ngày; đủ điều kiện; không thu phí | `data/policies/terms-b.md` | Đạt |
| Thiếu trạng thái kích hoạt | Agent hỏi sản phẩm/dịch vụ đã kích hoạt chưa; chưa kết luận điều kiện hoàn tiền, không tự giả định chưa kích hoạt | Đọc skill để xác định thông tin cần hỏi | Đạt |

### Kiểm tra đổi tên file

Ở stage 02, đổi `policy-before-oct.md` thành `archive-a.md`, đổi `policy-from-oct.md` thành `terms-b.md`, giữ nguyên nội dung. Hai lần chạy sau đổi tên đều bắt đầu cuộc trò chuyện mới và có Conversation ID khác các lần trước.

Sequence 13 trong hai trace sau đổi tên trả chính các tên mới. Sequence 17 và 21 đọc `archive-a.md` và `terms-b.md` thành công. Câu trả lời dẫn tên mới và giữ nguyên kết luận, cho thấy agent tìm tài liệu bằng kết quả tool thay vì phụ thuộc tên file cũ hoặc kết quả đọc trong lịch sử trước.

### Kiểm tra thiếu thông tin

Câu hỏi: “Tôi mua ngày 02/10/2026, muốn hoàn ngày 12/10/2026.”

Trace `20261007-231750_274af3e6_turn01_4cab4578.jsonl` có hai lần gọi model, một tool call và tám sự kiện. Sequence 5 đọc skill thành công; sequence 6 đưa nội dung skill vào context trước khi trả lời. Agent hỏi: “Sản phẩm/dịch vụ bạn mua đã được kích hoạt chưa?” và yêu cầu thông tin để kiểm tra điều kiện. Conversation JSON được xuất ngay sau câu hỏi làm rõ, trước khi cung cấp trạng thái kích hoạt.

Việc dừng để hỏi tại đây phù hợp với bước kiểm tra dữ liệu đầu vào của skill; chưa cần đọc reference hay chính sách để đưa ra kết luận.

## 7. Trả lời câu hỏi cuối bài

**Vì sao cần tool tìm file và skill chọn chính sách?** Tool cung cấp khả năng thao tác với dữ liệu thật: liệt kê tên file hiện có và đọc nội dung tài liệu. Skill cung cấp quy trình: kiểm tra dữ liệu khách hàng, tìm và đọc chính sách, chọn theo ngày mua, tính số ngày, đối chiếu điều kiện và trả lời có căn cứ. Chỉ có tool thì agent đã truy cập được dữ liệu nhưng chưa được hướng dẫn đầy đủ cách áp dụng chính sách; skill giúp quy trình rõ ràng và có thể tái sử dụng.

**Nếu chưa có tool tìm file, sửa prompt có giải quyết được yêu cầu đổi tên không?** Không, nếu agent cũng không có khả năng khám phá tương đương. Prompt có thể yêu cầu tìm tài liệu nhưng không tự cung cấp quyền truy cập danh sách file. `read_file` chỉ đọc được khi đã biết đường dẫn; ghi cố định đường dẫn trong prompt sẽ mất tác dụng khi file đổi tên. `list_files` cho agent biết đường dẫn thực tế, sau đó `read_file` và skill mới xử lý được tài liệu phù hợp.

## 8. Hồ sơ nộp bài

- Mã nguồn `tools/files.py`, `tools/__init__.py`, `agent.py` đã bổ sung và đăng ký tool ở cả stage 01 và stage 02.
- Hai tài liệu chính sách của mỗi stage; stage 02 dùng các tên đã đổi để thể hiện bài kiểm tra.
- Thư mục `stage-02-skills/workspace/skills/refund-policy/` gồm skill và reference.
- Các trace JSONL của bài tập và Conversation JSON tương ứng; gồm bằng chứng A, B sau đổi tên và trường hợp thiếu thông tin.
- File `analysis.md` này.
- Không đưa `.env` hoặc API key vào hồ sơ nộp bài.

