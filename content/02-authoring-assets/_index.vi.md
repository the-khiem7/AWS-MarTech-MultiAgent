---
title: "Biên Soạn Tài Nguyên"
date: 2026-04-29
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

## Mục Đích

Trang này minh họa quy ước tài nguyên mặc định cho nội dung mới: đặt media trong page bundle và dùng đường dẫn Markdown tương đối.

## Tài Nguyên Static Trong Page Bundle

Hình bên dưới nằm cạnh trang này trong thư mục `_static/`.

![Ví dụ tài nguyên page bundle](_static/page-bundle-example.svg)

## Mẫu Được Khuyến Nghị

Dùng cấu trúc này cho tài nguyên riêng của một trang:

```text
content/02-authoring-assets/
  _index.md
  _index.vi.md
  _static/
    page-bundle-example.svg
```

Sau đó tham chiếu tài nguyên bằng đường dẫn tương đối:

```markdown
![Ví dụ tài nguyên page bundle](_static/page-bundle-example.svg)
```

Tránh dùng đường dẫn root-relative dạng `/images/...` trong nội dung mới. Dạng này chỉ được hỗ trợ để tương thích với nội dung cũ.
