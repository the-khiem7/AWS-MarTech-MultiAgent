---
title: "Mẫu Tài Liệu Hugo"
date: 2026-04-29
weight: 1
chapter: false
---

# Mẫu Tài Liệu Hugo

Sử dụng template này để khởi tạo một website tài liệu Hugo song ngữ, có triển khai GitHub Pages, quy ước tài nguyên theo page bundle, và devcontainer sẵn sàng cho quy trình biên soạn.

## Template Này Cung Cấp

- Cấu trúc nội dung tiếng Anh và tiếng Việt
- Triển khai bằng GitHub Pages Actions
- Quy ước tài nguyên theo page bundle
- Lớp tương thích cho nội dung cũ dùng `/images/...`
- Devcontainer cho Hugo, sơ đồ, ảnh chụp màn hình, OCR, và tối ưu ảnh

## Các Mục Ví Dụ

{{% children description="true" /%}}

## Định Hướng Biên Soạn

Trang mới nên dùng page bundle và đường dẫn Markdown tương đối cho tài nguyên riêng của trang. Tài nguyên static dùng chung vẫn có thể được dùng khi phù hợp, nhưng nội dung mới nên tránh đường dẫn root-relative dạng `/images/...`.
