# s04: Agent con

`s01 > s02 > s03 > [ s04 ] s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"Chia nhỏ tác vụ lớn; mỗi tác vụ con nhận một ngữ cảnh sạch"* -- agent con sử dụng messages[] độc lập, giữ cho cuộc hội thoại chính gọn gàng.

## Vấn đề

Khi agent làm việc, mảng messages của nó ngày càng phình to. Mỗi lần đọc tệp, mỗi kết quả bash đều nằm lại trong ngữ cảnh vĩnh viễn. Câu hỏi "Dự án này dùng framework kiểm thử nào?" có thể cần đọc 5 tệp, nhưng agent cha chỉ cần câu trả lời: "pytest."

## Giải pháp

```
Parent agent                     Subagent
+------------------+             +------------------+
| messages=[...]   |             | messages=[]      | <-- mới
|                  |  dispatch   |                  |
| tool: task       | ----------> | while tool_use:  |
|   prompt="..."   |             |   call tools     |
|                  |  summary    |   append results |
|   result = "..." | <---------- | return last text |
+------------------+             +------------------+

Ngữ cảnh của agent cha giữ nguyên sạch sẽ. Ngữ cảnh của agent con bị loại bỏ.
```

## Cách hoạt động

1. Agent cha có công cụ `task`. Agent con nhận tất cả công cụ cơ bản ngoại trừ `task` (không cho phép sinh đệ quy).

```python
PARENT_TOOLS = CHILD_TOOLS + [
    {"name": "task",
     "description": "Spawn a subagent with fresh context.",
     "input_schema": {
         "type": "object",
         "properties": {"prompt": {"type": "string"}},
         "required": ["prompt"],
     }},
]
```

2. Agent con bắt đầu với `messages=[]` và chạy vòng lặp riêng của nó. Chỉ có văn bản cuối cùng được trả về cho agent cha.

```python
def run_subagent(prompt: str) -> str:
    sub_messages = [{"role": "user", "content": prompt}]
    for _ in range(30):  # giới hạn an toàn
        response = client.messages.create(
            model=MODEL, system=SUBAGENT_SYSTEM,
            messages=sub_messages,
            tools=CHILD_TOOLS, max_tokens=8000,
        )
        sub_messages.append({"role": "assistant",
                             "content": response.content})
        if response.stop_reason != "tool_use":
            break
        results = []
        for block in response.content:
            if block.type == "tool_use":
                handler = TOOL_HANDLERS.get(block.name)
                output = handler(**block.input)
                results.append({"type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(output)[:50000]})
        sub_messages.append({"role": "user", "content": results})
    return "".join(
        b.text for b in response.content if hasattr(b, "text")
    ) or "(no summary)"
```

Toàn bộ lịch sử tin nhắn của agent con (có thể lên đến 30+ lần gọi công cụ) bị loại bỏ. Agent cha nhận được một đoạn tóm tắt ngắn dưới dạng `tool_result` thông thường.

## Những gì đã thay đổi so với s03

| Thành phần        | Trước (s03)      | Sau (s04)                         |
|--------------------|------------------|-----------------------------------|
| Công cụ           | 5                | 5 (cơ bản) + task (agent cha)     |
| Ngữ cảnh          | Một bộ dùng chung | Agent cha + agent con cách ly    |
| Agent con          | Không có         | Hàm `run_subagent()`             |
| Giá trị trả về    | Không áp dụng    | Chỉ văn bản tóm tắt              |

## Thử nghiệm

```sh
cd learn-claude-code
python agents/s04_subagent.py
```

1. `Use a subtask to find what testing framework this project uses`
2. `Delegate: read all .py files and summarize what each one does`
3. `Use a task to create a new module, then verify it from here`
