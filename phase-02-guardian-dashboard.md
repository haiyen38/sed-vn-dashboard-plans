# Phase 02: Guardian Dashboard (`/guardian/dashboard`)

**Status:** pending  
**Priority:** high  
**Depends on:** Phase 01 (shared components)

## Context
- Role color: `#7C3AED` (tím)
- URL: `/guardian/dashboard`
- Current state: redirect thẳng sang `/guardian/students`, không có dashboard
- User là phụ huynh/nhà giáo dục — người MUA sản phẩm
- Brainstorm: `plans/reports/brainstorm-260407-1741-dashboard-design-4-accounts.md`
- Research: `plans/reports/researcher-260407-1719-sed-vn-account-features-research.md`

## Account Type Context (CONFIRMED)
- Cha mẹ mua = **1 TK Người bảo trợ** (nhà giáo dục) + **N TK học sinh** (N theo gói)
- Free: tối đa 1 học sinh
- Gói Gia đình: tối đa 4 học sinh
- **Giáo viên** là role riêng biệt, bán riêng — KHÔNG phải từ TK Người bảo trợ
- Người bảo trợ chỉ mua gói "Gia đình" hoặc ở free

## Business Logic

### 3 trạng thái dashboard
| State | Điều kiện | Hiển thị |
|-------|-----------|----------|
| `new_user` | Chưa có học sinh | Onboarding flow |
| `free` | Có HS, chưa trả phí | Stats + Upsell banner cố định |
| `paid` | Đã có gói Gia đình | Full dashboard, không upsell |

### Data cần fetch
```
GET /guardian/students           → danh sách học sinh (để tab switcher)
GET /guardian/submission         → bài làm gần nhất per student
GET /guardian/weakness           → điểm yếu per student
```
Dữ liệu học tập theo ngày/tuần cần thêm API endpoint (ghi chú cho backend).

## Layout Design

### Mobile (single column)
```
1. Header: Logo + tên user + 🔔
2. Student Tab Switcher (horizontal scroll, nếu >1 HS)
3. Stat Cards: [Số đề hôm nay] [% câu đúng] [Thời gian học]
4. [FREE ONLY] Upsell banner compact
5. Biểu đồ 7 ngày (bar chart - Ant Design Column chart)
6. Điểm yếu nổi bật (top 2 items)
7. Bài làm gần nhất (3 items)
8. [NEW USER / < 7 ngày] Support widget (dismissible)
```

### Desktop (2 columns)
```
Col trái (60%):          Col phải (40%):
- Student Tab Switcher   - Điểm yếu (top 3)
- Stat Cards (3 col)     - Bài làm gần nhất (5 items)
- Biểu đồ 7 ngày        - [FREE] Upsell card
                         - Support widget
```

## Related Code Files
- **Tạo mới:**
  - `src/pages/guardian/dashboard.tsx` (hoặc đường dẫn tương tự)
  - `src/components/guardian/student-tab-switcher.tsx`
  - `src/components/guardian/guardian-stat-cards.tsx`
  - `src/components/guardian/weekly-activity-chart.tsx`
  - `src/components/guardian/weakness-summary.tsx`
  - `src/components/guardian/recent-submissions.tsx`
  - `src/components/guardian/upsell-banner.tsx`
  - `src/components/guardian/support-widget.tsx`
  - `src/components/guardian/onboarding-welcome.tsx`
- **Đọc để hiểu:**
  - `src/pages/guardian/students.tsx` (existing, để hiểu data shape)
  - `src/pages/guardian/submission.tsx` (hiểu submission data)
  - `src/pages/guardian/weakness.tsx` (hiểu weakness data)
  - Guardian services/hooks

## Implementation Steps

### Step 1: Scout guardian pages
Đọc existing guardian pages để hiểu:
- Data types (Student, Submission, Weakness interfaces)
- Existing hooks/services đang dùng
- Route config cho `/guardian/dashboard`

### Step 2: Routing
Kiểm tra TanStack Router config, đảm bảo `/guardian/dashboard` route tồn tại và là trang chủ mặc định khi vào `/guardian/`.

### Step 3: State Detection Hook
```tsx
// src/hooks/guardian/use-guardian-dashboard-state.ts
type GuardianState = 'new_user' | 'free' | 'paid';

function useGuardianDashboardState(): {
  state: GuardianState;
  students: Student[];
  isLoading: boolean;
}
```
Logic:
- `students.length === 0` → `new_user`
- `students.length > 0 && !hasActiveMembership` → `free`
- `hasActiveMembership` → `paid`

### Step 4: Onboarding State (new_user)
```tsx
// Hiển thị khi chưa có học sinh
<OnboardingWelcome>
  <h2>Chào mừng đến với SED! 👋</h2>
  <p>Bắt đầu bằng cách tạo tài khoản học sinh cho con</p>
  <Steps items={[
    { title: 'Tạo tài khoản học sinh' },
    { title: 'Chọn giáo trình' },
    { title: 'Con bắt đầu học' },
  ]} />
  <Button type="primary" size="large" icon={<PlusIcon />}>
    + Tạo tài khoản học sinh
  </Button>
  <SupportWidget />
</OnboardingWelcome>
```

