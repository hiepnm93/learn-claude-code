# s01: Vòng Lặp Agent

`[ s01 ] s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"Chỉ cần một vòng lặp và Bash là đủ"* -- một công cụ + một vòng lặp = một agent.

## Vấn đề

Mô hình ngôn ngữ có thể suy luận về mã nguồn, nhưng nó không thể *tương tác* với thế giới thực -- không thể đọc tệp, chạy kiểm thử, hay kiểm tra lỗi. Nếu không có vòng lặp, mỗi lần gọi công cụ bạn phải tự sao chép-dán kết quả trả về. Bạn trở thành vòng lặp.

## Giải pháp

```
+--------+      +-------+      +---------+
|  User  | ---> |  LLM  | ---> |  Tool   |
| prompt |      |       |      | execute |
+--------+      +---+---+      +----+----+
                    ^                |
                    |   tool_result  |
                    +----------------+
                    (loop until stop_reason != "tool_use")
```

Một điều kiện thoát duy nhất kiểm soát toàn bộ luồng xử lý. Vòng lặp chạy cho đến khi mô hình ngừng gọi công cụ.

## Cách hoạt động

1. Prompt của người dùng trở thành tin nhắn đầu tiên.

```python
messages.append({"role": "user", "content": query})
```

2. Gửi danh sách tin nhắn + định nghĩa công cụ đến LLM.

```python
response = client.messages.create(
    model=MODEL, system=SYSTEM, messages=messages,
    tools=TOOLS, max_tokens=8000,
)
```

3. Thêm phản hồi của trợ lý vào danh sách. Kiểm tra `stop_reason` -- nếu mô hình không gọi công cụ nào, ta đã hoàn tất.

```python
messages.append({"role": "assistant", "content": response.content})
if response.stop_reason != "tool_use":
    return
```

4. Thực thi từng lệnh gọi công cụ, thu thập kết quả, thêm vào như một tin nhắn người dùng. Quay lại bước 2.

```python
results = []
for block in response.content:
    if block.type == "tool_use":
        output = run_bash(block.input["command"])
        results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": output,
        })
messages.append({"role": "user", "content": results})
```

Ghép lại thành một hàm duy nhất:

```python
def agent_loop(query):
    messages = [{"role": "user", "content": query}]
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = run_bash(block.input["command"])
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        messages.append({"role": "user", "content": results})
```

Đó là toàn bộ agent chỉ trong chưa đầy 30 dòng. Mọi thứ khác trong khóa học này được xây dựng thêm bên trên -- mà không thay đổi vòng lặp.

## Những gì đã thay đổi

| Thành phần       | Trước      | Sau                            |
|-------------------|------------|--------------------------------|
| Vòng lặp agent   | (không có) | `while True` + stop_reason     |
| Công cụ          | (không có) | `bash` (một công cụ)           |
| Tin nhắn         | (không có) | Danh sách tích lũy             |
| Luồng điều khiển | (không có) | `stop_reason != "tool_use"`    |

## Thử nghiệm

```sh
cd learn-claude-code
python agents/s01_agent_loop.py
```

1. `Create a file called hello.py that prints "Hello, World!"`
2. `List all Python files in this directory`
3. `What is the current git branch?`
4. `Create a directory called test_output and write 3 files in it`
