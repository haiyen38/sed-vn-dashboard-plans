# Phase 05: Admin Dashboard (`/admin/dashboard`)

**Status:** pending  
**Priority:** high  
**Depends on:** Phase 01

## Context
- Role color: `#2563EB` (xanh dương)
- URL: `/admin/dashboard`
- Current state: H1 "Dashboard với Widgets" + Empty state "Không có widget nào"
- User: admin — vận hành hệ thống, quản lý users, doanh thu
- Existing admin pages: Users, Curriculum, Labels, Prompt Patterns, Payments, Membership Orders, Logs

## Layout Design

### Mobile (single column)
```
1. Header Admin
2. Stats hôm nay: [Đăng ký mới] [Active users] [Giao dịch hôm nay]
3. Doanh thu: hôm nay + tuần + delta
4. Người dùng mới (3 items recent)
5. Cần xử lý: đơn chờ + giao dịch lỗi
6. Quick actions
```

### Desktop (3 columns)
```
Col trái (35%):          Col giữa (35%):        Col phải (30%):
- System stats (3 cards) - New users (5 items)  - Quick actions
- Revenue card           - Pending orders        - Recent logs summary
```

## Related Code Files
- **Tạo mới:**
  - `src/pages/admin/dashboard.tsx`
  - `src/components/admin/admin-system-stats.tsx`
  - `src/components/admin/admin-revenue-card.tsx`
  - `src/components/admin/admin-new-users.tsx`
  - `src/components/admin/admin-pending-tasks.tsx`
  - `src/components/admin/admin-quick-actions.tsx`
  - `src/hooks/admin/use-admin-dashboard-data.ts`
- **Đọc để hiểu:**
  - `src/pages/admin/users.tsx` (user data types)
  - `src/pages/admin/payment/transaction.tsx` (payment data)
  - `src/pages/admin/membership/orders.tsx` (order data)
  - Admin services/hooks

## Data Needed

```tsx
interface AdminDashboardData {
  todayStats: {
    newRegistrations: number;   // đăng ký mới hôm nay
    activeUsers: number;        // users active hôm nay
    todayTransactions: number;  // số giao dịch hôm nay
  };
  revenue: {
    today: number;    // VND
    thisWeek: number; // VND
    weekDelta: number; // % so với tuần trước (+ hoặc -)
  };
  recentUsers: RecentUser[];  // 5 users mới nhất
  pendingTasks: {
    pendingOrders: number;    // đơn hàng chờ xác nhận
    failedTransactions: number; // giao dịch lỗi
  };
}

interface RecentUser {
  name: string;
  email: string;
  role: string;
  createdAt: string; // "vừa xong", "2 phút trước"
}
```

## Implementation Steps

### Step 1: Scout admin pages
Đọc existing admin pages để hiểu:
- Payment/transaction data types và status values
- Membership order status values ("Thành công", "Chờ xử lý", etc.)
- User data shape từ `/admin/users`
- Có sẵn aggregate API endpoint không, hay cần tính client-side

### Step 2: Data Hook
```tsx
// src/hooks/admin/use-admin-dashboard-data.ts
function useAdminDashboardData(): {
  data: AdminDashboardData;
  isLoading: boolean;
}
```

Data sources:
- `GET /admin/users?sort=createdAt&limit=5` → recent users
- `GET /admin/payment/transaction?date=today` → today revenue + transactions
- `GET /admin/membership/orders?status=pending` → pending orders count
- `GET /admin/payment/transaction?status=failed` → failed transactions count

Nếu không có filter API → fetch all và filter client-side (note để optimize sau).

### Step 3: AdminSystemStats
```tsx
// 3 stat cards hàng đầu
<div className="grid grid-cols-3 gap-4">
  <StatCard
    icon={<UserPlusIcon />}
    value={todayStats.newRegistrations}
    label="đăng ký hôm nay"
    accentColor="#2563EB"
  />
  <StatCard
    icon={<ActivityIcon />}
    value={todayStats.activeUsers}
    label="users active"
    accentColor="#2563EB"
  />
  <StatCard
    icon={<CreditCardIcon />}
    value={todayStats.todayTransactions}
    label="giao dịch hôm nay"
    accentColor="#2563EB"
  />
</div>
```

