# Phase 01: Shared Components & Design System

**Status:** pending  
**Priority:** high  
**Phải hoàn thành trước:** Phase 02, 03, 04, 05

## Context
- Tech stack: React + Vite + Ant Design + TanStack Router + TanStack Query + Zustand + Lucide React + Tailwind CSS
- Layout hiện tại: `.app-container.flex` → `.left-container` (sidebar 250px) + `.main-container`
- Dashboard hiện tại: H1 "Dashboard với Widgets" + Ant Design Empty state "Không có widget nào"
- Brainstorm: `plans/reports/brainstorm-260407-1741-dashboard-design-4-accounts.md`

## Mục tiêu
Tạo shared design tokens, base components dùng chung cho tất cả 4 dashboard.

## Related Code Files
- **Tạo mới:**
  - `src/components/dashboard/stat-card.tsx`
  - `src/components/dashboard/section-header.tsx`
  - `src/components/dashboard/empty-dashboard-state.tsx`
  - `src/styles/dashboard-tokens.css` (hoặc Tailwind config extension)
- **Đọc để hiểu:**
  - `src/components/` (existing components)
  - `src/styles/` hoặc global CSS files
  - Existing page files (mentor/study/admin/guardian)

## Implementation Steps

### 1. Scout codebase structure
```bash
# Tìm cấu trúc thư mục src
find src -type f -name "*.tsx" | head -50
# Tìm existing dashboard files
find src -path "*/dashboard*" -type f
# Tìm existing shared components
ls src/components/
```

### 2. Design Tokens (CSS Variables)
**Brand identity:** iSED logo sử dụng màu đa dạng (color wheel) thể hiện "diversity in education"  
Logo: `https://cdn0145.cdn4s.com/media/logo/logo-ised_1.png`

Thêm vào global CSS hoặc Tailwind config:
```css
:root {
  /* Brand — từ iSED logo */
  --sed-navy: #1B2F6E;    /* Navy từ logo SED shield */
  --sed-gold: #C9A227;    /* Gold từ logo laurel */
  --ised-blue: #3B82F6;   /* Blue từ text "iSED" */
  --ised-yellow: #F5C518; /* Yellow dot trên chữ "i" */
  
  /* Role colors — lấy từ color wheel của logo */
  --color-guardian: #7C3AED;  /* Tím — wisdom, trust */
  --color-student:  #F97316;  /* Cam — energy, youth */
  --color-teacher:  #059669;  /* Xanh lá — growth */
  --color-admin:    #2563EB;  /* Xanh dương — professional */
  
  /* Dashboard cards */
  --card-radius: 12px;
  --card-shadow: 0 1px 3px rgba(0,0,0,0.08);
  --card-padding: 16px;
}
```

Hoặc extend `tailwind.config.js`:
```js
theme: {
  extend: {
    colors: {
      'sed-navy':   '#1B2F6E',
      'sed-gold':   '#C9A227',
      'ised-blue':  '#3B82F6',
      guardian:     '#7C3AED',
      student:      '#F97316',
      teacher:      '#059669',
      'sed-admin':  '#2563EB',
    }
  }
}
```

### 3. StatCard Component
File: `src/components/dashboard/stat-card.tsx`

```tsx
interface StatCardProps {
  icon: React.ReactNode;
  value: string | number;
  label: string;
  delta?: { value: string; positive: boolean };
  accentColor?: string; // role color
  loading?: boolean;
}
```

Design spec:
- `border-radius: 12px`
- `box-shadow: 0 1px 3px rgba(0,0,0,0.08)`
- Left border accent: `4px solid accentColor`
- Value: `text-3xl font-bold`
- Label: `text-xs text-gray-500`
- Delta badge: `text-xs` với màu xanh/đỏ
- Hover (desktop): `scale-[1.02] transition-transform duration-150`
- Sử dụng Ant Design `Skeleton` khi `loading={true}`

### 4. SectionHeader Component
File: `src/components/dashboard/section-header.tsx`

```tsx
interface SectionHeaderProps {
  title: string;
  icon?: React.ReactNode; // Lucide icon
  action?: { label: string; onClick: () => void };
}
```

Simple header với icon + title + optional action link.

### 5. EmptyDashboardState Component
File: `src/components/dashboard/empty-dashboard-state.tsx`

```tsx
interface EmptyDashboardStateProps {
  title: string;
  description: string;
  action?: { label: string; onClick: () => void; icon?: React.ReactNode };
  illustration?: React.ReactNode; // emoji or SVG
}
```

Dùng cho trường hợp chưa có data (HS chưa làm bài, GV chưa có lớp...).

### 6. Mobile Bottom Navigation wrapper
Tạo logic hiển thị bottom nav chỉ trên mobile (`md:hidden`).
Mỗi role sẽ định nghĩa `bottomNavItems` trong config riêng.

```tsx
// src/components/dashboard/mobile-bottom-nav.tsx
interface BottomNavItem {
  icon: React.ReactNode;
  label: string;
  href: string;
}
interface MobileBottomNavProps {
  items: BottomNavItem[];
  accentColor: string;
}
```

## Todo List
- [ ] Scout codebase để hiểu file structure thực tế
- [ ] Tìm global CSS/Tailwind config file
- [ ] Tạo design tokens
- [ ] Implement `stat-card.tsx`
- [ ] Implement `section-header.tsx`
- [ ] Implement `empty-dashboard-state.tsx`
- [ ] Implement `mobile-bottom-nav.tsx`
- [ ] Test render các components

## Success Criteria
- [ ] StatCard render đúng với loading skeleton
- [ ] Left border accent đúng màu role
- [ ] Hover animation hoạt động trên desktop
- [ ] MobileBottomNav ẩn trên desktop (`md:hidden`)
- [ ] Design tokens apply đúng across components

## Risk Assessment
- **Không biết cấu trúc src**: Scout trước khi tạo file, đặt đúng vị trí
- **Tailwind config có thể không extend được**: Fallback dùng CSS custom properties
- **Ant Design version**: Kiểm tra version để dùng đúng API (v4 vs v5)
