# s12: Worktree + Cách ly Nhiệm vụ

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > [ s12 ]`

> *"Mỗi người làm việc trong thư mục riêng, không can thiệp lẫn nhau"* -- nhiệm vụ quản lý mục tiêu, worktree quản lý thư mục, liên kết bằng ID.

## Vấn đề

Đến s11, các agent có thể nhận và hoàn thành nhiệm vụ một cách tự chủ. Nhưng mọi nhiệm vụ đều chạy trong cùng một thư mục chung. Hai agent tái cấu trúc các module khác nhau cùng lúc sẽ xung đột: agent A sửa `config.py`, agent B sửa `config.py`, các thay đổi chưa stage bị trộn lẫn, và không ai có thể rollback sạch sẽ.

Bảng nhiệm vụ theo dõi *cần làm gì* nhưng không quan tâm *làm ở đâu*. Cách khắc phục: cấp cho mỗi nhiệm vụ một thư mục git worktree riêng. Nhiệm vụ quản lý mục tiêu, worktree quản lý ngữ cảnh thực thi. Liên kết chúng bằng task ID.

## Giải pháp

```
Mặt phẳng điều khiển (.tasks/)       Mặt phẳng thực thi (.worktrees/)
+------------------+                +------------------------+
| task_1.json      |                | auth-refactor/         |
|   status: in_progress  <------>   branch: wt/auth-refactor
|   worktree: "auth-refactor"   |   task_id: 1             |
+------------------+                +------------------------+
| task_2.json      |                | ui-login/              |
|   status: pending    <------>     branch: wt/ui-login
|   worktree: "ui-login"       |   task_id: 2             |
+------------------+                +------------------------+
                                    |
                          index.json (sổ đăng ký worktree)
                          events.jsonl (nhật ký vòng đời)

Máy trạng thái:
  Nhiệm vụ:  pending -> in_progress -> completed
  Worktree:   absent  -> active      -> removed | kept
```

## Cách hoạt động

1. **Tạo nhiệm vụ.** Lưu mục tiêu trước.

```python
TASKS.create("Implement auth refactor")
# -> .tasks/task_1.json  status=pending  worktree=""
```

2. **Tạo worktree và liên kết với nhiệm vụ.** Truyền `task_id` sẽ tự động chuyển nhiệm vụ sang `in_progress`.

```python
WORKTREES.create("auth-refactor", task_id=1)
# -> git worktree add -b wt/auth-refactor .worktrees/auth-refactor HEAD
# -> index.json có mục mới, task_1.json có worktree="auth-refactor"
```

Việc liên kết ghi trạng thái vào cả hai phía:

```python
def bind_worktree(self, task_id, worktree):
    task = self._load(task_id)
    task["worktree"] = worktree
    if task["status"] == "pending":
        task["status"] = "in_progress"
    self._save(task)
```

3. **Chạy lệnh trong worktree.** `cwd` trỏ đến thư mục cách ly.

```python
subprocess.run(command, shell=True, cwd=worktree_path,
               capture_output=True, text=True, timeout=300)
```

4. **Kết thúc.** Hai lựa chọn:
   - `worktree_keep(name)` -- giữ lại thư mục để dùng sau.
   - `worktree_remove(name, complete_task=True)` -- xóa thư mục, hoàn thành nhiệm vụ liên kết, phát sự kiện. Một lệnh xử lý cả dọn dẹp + hoàn thành.

```python
def remove(self, name, force=False, complete_task=False):
    self._run_git(["worktree", "remove", wt["path"]])
    if complete_task and wt.get("task_id") is not None:
        self.tasks.update(wt["task_id"], status="completed")
        self.tasks.unbind_worktree(wt["task_id"])
        self.events.emit("task.completed", ...)
```

5. **Luồng sự kiện.** Mỗi bước vòng đời đều phát ra `.worktrees/events.jsonl`:

```json
{
  "event": "worktree.remove.after",
  "task": {"id": 1, "status": "completed"},
  "worktree": {"name": "auth-refactor", "status": "removed"},
  "ts": 1730000000
}
```

Các sự kiện được phát: `worktree.create.before/after/failed`, `worktree.remove.before/after/failed`, `worktree.keep`, `task.completed`.

Sau khi gặp sự cố, trạng thái được tái tạo từ `.tasks/` + `.worktrees/index.json` trên ổ đĩa. Bộ nhớ hội thoại là tạm thời; trạng thái tệp là bền vững.

## Thay đổi so với s11

| Thành phần           | Trước (s11)                  | Sau (s12)                                      |
|----------------------|------------------------------|-------------------------------------------------|
| Phối hợp             | Bảng nhiệm vụ (chủ/trạng thái) | Bảng nhiệm vụ + liên kết worktree rõ ràng   |
| Phạm vi thực thi     | Thư mục chung                | Thư mục cách ly theo nhiệm vụ                  |
| Khả năng phục hồi    | Chỉ trạng thái nhiệm vụ     | Trạng thái nhiệm vụ + chỉ mục worktree         |
| Dọn dẹp              | Hoàn thành nhiệm vụ         | Hoàn thành nhiệm vụ + giữ/xóa rõ ràng         |
| Khả năng quan sát vòng đời | Ẩn trong log           | Sự kiện rõ ràng trong `.worktrees/events.jsonl` |

## Thử nghiệm

```sh
cd learn-claude-code
python agents/s12_worktree_task_isolation.py
```

1. `Create tasks for backend auth and frontend login page, then list tasks.`
2. `Create worktree "auth-refactor" for task 1, then bind task 2 to a new worktree "ui-login".`
3. `Run "git status --short" in worktree "auth-refactor".`
4. `Keep worktree "ui-login", then list worktrees and inspect events.`
5. `Remove worktree "auth-refactor" with complete_task=true, then list tasks/worktrees/events.`
