# s02: Sử dụng Công cụ

`s01 > [ s02 ] s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"Thêm một công cụ nghĩa là thêm một hàm xử lý"* -- vòng lặp giữ nguyên; các công cụ mới được đăng ký vào bảng điều phối.

## Vấn đề

Chỉ với `bash`, agent phải chạy lệnh shell cho mọi thứ. `cat` cắt ngắn không lường trước, `sed` lỗi với các ký tự đặc biệt, và mỗi lần gọi bash là một bề mặt bảo mật không bị ràng buộc. Các công cụ chuyên biệt như `read_file` và `write_file` cho phép bạn áp dụng giới hạn đường dẫn ở cấp độ công cụ.

Điểm mấu chốt: thêm công cụ không cần thay đổi vòng lặp.

## Giải pháp

```
+--------+      +-------+      +------------------+
|  User  | ---> |  LLM  | ---> | Tool Dispatch    |
| prompt |      |       |      | {                |
+--------+      +---+---+      |   bash: run_bash |
                    ^           |   read: run_read |
                    |           |   write: run_wr  |
                    +-----------+   edit: run_edit |
                    tool_result | }                |
                                +------------------+

Bảng điều phối là một dict: {tool_name: handler_function}.
Một phép tra cứu thay thế mọi chuỗi if/elif.
```

## Cách hoạt động

1. Mỗi công cụ có một hàm xử lý. Cơ chế giới hạn đường dẫn ngăn chặn việc thoát khỏi thư mục làm việc.

```python
def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path

def run_read(path: str, limit: int = None) -> str:
    text = safe_path(path).read_text()
    lines = text.splitlines()
    if limit and limit < len(lines):
        lines = lines[:limit]
    return "\n".join(lines)[:50000]
```

2. Bảng điều phối liên kết tên công cụ với hàm xử lý.

```python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"],
                                        kw["new_text"]),
}
```

3. Trong vòng lặp, tra cứu hàm xử lý theo tên. Phần thân vòng lặp không thay đổi so với s01.

```python
for block in response.content:
    if block.type == "tool_use":
        handler = TOOL_HANDLERS.get(block.name)
        output = handler(**block.input) if handler \
            else f"Unknown tool: {block.name}"
        results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": output,
        })
```

Thêm một công cụ = thêm một hàm xử lý + thêm một mục định nghĩa schema. Vòng lặp không bao giờ thay đổi.

## Những gì đã thay đổi so với s01

| Thành phần        | Trước (s01)            | Sau (s02)                          |
|--------------------|------------------------|------------------------------------|
| Công cụ           | 1 (chỉ bash)           | 4 (bash, read, write, edit)        |
| Điều phối         | Gọi bash cố định       | Dict `TOOL_HANDLERS`               |
| Bảo mật đường dẫn | Không có               | Sandbox `safe_path()`              |
| Vòng lặp agent    | Không thay đổi         | Không thay đổi                     |

## Thử nghiệm

```sh
cd learn-claude-code
python agents/s02_tool_use.py
```

1. `Read the file requirements.txt`
2. `Create a file called greet.py with a greet(name) function`
3. `Edit greet.py to add a docstring to the function`
4. `Read greet.py to verify the edit worked`
