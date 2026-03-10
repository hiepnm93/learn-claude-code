# s07: Hệ thống tác vụ (Task System)

`s01 > s02 > s03 > s04 > s05 > s06 | [ s07 ] s08 > s09 > s10 > s11 > s12`

> *"Chia mục tiêu lớn thành các tác vụ nhỏ, sắp xếp thứ tự, lưu trữ ra đĩa"* -- đồ thị tác vụ dựa trên file với các phụ thuộc, đặt nền tảng cho cộng tác đa agent.

## Vấn đề

TodoManager của s03 là một danh sách phẳng trong bộ nhớ: không có thứ tự, không có phụ thuộc, không có trạng thái nào ngoài hoàn thành hay chưa. Các mục tiêu thực tế có cấu trúc -- tác vụ B phụ thuộc vào tác vụ A, tác vụ C và D có thể chạy song song, tác vụ E chờ cả C và D.

Nếu không có các mối quan hệ tường minh, agent không thể biết cái nào đã sẵn sàng, cái nào đang bị chặn, hay cái nào có thể chạy đồng thời. Và vì danh sách chỉ tồn tại trong bộ nhớ, nén ngữ cảnh (s06) sẽ xóa sạch nó.

## Giải pháp

Nâng cấp danh sách thành một **đồ thị tác vụ** được lưu trữ ra đĩa. Mỗi tác vụ là một file JSON với trạng thái, phụ thuộc (`blockedBy`), và các tác vụ phụ thuộc ngược (`blocks`). Đồ thị trả lời ba câu hỏi tại bất kỳ thời điểm nào:

- **Cái nào đã sẵn sàng?** -- tác vụ có trạng thái `pending` và `blockedBy` rỗng.
- **Cái nào đang bị chặn?** -- tác vụ đang chờ các phụ thuộc chưa hoàn thành.
- **Cái nào đã xong?** -- tác vụ `completed`, khi hoàn thành sẽ tự động mở khóa các tác vụ phụ thuộc.

```
.tasks/
  task_1.json  {"id":1, "status":"completed"}
  task_2.json  {"id":2, "blockedBy":[1], "status":"pending"}
  task_3.json  {"id":3, "blockedBy":[1], "status":"pending"}
  task_4.json  {"id":4, "blockedBy":[2,3], "status":"pending"}

Đồ thị tác vụ (DAG):
                 +----------+
            +--> | task 2   | --+
            |    | pending  |   |
+----------+     +----------+    +--> +----------+
| task 1   |                          | task 4   |
| completed| --> +----------+    +--> | blocked  |
+----------+     | task 3   | --+     +----------+
                 | pending  |
                 +----------+

Thứ tự:        task 1 phải hoàn thành trước 2 và 3
Song song:     task 2 và 3 có thể chạy cùng lúc
Phụ thuộc:     task 4 chờ cả 2 và 3
Trạng thái:    pending -> in_progress -> completed
```

Đồ thị tác vụ này trở thành xương sống điều phối cho mọi thứ sau s07: thực thi nền (s08), đội ngũ đa agent (s09+), và cách ly worktree (s12) đều đọc và ghi vào cùng cấu trúc này.

## Cách hoạt động

1. **TaskManager**: mỗi tác vụ một file JSON, CRUD với đồ thị phụ thuộc.

```python
class TaskManager:
    def __init__(self, tasks_dir: Path):
        self.dir = tasks_dir
        self.dir.mkdir(exist_ok=True)
        self._next_id = self._max_id() + 1

    def create(self, subject, description=""):
        task = {"id": self._next_id, "subject": subject,
                "status": "pending", "blockedBy": [],
                "blocks": [], "owner": ""}
        self._save(task)
        self._next_id += 1
        return json.dumps(task, indent=2)
```

2. **Giải quyết phụ thuộc**: hoàn thành một tác vụ sẽ xóa ID của nó khỏi danh sách `blockedBy` của mọi tác vụ khác, tự động mở khóa các tác vụ phụ thuộc.

```python
def _clear_dependency(self, completed_id):
    for f in self.dir.glob("task_*.json"):
        task = json.loads(f.read_text())
        if completed_id in task.get("blockedBy", []):
            task["blockedBy"].remove(completed_id)
            self._save(task)
```

3. **Trạng thái + liên kết phụ thuộc**: `update` xử lý các chuyển đổi trạng thái và các cạnh phụ thuộc.

```python
def update(self, task_id, status=None,
           add_blocked_by=None, add_blocks=None):
    task = self._load(task_id)
    if status:
        task["status"] = status
        if status == "completed":
            self._clear_dependency(task_id)
    self._save(task)
```

4. Bốn công cụ tác vụ được đưa vào bản đồ điều phối.

```python
TOOL_HANDLERS = {
    # ...các tool cơ bản...
    "task_create": lambda **kw: TASKS.create(kw["subject"]),
    "task_update": lambda **kw: TASKS.update(kw["task_id"], kw.get("status")),
    "task_list":   lambda **kw: TASKS.list_all(),
    "task_get":    lambda **kw: TASKS.get(kw["task_id"]),
}
```

Từ s07 trở đi, đồ thị tác vụ là mặc định cho công việc đa bước. Todo của s03 vẫn dùng cho các danh sách kiểm tra nhanh trong một phiên.

## Thay đổi so với s06

| Thành phần | Trước (s06) | Sau (s07) |
|---|---|---|
| Công cụ | 5 | 8 (`task_create/update/list/get`) |
| Mô hình lập kế hoạch | Danh sách phẳng (trong bộ nhớ) | Đồ thị tác vụ với phụ thuộc (trên đĩa) |
| Quan hệ | Không có | Các cạnh `blockedBy` + `blocks` |
| Theo dõi trạng thái | Xong hoặc chưa | `pending` -> `in_progress` -> `completed` |
| Lưu trữ | Mất khi nén | Tồn tại qua nén và khởi động lại |

## Thử nghiệm

```sh
cd learn-claude-code
python agents/s07_task_system.py
```

1. `Create 3 tasks: "Setup project", "Write code", "Write tests". Make them depend on each other in order.`
2. `List all tasks and show the dependency graph`
3. `Complete task 1 and then list tasks to see task 2 unblocked`
4. `Create a task board for refactoring: parse -> transform -> emit -> test, where transform and emit can run in parallel after parse`
