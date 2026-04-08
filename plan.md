---
title: Dashboard 4 Loại Tài Khoản SED.VN
status: pending
priority: high
created: 2026-04-07
blockedBy: []
blocks: []
---

# Dashboard 4 Loại Tài Khoản SED.VN

## Mục tiêu
Implement trang chủ (dashboard) cho 4 role: Guardian, Student, Teacher, Admin — là trang đầu tiên người dùng thấy sau khi đăng nhập.

## Tech Stack (confirmed)
- **React + Vite** (frontend framework)
- **Ant Design** (UI components — Card, Statistic, Table, Tabs, Badge, Progress, etc.)
- **TanStack Router** (routing)
- **TanStack Query** (data fetching — useQuery)
- **Zustand** (state — useAuthStore, useNavStore, useAppStore)
- **Lucide React** (icons)
- **Tailwind CSS** (utility styling)
- **Service-based** pattern (exam.services, auth.services, etc.)

## Context
- Brainstorm: `plans/reports/brainstorm-260407-1741-dashboard-design-4-accounts.md`
- Research: `plans/reports/researcher-260407-1719-sed-vn-account-features-research.md`
- URLs: `/guardian/dashboard`, `/study/dashboard`, `/mentor/dashboard`, `/admin/dashboard`

## Brand Identity
**iSED** — Viện Khoa Học và Phát Triển Giáo Dục  
Logo: `https://cdn0145.cdn4s.com/media/logo/logo-ised_1.png`  
Ý nghĩa SED: *Shaping Every Dream* / *Solutions for Every Difference* — đa dạng, cá nhân hoá

Logo màu sắc đa dạng (navy, vàng, cam, xanh dương, teal) → thể hiện "diversity in education"

## Account Type Clarification (QUAN TRỌNG)
| Loại TK | Ai dùng | Cách mua |
|---------|---------|----------|
| **Người bảo trợ (Nhà giáo dục)** | Cha mẹ | Đăng ký miễn phí, nâng cấp theo gói |
| **Học sinh** | Con em | Tạo bởi Người bảo trợ |
| **Giáo viên** | Giáo viên chuyên nghiệp | Bán riêng (gói riêng), tạo lớp độc lập |
| **Admin** | Vận hành hệ thống | Nội bộ |

**Cha mẹ mua = 1 TK Người bảo trợ + N TK học sinh** (N phụ thuộc gói: free=1, gia đình=4)  
**Giáo viên mua riêng** → gói Giáo viên = tối đa 50 học sinh

## Color Palette (thiết kế đã thống nhất)
```
--sed-navy:   #1B2F6E  (primary brand — từ logo)
--sed-gold:   #C9A227  (accent — từ logo)
--ised-blue:  #3B82F6  (iSED text blue)
--ised-yellow:#F5C518  (i dot yellow)

Role colors (từ logo color wheel):
Guardian:     #7C3AED  (tím — wisdom)
Student:      #F97316  (cam — energy)
Teacher:      #059669  (xanh lá — growth)
Admin:        #2563EB  (xanh dương — trust)
```

## Phases

| # | Phase | Status | Priority |
|---|-------|--------|----------|
| 01 | Shared Components & Design System | pending | high |
| 02 | Guardian Dashboard | pending | high |
| 03 | Student Dashboard | pending | high |
| 04 | Teacher Dashboard | pending | high |
| 05 | Admin Dashboard | pending | high |

## Dependencies
- Phase 01 phải hoàn thành trước các phase 02-05
- Phase 02-05 có thể implement song song sau Phase 01

## Success Criteria
- [ ] 4 dashboard đều responsive (mobile + desktop)
- [ ] State-aware: Guardian dashboard thay đổi theo free/paid/new user
- [ ] Student streak counter + confetti animation hoạt động
- [ ] Teacher alert HS cần chú ý hiển thị đúng
- [ ] Admin dashboard hiển thị đúng stats hệ thống
- [ ] Mobile bottom navigation đúng cho từng role
