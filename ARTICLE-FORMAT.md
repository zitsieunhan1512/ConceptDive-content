# Định dạng bài viết ConceptDive / ConceptDive article format

Hướng dẫn cho người viết bài và người làm thư viện GitHub. App đọc mọi file `.md`, nhưng bài theo đúng khuôn dưới đây mới có đủ tóm tắt 30 giây, ý cốt lõi, Tự kiểm tra và sơ đồ. App không dịch bài: viết bằng ngôn ngữ nào thì đọc bằng giọng của ngôn ngữ đó.

A guide for article authors and GitHub library maintainers. The app reads any `.md` file, but only articles in the format below get the 30-second summary, key insight, self-check and diagrams. The app never translates articles: each article is read aloud in its own language.

---

## Tiếng Việt

### Đầu bài (frontmatter, tuỳ chọn)

```markdown
---
id: kotlin-coroutines
title: Bản chất Coroutines trong Kotlin
lang: vi
domain: kotlin
tags: [kotlin, concurrency]
related: [kotlin-flow]
diagrams: [Cơ chế|coroutine-state-machine.png|Hàm suspend được biên dịch thành máy trạng thái]
---
```

- `lang`: `vi` hoặc `en`. Bỏ trống thì app tự nhận diện theo tỷ lệ chữ có dấu tiếng Việt.
- `diagrams`: mỗi mục dạng `Tên mục|tên-file.png|chú thích`, ảnh PNG đặt trong thư mục `diagrams/` cạnh bài. App tải ảnh cùng bài khi đồng bộ.
- `related`: `id` của bài liên quan trong cùng repo. Trong thân bài có thể liên kết `[nhãn](concept:id)`.

### Các mục

- `## Bản 30 giây` — luôn là mục đầu: 2–3 câu nói được bản chất.
- Phần giữa tuỳ chủ đề, gợi ý: `## Vấn đề buộc nó ra đời`, `## Nó thực sự là cái gì`, `## Cơ chế`, `## Đánh đổi`, `## Ví von`.
- `## Bẫy & ranh giới` — tốn tài nguyên gì, khi nào không nên dùng, dấu hiệu dùng sai.
- `## Nên học tiếp` (tuỳ chọn).
- `## Key insight` — đúng một câu.
- `## Câu hỏi thực chiến` — mỗi câu hỏi một dòng `**Hỏi:** …`, câu trả lời nằm ngay dòng dưới (không có dòng trống).

---

## English

### Frontmatter (optional)

```markdown
---
id: kotlin-coroutines
title: Kotlin Coroutines Deep Dive
lang: en
domain: kotlin
tags: [kotlin, concurrency]
related: [kotlin-flow]
diagrams: [Mechanism|coroutine-state-machine.png|Suspend functions compile into a state machine]
---
```

- `lang`: `en` or `vi`. If omitted, the app detects the language from the share of Vietnamese diacritics.
- `diagrams`: each entry is `Section|file-name.png|caption`; put the PNG in a `diagrams/` folder next to the article. The app downloads images with the article.
- `related`: `id`s of related articles in the same repo. Inside the body you can link `[label](concept:id)`.

### Sections

- `## 30-Second Summary` — always first: 2–3 sentences that capture the essence.
- Middle sections fit the topic, for example: `## Why It Had to Exist`, `## What It Really Is`, `## Mechanism`, `## Trade-offs`, `## Analogy`.
- `## Pitfalls & Boundaries` — what it costs, when not to use it, symptoms of misuse.
- `## What to Learn Next` (optional).
- `## Key insight` — exactly one sentence.
- `## Interview Questions` — each question on a `**Question:** …` line, with the answer on the very next line (no blank line).

---

## Sơ đồ trong thân bài / Inline diagrams

Ngoài ảnh PNG ở frontmatter, bài có thể nhúng sơ đồ SVG ngay trong mục bằng khối mã `sodo` (app hiển thị thành hình):

Besides PNG images in the frontmatter, an article can embed an SVG diagram inside a section with a `sodo` code block (the app renders it as an image):

````markdown
```sodo
<svg viewBox="0 0 400 200" xmlns="http://www.w3.org/2000/svg">…</svg>
```
````

Lệnh mẫu "Chia sẻ lệnh" trong app đã yêu cầu ChatGPT viết đúng khuôn này cho cả hai ngôn ngữ.

The app's "Share prompt" templates already ask ChatGPT to follow this format in either language.
