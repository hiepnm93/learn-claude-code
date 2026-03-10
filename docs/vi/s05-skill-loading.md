# s05: Kỹ năng (Skills)

`s01 > s02 > s03 > s04 > [ s05 ] s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"Tải kiến thức khi cần, không phải ngay từ đầu"* -- tiêm qua tool_result, không phải system prompt.

## Vấn đề

Bạn muốn agent tuân theo các quy trình chuyên biệt: quy ước git, mẫu kiểm thử, danh sách kiểm tra đánh giá mã nguồn. Đặt tất cả vào system prompt sẽ lãng phí token cho các kỹ năng không được sử dụng. 10 kỹ năng với 2000 token mỗi kỹ năng = 20.000 token, phần lớn không liên quan đến bất kỳ tác vụ nào.

## Giải pháp

```
System prompt (Lớp 1 -- luôn hiện diện):
+--------------------------------------+
| You are a coding agent.              |
| Skills available:                    |
|   - git: Git workflow helpers        |  ~100 token/kỹ năng
|   - test: Testing best practices     |
+--------------------------------------+

Khi model gọi load_skill("git"):
+--------------------------------------+
| tool_result (Lớp 2 -- theo yêu cầu):|
| <skill name="git">                   |
|   Full git workflow instructions...  |  ~2000 token
|   Step 1: ...                        |
| </skill>                             |
+--------------------------------------+
```

Lớp 1: *tên* kỹ năng trong system prompt (chi phí thấp). Lớp 2: *nội dung* đầy đủ qua tool_result (theo yêu cầu).

## Cách hoạt động

1. Mỗi kỹ năng là một thư mục chứa file `SKILL.md` với YAML frontmatter.

```
skills/
  pdf/
    SKILL.md       # ---\n name: pdf\n description: Process PDF files\n ---\n ...
  code-review/
    SKILL.md       # ---\n name: code-review\n description: Review code\n ---\n ...
```

2. SkillLoader quét các file `SKILL.md`, sử dụng tên thư mục làm định danh kỹ năng.

```python
class SkillLoader:
    def __init__(self, skills_dir: Path):
        self.skills = {}
        for f in sorted(skills_dir.rglob("SKILL.md")):
            text = f.read_text()
            meta, body = self._parse_frontmatter(text)
            name = meta.get("name", f.parent.name)
            self.skills[name] = {"meta": meta, "body": body}

    def get_descriptions(self) -> str:
        lines = []
        for name, skill in self.skills.items():
            desc = skill["meta"].get("description", "")
            lines.append(f"  - {name}: {desc}")
        return "\n".join(lines)

    def get_content(self, name: str) -> str:
        skill = self.skills.get(name)
        if not skill:
            return f"Error: Unknown skill '{name}'."
        return f"<skill name=\"{name}\">\n{skill['body']}\n</skill>"
```

3. Lớp 1 được đưa vào system prompt. Lớp 2 chỉ là một trình xử lý tool khác.

```python
SYSTEM = f"""You are a coding agent at {WORKDIR}.
Skills available:
{SKILL_LOADER.get_descriptions()}"""

TOOL_HANDLERS = {
    # ...các tool cơ bản...
    "load_skill": lambda **kw: SKILL_LOADER.get_content(kw["name"]),
}
```

Model học được những kỹ năng nào tồn tại (chi phí thấp) và tải chúng khi cần thiết (chi phí cao).

## Thay đổi so với s04

| Thành phần     | Trước (s04)      | Sau (s05)                    |
|----------------|------------------|------------------------------|
| Công cụ        | 5 (cơ bản + task)| 5 (cơ bản + load_skill)      |
| System prompt  | Chuỗi tĩnh      | + mô tả kỹ năng             |
| Kiến thức      | Không có         | skills/\*/SKILL.md files     |
| Tiêm dữ liệu  | Không có         | Hai lớp (system + result)    |

## Thử nghiệm

```sh
cd learn-claude-code
python agents/s05_skill_loading.py
```

1. `What skills are available?`
2. `Load the agent-builder skill and follow its instructions`
3. `I need to do a code review -- load the relevant skill first`
4. `Build an MCP server using the mcp-builder skill`