### Step 5: StudentTabSwitcher
```tsx
// Horizontal scrollable tabs, mỗi tab là 1 học sinh
// Nếu free & đã đủ giới hạn (1 HS): nút "+" click → toast/modal thông báo + link nâng cấp
// Nếu paid Gia đình: nút "+" click → modal tạo HS mới (tối đa 4)
// KHÔNG có gói Giáo viên cho Người bảo trợ — chỉ free hoặc Gia đình
<StudentTabSwitcher
  students={students}
  selectedId={selectedStudentId}
  onSelect={setSelectedStudentId}
  canAddMore={isPaid && students.length < 4}
  maxStudents={isPaid ? 4 : 1}
  onAddMoreLocked={() => {
    // Hiện modal thông báo rõ ràng:
    // "Gói miễn phí chỉ tạo được 1 học sinh.
    //  Nâng cấp Gói Gia đình để tạo đến 4 học sinh."
    // + nút [Nâng cấp ngay] → /settings/membership
    showUpgradeModal();
  }}
/>
```

### Step 6: Stat Cards (3 cards)
Dữ liệu hôm nay cho student đang được chọn:
- **Số đề hôm nay**: count submissions với date = today
- **% câu đúng**: avg score hôm nay
- **Thời gian học**: tổng thời gian (nếu API có, nếu không hiện `--`)

```tsx
<div className="grid grid-cols-3 gap-3">
  <StatCard icon={<FileTextIcon />} value={todayExams} label="đề hôm nay" accentColor="#7C3AED" />
  <StatCard icon={<TrendingUpIcon />} value={`${avgScore}%`} label="câu đúng" accentColor="#7C3AED" />
  <StatCard icon={<ClockIcon />} value={studyTime} label="học hôm nay" accentColor="#7C3AED" />
</div>
```

### Step 7: Weekly Activity Chart
Ant Design `Column` chart (nếu có @ant-design/charts) hoặc dùng Ant Design `Progress` bars + CSS nếu không có chart lib.

Fallback đơn giản (không cần chart lib):
```tsx
// 7 mini progress bars theo ngày, full width
<WeeklyActivityChart data={weeklyData} accentColor="#7C3AED" />
```

### Step 8: Weakness Summary (top 2)
```tsx
<WeaknessSummary
  items={weaknesses.slice(0, 2)}
  onViewAll={() => navigate('/guardian/weakness')}
/>
// Item: topic name + accuracy % + progress bar
```

### Step 9: Recent Submissions (3 items)
```tsx
<RecentSubmissions
  items={submissions.slice(0, 3)}
  onViewAll={() => navigate('/guardian/submission')}
/>
// Item: exam title + score + date + badge Đã nộp/Đã chấm
```

### Step 10: Upsell Banner (FREE only)
```tsx
// Compact, ở dưới stat cards (mobile) hoặc col phải (desktop)
<UpsellBanner
  visible={isFree}
  message="Gói miễn phí: 1/1 đề hôm nay"
  ctaText="🚀 Nâng cấp — Không giới hạn"
  onUpgrade={() => navigate('/settings/membership')}
/>
```

### Step 11: Support Widget (new users)
```tsx
// Chỉ hiện nếu accountAge < 7 ngày hoặc không có submissions
// localStorage key: 'sed_support_widget_dismissed'
<SupportWidget
  visible={showSupport && !isDismissed}
  onDismiss={() => { localStorage.setItem('sed_support_widget_dismissed', '1'); }}
  links={{
    zalo: 'https://zalo.me/...',
    tvgd: 'https://...',
    youtube: 'https://youtube.com/...',
  }}
/>
```

### Step 12: Mobile Bottom Navigation
```tsx
const guardianBottomNav = [
  { icon: <HomeIcon />, label: 'Trang chủ', href: '/guardian/dashboard' },
  { icon: <UsersIcon />, label: 'Học sinh', href: '/guardian/students' },
  { icon: <FileTextIcon />, label: 'Bài làm', href: '/guardian/submission' },
  { icon: <SettingsIcon />, label: 'Cài đặt', href: '/settings/profile' },
]
```

## Todo List
- [ ] Scout existing guardian files, hiểu data types
- [ ] Kiểm tra/tạo route `/guardian/dashboard`
- [ ] Implement `use-guardian-dashboard-state.ts` hook
- [ ] Implement `onboarding-welcome.tsx` (State: new_user)
- [ ] Implement `student-tab-switcher.tsx`
- [ ] Implement `guardian-stat-cards.tsx`
- [ ] Implement `weekly-activity-chart.tsx`
- [ ] Implement `weakness-summary.tsx`
- [ ] Implement `recent-submissions.tsx`
- [ ] Implement `upsell-banner.tsx`
- [ ] Implement `support-widget.tsx`
- [ ] Wiring tất cả vào `dashboard.tsx`
- [ ] Mobile layout + bottom nav
- [ ] Test cả 3 states (new_user / free / paid)

## Success Criteria
- [ ] new_user state: hiển thị onboarding + support widget
- [ ] free state: stats + upsell banner + support widget (nếu mới)
- [ ] paid state: full dashboard, không có upsell banner
- [ ] Tab switcher chuyển đổi data đúng theo từng HS
- [ ] Responsive: mobile stack + desktop 2-col
- [ ] Upsell banner dẫn đến `/settings/membership`
- [ ] Support widget dismissible, nhớ state qua localStorage

## Unresolved Questions
- API endpoint nào trả về thời gian học (study time) per student per day?
- Membership status lấy từ đâu? `useAuthStore`? hay riêng API?
- Người bảo trợ có thể config "mục tiêu học ngày" cho học sinh không? (số đề/ngày)
