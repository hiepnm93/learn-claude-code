# s03: TodoWrite

`s01 > s02 > [ s03 ] s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"Agent không có kế hoạch sẽ lạc hướng"* -- liệt kê các bước trước, rồi mới thực thi.

## Vấn đề

Với các tác vụ nhiều bước, mô hình mất dấu tiến trình. Nó lặp lại công việc, bỏ qua bước, hoặc đi lạc hướng. Cuộc hội thoại dài làm tình trạng này tệ hơn -- system prompt bị lu mờ khi kết quả từ các công cụ lấp đầy ngữ cảnh. Một lần tái cấu trúc 10 bước có thể hoàn thành bước 1-3, rồi mô hình bắt đầu ứng biến vì nó quên mất bước 4-10.

## Giải pháp

```
+--------+      +-------+      +---------+
|  User  | ---> |  LLM  | ---> | Tools   |
| prompt |      |       |      | + todo  |
+--------+      +---+---+      +----+----+
                    ^                |
                    |   tool_result  |
                    +----------------+
                          |
              +-----------+-----------+
              | TodoManager state     |
              | [ ] task A            |
              | [>] task B  <- doing  |
              | [x] task C            |
              +-----------------------+
                          |
              if rounds_since_todo >= 3:
                inject <reminder> into tool_result
```

## Cách hoạt động

1. TodoManager lưu trữ các mục với trạng thái. Chỉ một mục được phép ở trạng thái `in_progress` tại một thời điểm.

```python
class TodoManager:
    def update(self, items: list) -> str:
        validated, in_progress_count = [], 0
        for item in items:
            status = item.get("status", "pending")
            if status == "in_progress":
                in_progress_count += 1
            validated.append({"id": item["id"], "text": item["text"],
                              "status": status})
        if in_progress_count > 1:
            raise ValueError("Only one task can be in_progress")
        self.items = validated
        return self.render()
```

2. Công cụ `todo` được thêm vào bảng điều phối giống như bất kỳ công cụ nào khác.

```python
TOOL_HANDLERS = {
    # ...các công cụ cơ bản...
    "todo": lambda **kw: TODO.update(kw["items"]),
}
```

3. Cơ chế nhắc nhở sẽ chèn một lời thúc giục nếu mô hình trải qua 3+ lượt mà không gọi `todo`.

```python
if rounds_since_todo >= 3 and messages:
    last = messages[-1]
    if last["role"] == "user" and isinstance(last.get("content"), list):
        last["content"].insert(0, {
            "type": "text",
            "text": "<reminder>Update your todos.</reminder>",
        })
```

Ràng buộc "chỉ một mục in_progress tại một thời điểm" buộc mô hình tập trung tuần tự. Cơ chế nhắc nhở tạo ra tính chịu trách nhiệm.

## Những gì đã thay đổi so với s02

| Thành phần        | Trước (s02)      | Sau (s03)                          |
|--------------------|------------------|------------------------------------|
| Công cụ           | 4                | 5 (+todo)                          |
| Lập kế hoạch      | Không có         | TodoManager với các trạng thái     |
| Chèn nhắc nhở     | Không có         | `<reminder>` sau 3 lượt            |
| Vòng lặp agent    | Điều phối đơn giản | + bộ đếm rounds_since_todo       |

## Thử nghiệm

```sh
cd learn-claude-code
python agents/s03_todo_write.py
```

1. `Refactor the file hello.py: add type hints, docstrings, and a main guard`
2. `Create a Python package with __init__.py, utils.py, and tests/test_utils.py`
3. `Review all Python files and fix any style issues`
