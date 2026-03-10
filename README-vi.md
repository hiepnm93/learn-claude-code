[English](./README.md) | [中文](./README-zh.md) | [日本語](./README-ja.md) | [Tiếng Việt](./README-vi.md)
# Learn Claude Code -- Một agent nano giống Claude Code, xây dựng từ 0 đến 1
<img width="260" src="https://github.com/user-attachments/assets/fe8b852b-97da-4061-a467-9694906b5edf" /><br>

Quét mã QR bằng Wechat để theo dõi chúng tôi,
hoặc theo dõi trên X: [shareAI-Lab](https://x.com/baicai003)


```
                    MẪU AGENT
                    =================

    User --> messages[] --> LLM --> response
                                      |
                            stop_reason == "tool_use"?
                           /                          \
                         yes                           no
                          |                             |
                    thực thi công cụ                 trả về văn bản
                    thêm kết quả
                    lặp lại -----------------> messages[]


    Đó là vòng lặp tối giản. Mọi agent lập trình AI đều cần vòng lặp này.
    Agent trong thực tế bổ sung thêm chính sách, quyền hạn và các lớp vòng đời.
```

**12 phiên học tiến dần, từ một vòng lặp đơn giản đến thực thi tự chủ có cô lập.**
**Mỗi phiên thêm một cơ chế. Mỗi cơ chế có một phương châm.**

> **s01** &nbsp; *"Chỉ cần một vòng lặp và Bash"* &mdash; một công cụ + một vòng lặp = một agent
>
> **s02** &nbsp; *"Thêm công cụ nghĩa là thêm một handler"* &mdash; vòng lặp giữ nguyên; công cụ mới đăng ký vào bảng dispatch
>
> **s03** &nbsp; *"Agent không có kế hoạch sẽ trôi dạt"* &mdash; liệt kê các bước trước, rồi thực thi; hiệu suất hoàn thành tăng gấp đôi
>
> **s04** &nbsp; *"Chia nhỏ tác vụ lớn; mỗi tác vụ con có ngữ cảnh riêng"* &mdash; subagent sử dụng messages[] độc lập, giữ cuộc hội thoại chính sạch sẽ
>
> **s05** &nbsp; *"Tải kiến thức khi cần, không tải trước"* &mdash; đưa vào qua tool_result, không qua system prompt
>
> **s06** &nbsp; *"Ngữ cảnh sẽ đầy; bạn cần cách để giải phóng"* &mdash; chiến lược nén ba lớp cho phiên làm việc vô hạn
>
> **s07** &nbsp; *"Chia mục tiêu lớn thành tác vụ nhỏ, sắp xếp thứ tự, lưu xuống đĩa"* &mdash; đồ thị tác vụ dạng file với quan hệ phụ thuộc, đặt nền tảng cho cộng tác đa agent
>
> **s08** &nbsp; *"Chạy thao tác chậm trong nền; agent tiếp tục suy nghĩ"* &mdash; daemon thread chạy lệnh, đưa thông báo khi hoàn thành
>
> **s09** &nbsp; *"Khi tác vụ quá lớn cho một mình, giao cho đồng đội"* &mdash; đồng đội cố định + hộp thư bất đồng bộ
>
> **s10** &nbsp; *"Đồng đội cần quy tắc giao tiếp chung"* &mdash; một mẫu yêu cầu-phản hồi điều phối mọi đàm phán
>
> **s11** &nbsp; *"Đồng đội tự quét bảng và nhận tác vụ"* &mdash; không cần trưởng nhóm phân công từng việc
>
> **s12** &nbsp; *"Mỗi người làm trong thư mục riêng, không ảnh hưởng lẫn nhau"* &mdash; tác vụ quản lý mục tiêu, worktree quản lý thư mục, liên kết bằng ID

---

## Mẫu Cốt Lõi

```python
def agent_loop(messages):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM,
            messages=messages, tools=TOOLS,
        )
        messages.append({"role": "assistant",
                         "content": response.content})

        if response.stop_reason != "tool_use":
            return

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = TOOL_HANDLERS[block.name](**block.input)
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        messages.append({"role": "user", "content": results})
```

Mỗi phiên học xếp chồng một cơ chế lên trên vòng lặp này -- mà không thay đổi bản thân vòng lặp.

## Phạm Vi (Quan Trọng)

Kho mã nguồn này là một dự án học từ 0 đến 1 để xây dựng một agent nano giống Claude Code.
Nó cố ý đơn giản hóa hoặc bỏ qua một số cơ chế của phiên bản thực tế:

- Bus sự kiện/hook đầy đủ (ví dụ PreToolUse, SessionStart/End, ConfigChange).
  s12 chỉ bao gồm một luồng sự kiện vòng đời tối giản kiểu append-only phục vụ giảng dạy.
- Quản trị quyền dựa trên quy tắc và quy trình tin cậy
- Điều khiển vòng đời phiên (resume/fork) và điều khiển vòng đời worktree nâng cao
- Chi tiết đầy đủ về MCP runtime (transport/OAuth/resource subscribe/polling)

Hãy coi giao thức hộp thư JSONL của team trong repo này như một triển khai phục vụ giảng dạy, không phải mô tả về bất kỳ nội bộ sản phẩm cụ thể nào.

## Bắt Đầu Nhanh

```sh
git clone https://github.com/shareAI-lab/learn-claude-code
cd learn-claude-code
pip install -r requirements.txt
cp .env.example .env   # Chỉnh sửa .env với ANTHROPIC_API_KEY của bạn

python agents/s01_agent_loop.py       # Bắt đầu từ đây
python agents/s12_worktree_task_isolation.py  # Điểm cuối của lộ trình
python agents/s_full.py               # Tổng hợp: tất cả cơ chế kết hợp
```

### Nền Tảng Web

Trực quan hóa tương tác, sơ đồ từng bước, trình xem mã nguồn và tài liệu.

```sh
cd web && npm install && npm run dev   # http://localhost:3000
```

## Lộ Trình Học

```
Giai đoạn 1: VÒNG LẶP                  Giai đoạn 2: LẬP KẾ HOẠCH & KIẾN THỨC
==================                       ==============================
s01  Vòng Lặp Agent           [1]       s03  TodoWrite               [5]
     while + stop_reason                      TodoManager + nhắc nhở
     |                                        |
     +-> s02  Sử Dụng Công Cụ     [4]   s04  Subagent               [5]
              dispatch map: name->handler     messages[] mới cho mỗi con
                                              |
                                         s05  Kỹ Năng               [5]
                                              SKILL.md qua tool_result
                                              |
                                         s06  Nén Ngữ Cảnh          [5]
                                              nén 3 lớp

Giai đoạn 3: LƯU TRỮ                   Giai đoạn 4: NHÓM
==================                       =====================
s07  Tác Vụ                    [8]       s09  Nhóm Agent             [9]
     CRUD dạng file + đồ thị phụ thuộc        đồng đội + hộp thư JSONL
     |                                        |
s08  Tác Vụ Nền               [6]       s10  Giao Thức Nhóm         [12]
     daemon thread + hàng đợi thông báo       shutdown + FSM duyệt kế hoạch
                                              |
                                         s11  Agent Tự Chủ           [14]
                                              chu kỳ chờ + tự nhận việc
                                         |
                                         s12  Cô Lập Worktree        [16]
                                              phối hợp tác vụ + làn thực thi cô lập tùy chọn

                                         [N] = số lượng công cụ
```

## Kiến Trúc

```
learn-claude-code/
|
|-- agents/                        # Triển khai tham khảo bằng Python (s01-s12 + tổng hợp s_full)
|-- docs/{en,zh,ja}/               # Tài liệu ưu tiên mô hình tư duy (3 ngôn ngữ)
|-- web/                           # Nền tảng học tương tác (Next.js)
|-- skills/                        # File kỹ năng cho s05
+-- .github/workflows/ci.yml      # CI: kiểm tra kiểu + build
```

## Tài Liệu

Ưu tiên mô hình tư duy: vấn đề, giải pháp, sơ đồ ASCII, mã tối giản.
Có sẵn bằng [English](./docs/en/) | [中文](./docs/zh/) | [日本語](./docs/ja/).

| Phiên | Chủ Đề | Phương Châm |
|-------|--------|-------------|
| [s01](./docs/en/s01-the-agent-loop.md) | Vòng Lặp Agent | *Chỉ cần một vòng lặp và Bash* |
| [s02](./docs/en/s02-tool-use.md) | Sử Dụng Công Cụ | *Thêm công cụ nghĩa là thêm một handler* |
| [s03](./docs/en/s03-todo-write.md) | TodoWrite | *Agent không có kế hoạch sẽ trôi dạt* |
| [s04](./docs/en/s04-subagent.md) | Subagent | *Chia nhỏ tác vụ lớn; mỗi tác vụ con có ngữ cảnh riêng* |
| [s05](./docs/en/s05-skill-loading.md) | Kỹ Năng | *Tải kiến thức khi cần, không tải trước* |
| [s06](./docs/en/s06-context-compact.md) | Nén Ngữ Cảnh | *Ngữ cảnh sẽ đầy; bạn cần cách để giải phóng* |
| [s07](./docs/en/s07-task-system.md) | Tác Vụ | *Chia mục tiêu lớn thành tác vụ nhỏ, sắp xếp thứ tự, lưu xuống đĩa* |
| [s08](./docs/en/s08-background-tasks.md) | Tác Vụ Nền | *Chạy thao tác chậm trong nền; agent tiếp tục suy nghĩ* |
| [s09](./docs/en/s09-agent-teams.md) | Nhóm Agent | *Khi tác vụ quá lớn cho một mình, giao cho đồng đội* |
| [s10](./docs/en/s10-team-protocols.md) | Giao Thức Nhóm | *Đồng đội cần quy tắc giao tiếp chung* |
| [s11](./docs/en/s11-autonomous-agents.md) | Agent Tự Chủ | *Đồng đội tự quét bảng và nhận tác vụ* |
| [s12](./docs/en/s12-worktree-task-isolation.md) | Worktree + Cô Lập Tác Vụ | *Mỗi người làm trong thư mục riêng, không ảnh hưởng lẫn nhau* |

## Tiếp Theo -- từ hiểu biết đến triển khai

Sau 12 phiên học, bạn hiểu rõ cách một agent hoạt động từ trong ra ngoài. Hai cách để áp dụng kiến thức đó:

### Kode Agent CLI -- CLI Agent Lập Trình Mã Nguồn Mở

> `npm i -g @shareai-lab/kode`

Hỗ trợ Skill & LSP, sẵn sàng cho Windows, tích hợp được với GLM / MiniMax / DeepSeek và các mô hình mở khác. Cài đặt và dùng ngay.

GitHub: **[shareAI-lab/Kode-cli](https://github.com/shareAI-lab/Kode-cli)**

### Kode Agent SDK -- Nhúng Khả Năng Agent Vào Ứng Dụng Của Bạn

Claude Code Agent SDK chính thức giao tiếp với một tiến trình CLI đầy đủ phía dưới -- mỗi người dùng đồng thời nghĩa là một tiến trình terminal riêng. Kode SDK là một thư viện độc lập không có chi phí tiến trình cho mỗi người dùng, có thể nhúng vào backend, tiện ích mở rộng trình duyệt, thiết bị nhúng, hoặc bất kỳ runtime nào.

GitHub: **[shareAI-lab/Kode-agent-sdk](https://github.com/shareAI-lab/Kode-agent-sdk)**

---

## Repo Liên Quan: từ *phiên theo yêu cầu* đến *trợ lý luôn hoạt động*

Agent mà repo này dạy là kiểu **dùng xong bỏ** -- mở terminal, giao tác vụ, đóng khi xong, phiên tiếp theo bắt đầu từ đầu. Đó là mô hình Claude Code.

[OpenClaw](https://github.com/openclaw/openclaw) đã chứng minh một khả năng khác: trên cùng lõi agent, hai cơ chế biến agent từ "phải chọc mới chạy" thành "tự thức dậy mỗi 30 giây để tìm việc":

- **Heartbeat** -- mỗi 30 giây hệ thống gửi cho agent một tin nhắn để kiểm tra xem có việc gì không. Không có? Quay lại ngủ. Có? Hành động ngay.
- **Cron** -- agent có thể tự lên lịch các tác vụ tương lai, tự động thực thi khi đến giờ.

Thêm định tuyến IM đa kênh (WhatsApp / Telegram / Slack / Discord, hơn 13 nền tảng), bộ nhớ ngữ cảnh bền vững, và hệ thống tính cách Soul, agent chuyển từ công cụ dùng một lần thành trợ lý AI cá nhân luôn hoạt động.

**[claw0](https://github.com/shareAI-lab/claw0)** là repo giảng dạy đồng hành, phân tích các cơ chế này từ đầu:

```
claw agent = lõi agent + heartbeat + cron + IM chat + bộ nhớ + tính cách
```

```
learn-claude-code                   claw0
(lõi runtime agent:                 (trợ lý luôn hoạt động chủ động:
 vòng lặp, công cụ, lập kế hoạch,   heartbeat, cron, kênh IM,
 nhóm, cô lập worktree)             bộ nhớ, tính cách soul)
```

## Giấy Phép

MIT

---

**Mô hình chính là agent. Việc của chúng ta là cung cấp công cụ và không cản đường.**