### Step 4: AdminRevenueCard
```tsx
// Card riêng, nổi bật hơn với số tiền lớn
<AdminRevenueCard
  today={revenue.today}
  thisWeek={revenue.thisWeek}
  weekDelta={revenue.weekDelta}
  onViewDetail={() => navigate('/admin/payment/transaction')}
/>
```

Format tiền VND: `toLocaleString('vi-VN')` + `đ`

```
┌───────────────────────────────────────┐
│ 💰 Doanh thu                          │
│                                       │
│ Hôm nay:    1,200,000đ                │
│ Tuần này:   8,500,000đ  ↑ 15%        │
│                                       │
│ [Xem chi tiết thanh toán →]           │
└───────────────────────────────────────┘
```

Delta badge: `↑ 15%` màu `#22C55E` (tăng), `↓ 5%` màu `#EF4444` (giảm)

### Step 5: AdminNewUsers
```tsx
// Danh sách 5 users đăng ký gần nhất
<AdminNewUsers
  users={recentUsers}
  onViewAll={() => navigate('/admin/users')}
/>
```

Mỗi item:
```
[Avatar initials]  lan.nguyen@gmail.com      Nhà giáo dục
                   vừa đăng ký
                                   [Login as]
```

Button "Login as" nhỏ (ghost/outline) — dùng impersonation feature đã có.

### Step 6: AdminPendingTasks
```tsx
// Alert section cho các việc cần xử lý
<AdminPendingTasks
  pendingOrders={pendingTasks.pendingOrders}
  failedTransactions={pendingTasks.failedTransactions}
  onViewOrders={() => navigate('/admin/membership/orders')}
  onViewTransactions={() => navigate('/admin/payment/transaction')}
/>
```

```
⚠️ Cần xử lý
─────────────────────────────────────
• 3 đơn hàng chờ xác nhận    [Xem →]
• 2 giao dịch lỗi             [Xem →]
```

Nếu không có gì cần xử lý: `"✅ Không có việc cần xử lý"` xanh nhạt.

### Step 7: AdminQuickActions
```tsx
// 3 nút hành động nhanh
<AdminQuickActions
  onCreateUser={() => navigate('/admin/users/create')} // kiểm tra route
  onManageCurriculum={() => navigate('/admin/curriculum')}
  onViewLogs={() => navigate('/admin/logs')}
/>
```

```
[+ Tạo user mới]  [📚 Giáo trình]  [📋 Logs]
```

### Step 8: Mobile Bottom Navigation
```tsx
const adminBottomNav = [
  { icon: <HomeIcon />, label: 'Trang chủ', href: '/admin/dashboard' },
  { icon: <UsersIcon />, label: 'Thành viên', href: '/admin/users' },
  { icon: <CreditCardIcon />, label: 'Thanh toán', href: '/admin/payment/transaction' },
  { icon: <SettingsIcon />, label: 'Quản lý', href: '/admin/curriculum' },
]
```

## Todo List
- [ ] Scout admin pages, hiểu payment/order/user data types
- [ ] Xác định có filter API không (date, status)
- [ ] Implement `use-admin-dashboard-data.ts`
- [ ] Implement `admin-system-stats.tsx`
- [ ] Implement `admin-revenue-card.tsx` với VND formatting
- [ ] Implement `admin-new-users.tsx` với impersonation button
- [ ] Implement `admin-pending-tasks.tsx`
- [ ] Implement `admin-quick-actions.tsx`
- [ ] Wire tất cả vào `dashboard.tsx`
- [ ] Mobile layout + bottom nav

## Success Criteria
- [ ] Stats cards hiển thị đúng số liệu hôm nay
- [ ] Revenue format đúng VND, delta badge đúng màu
- [ ] New users list refresh khi có user mới (TanStack Query refetch interval?)
- [ ] Pending tasks count đúng, link đúng trang
- [ ] Impersonation "Login as" hoạt động từ dashboard
- [ ] Responsive: desktop 3-col, mobile single-col

## Unresolved Questions
- Có aggregate API endpoint cho dashboard stats không? Hay phải fetch nhiều endpoint?
- "Active users" định nghĩa thế nào? Login trong ngày? Hay làm bài trong ngày?
- Auto-refresh interval cho dashboard: 1 phút? 5 phút? Hay manual refresh?
- Pending orders: status nào là "chờ"? (`pending`? `waiting`?)
