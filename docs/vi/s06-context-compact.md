# s06: Nén ngữ cảnh (Context Compact)

`s01 > s02 > s03 > s04 > s05 > [ s06 ] | s07 > s08 > s09 > s10 > s11 > s12`

> *"Ngữ cảnh sẽ đầy; bạn cần cách để giải phóng không gian"* -- chiến lược nén ba lớp cho các phiên làm việc không giới hạn.

## Vấn đề

Cửa sổ ngữ cảnh là hữu hạn. Một lần gọi `read_file` trên file 1000 dòng tốn khoảng 4000 token. Sau khi đọc 30 file và chạy 20 lệnh bash, bạn đạt hơn 100.000 token. Agent không thể làm việc trên các codebase lớn nếu không có nén.

## Giải pháp

Ba lớp, tăng dần mức độ nén:

```
Mỗi lượt:
+------------------+
| Tool call result |
+------------------+
        |
        v
[Lớp 1: micro_compact]        (ngầm, mỗi lượt)
  Thay thế tool_result cũ hơn 3 lượt
  bằng "[Previous: used {tool_name}]"
        |
        v
[Kiểm tra: tokens > 50000?]
   |               |
   không           có
   |               |
   v               v
tiếp tục    [Lớp 2: auto_compact]
              Lưu bản ghi vào .transcripts/
              LLM tóm tắt cuộc hội thoại.
              Thay thế tất cả tin nhắn bằng [tóm tắt].
                    |
                    v
            [Lớp 3: compact tool]
              Model gọi compact một cách tường minh.
              Cùng cơ chế tóm tắt như auto_compact.
```

## Cách hoạt động

1. **Lớp 1 -- micro_compact**: Trước mỗi lần gọi LLM, thay thế các kết quả tool cũ bằng placeholder.

```python
def micro_compact(messages: list) -> list:
    tool_results = []
    for i, msg in enumerate(messages):
        if msg["role"] == "user" and isinstance(msg.get("content"), list):
            for j, part in enumerate(msg["content"]):
                if isinstance(part, dict) and part.get("type") == "tool_result":
                    tool_results.append((i, j, part))
    if len(tool_results) <= KEEP_RECENT:
        return messages
    for _, _, part in tool_results[:-KEEP_RECENT]:
        if len(part.get("content", "")) > 100:
            part["content"] = f"[Previous: used {tool_name}]"
    return messages
```

2. **Lớp 2 -- auto_compact**: Khi số token vượt ngưỡng, lưu toàn bộ bản ghi ra đĩa, sau đó yêu cầu LLM tóm tắt.

```python
def auto_compact(messages: list) -> list:
    # Lưu bản ghi để phục hồi
    transcript_path = TRANSCRIPT_DIR / f"transcript_{int(time.time())}.jsonl"
    with open(transcript_path, "w") as f:
        for msg in messages:
            f.write(json.dumps(msg, default=str) + "\n")
    # LLM tóm tắt
    response = client.messages.create(
        model=MODEL,
        messages=[{"role": "user", "content":
            "Summarize this conversation for continuity..."
            + json.dumps(messages, default=str)[:80000]}],
        max_tokens=2000,
    )
    return [
        {"role": "user", "content": f"[Compressed]\n\n{response.content[0].text}"},
        {"role": "assistant", "content": "Understood. Continuing."},
    ]
```

3. **Lớp 3 -- nén thủ công**: Công cụ `compact` kích hoạt cùng cơ chế tóm tắt theo yêu cầu.

4. Vòng lặp tích hợp cả ba lớp:

```python
def agent_loop(messages: list):
    while True:
        micro_compact(messages)                        # Lớp 1
        if estimate_tokens(messages) > THRESHOLD:
            messages[:] = auto_compact(messages)       # Lớp 2
        response = client.messages.create(...)
        # ... thực thi tool ...
        if manual_compact:
            messages[:] = auto_compact(messages)       # Lớp 3
```

Bản ghi lưu giữ toàn bộ lịch sử trên đĩa. Không có gì thực sự bị mất -- chỉ được chuyển ra khỏi ngữ cảnh đang hoạt động.

## Thay đổi so với s05

| Thành phần       | Trước (s05)      | Sau (s06)                    |
|------------------|------------------|------------------------------|
| Công cụ          | 5                | 5 (cơ bản + compact)         |
| Quản lý ngữ cảnh | Không có         | Nén ba lớp                   |
| Micro-compact    | Không có         | Kết quả cũ -> placeholder    |
| Auto-compact     | Không có         | Kích hoạt khi vượt ngưỡng token|
| Bản ghi          | Không có         | Lưu vào .transcripts/        |

## Thử nghiệm

```sh
cd learn-claude-code
python agents/s06_context_compact.py
```

1. `Read every Python file in the agents/ directory one by one` (theo dõi micro-compact thay thế các kết quả cũ)
2. `Keep reading files until compression triggers automatically`
3. `Use the compact tool to manually compress the conversation`
