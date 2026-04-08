# Phase 04: Teacher Dashboard (`/mentor/dashboard`)

**Status:** pending  
**Priority:** high  
**Depends on:** Phase 01

## Context
- Role color: `#059669` (xanh lá)
- URL: `/mentor/dashboard`
- Current state: H1 "Dashboard với Widgets" + Empty state "Không có widget nào"
- User: giáo viên — quản lý lớp, giao bài, theo dõi HS
- Teacher hiện có: 2 lớp, 47+ HS, 104 đề thi

## Layout Design

### Mobile (single column)
```
1. Header + Greeting "Xin chào Cô/Thầy [tên]! ✨"
2. Stats tổng quan: [Số lớp] [Tổng HS]
3. Bài làm mới cần xem (count + link)
4. 🚨 Học sinh cần chú ý (alert list)
5. Thống kê từng lớp (card per class)
6. Quick actions: [+ Tạo đề thi] [+ Giao bài]
```

### Desktop (2 columns)
```
Col trái (55%):                  Col phải (45%):
- Greeting + Stats               - 🚨 HS cần chú ý
- Class cards (stats per class)  - Bài làm mới
- Quick actions                  - Weak areas per class
```

## Related Code Files
- **Tạo mới:**
  - `src/pages/mentor/dashboard.tsx`
  - `src/components/teacher/teacher-overview-stats.tsx`
  - `src/components/teacher/new-submissions-alert.tsx`
  - `src/components/teacher/students-need-attention.tsx`
  - `src/components/teacher/class-summary-card.tsx`
  - `src/components/teacher/teacher-quick-actions.tsx`
  - `src/hooks/teacher/use-teacher-dashboard-data.ts`
- **Đọc để hiểu:**
  - `src/pages/mentor/class.tsx` (class data)
  - `src/pages/mentor/student-exam.tsx` (submission data)
  - `src/pages/mentor/exam.tsx` (exam creation navigation)
  - Mentor services/hooks

## Data Needed

```tsx
interface TeacherDashboardData {
  classes: ClassSummary[];       // danh sách lớp với stats
  newSubmissions: number;        // số bài nộp mới chưa xem
  recentSubmissions: Submission[]; // 3-5 bài gần nhất
  studentsNeedAttention: StudentAlert[]; // HS cần chú ý
}

interface ClassSummary {
  id: string;
  name: string;
  code: string;
  studentCount: number;
  weeklyExams: number;    // lượt làm trong tuần
  avgScore: number;       // % đúng trung bình
}

interface StudentAlert {
  studentName: string;
  className: string;
  reason: 'low_score' | 'inactive' | 'declining';
  detail: string; // "Phân số 42% đúng" | "3 ngày chưa học" | "Giảm 20%"
}
```

**Alert criteria (CONFIRMED — configurable by Admin + Người bảo trợ):**
- `low_score`: avgScore < X% (default 60%, admin và người bảo trợ config được)
- `inactive`: không có submission trong N ngày (default 3 ngày, configurable)
- `declining`: avgScore tuần này thấp hơn tuần trước > Y% (default 15%, configurable)
- Config lưu ở: settings của admin (system-wide default) + override của từng người bảo trợ cho con mình
- Cần API: `GET /settings/alert-criteria` để lấy threshold

**Lưu ý về Giáo viên:**
- Tài khoản Giáo viên bán riêng (không phải Người bảo trợ)
- Giáo viên tạo lớp độc lập, học sinh join lớp qua class code
- Không có quan hệ parent-child với Người bảo trợ trong gói Giáo viên

## Implementation Steps

### Step 1: Scout mentor pages
Đọc existing mentor pages để hiểu:
- Class data structure, stats API
- Submission data (có "mới" / "đã xem" không?)
- Navigation pattern sang class detail, exam create

### Step 2: Data Hook
```tsx
// src/hooks/teacher/use-teacher-dashboard-data.ts
function useTeacherDashboardData(): {
  data: TeacherDashboardData;
  isLoading: boolean;
}
```

Aggregate data từ nhiều endpoints:
- `GET /mentor/class` → classes list
- `GET /mentor/student-exam?new=true` → new submissions (nếu API hỗ trợ)
- Tính StudentAlert từ class submission data

### Step 3: TeacherOverviewStats
```tsx
// 2 stat cards: Số lớp đang dạy | Tổng học sinh
<div className="grid grid-cols-2 gap-4">
  <StatCard
    icon={<SchoolIcon />}
    value={classes.length}
    label="lớp đang dạy"
    accentColor="#059669"
  />
  <StatCard
    icon={<UsersIcon />}
    value={totalStudents}
    label="học sinh"
    accentColor="#059669"
  />
</div>
```

### Step 4: NewSubmissionsAlert
```tsx
// Banner/card nổi bật nếu có bài làm mới
// newSubmissions > 0: hiển thị count + link
// newSubmissions === 0: ẩn hoặc hiện empty state nhẹ

<NewSubmissionsAlert
  count={newSubmissions}
  recentItems={recentSubmissions.slice(0, 3)}
  onViewAll={() => navigate('/mentor/student-exam')}
/>
```

Style: background `#F0FDF4` (xanh nhạt), border `#059669`, icon `InboxIcon`

### Step 5: StudentsNeedAttention
```tsx
// Alert list — phần quan trọng nhất với giáo viên
// Hiển thị tối đa 5 HS, link "Xem tất cả" sang class detail

<StudentsNeedAttention
  alerts={studentsNeedAttention.slice(0, 5)}
  onViewStudent={(studentId, classId) =>
    navigate(`/mentor/class/${classId}/students`)
  }
/>
```

Mỗi item:
```
🔴 Nguyễn Văn A  [Lớp Trạng Nguyên]
   Phân số: 42% đúng — Cần hỗ trợ
   
🟡 Trần Thị B    [Toán 1 Cô Thảo]
   3 ngày chưa học
   
🔴 Lê Văn C      [Lớp Trạng Nguyên]
   Điểm giảm 20% tuần này
```

Alert icon theo severity: `low_score` + `declining` → 🔴, `inactive` → 🟡

Empty state: `"✅ Tất cả học sinh đang học tốt!"` với icon xanh

### Step 6: ClassSummaryCard
```tsx
// Card per class, hiển thị stats tuần
<ClassSummaryCard
  class={cls}
  onViewClass={() => navigate(`/mentor/class/${cls.id}/stats`)}
  onAssignExam={() => navigate(`/mentor/class/${cls.id}/exam`)}
/>
```

Card content:
```
┌─────────────────────────────────────┐
│ 🏫 Trạng Nguyên Toàn Tài           │
│ Code: KDBBDMD5 • 23 học sinh        │
│ ─────────────────────────────────   │
│ Tuần này: 45 lượt làm • 85% đúng  │
│ ──────────────────────────────────  │
│ [Xem lớp]    [+ Giao bài tập]      │
└─────────────────────────────────────┘
```

### Step 7: TeacherQuickActions
```tsx
// 2 primary action buttons, luôn visible
<TeacherQuickActions
  onCreateExam={() => navigate('/mentor/exam/create')} // kiểm tra route
  onAssignExam={() => navigate('/mentor/class')} // chọn lớp rồi assign
/>
```

```
[+ Tạo đề thi mới]    [📋 Giao bài tập]
```

### Step 8: Mobile Bottom Navigation
```tsx
const teacherBottomNav = [
  { icon: <HomeIcon />, label: 'Trang chủ', href: '/mentor/dashboard' },
  { icon: <InboxIcon />, label: 'Bài làm', href: '/mentor/student-exam' },
  { icon: <UsersIcon />, label: 'Lớp học', href: '/mentor/class' },
  { icon: <FileTextIcon />, label: 'Đề thi', href: '/mentor/exam' },
]
```

## Todo List
- [ ] Scout mentor pages, hiểu class/submission data types
- [ ] Xác định API endpoint cho new submissions, class stats
- [ ] Implement `use-teacher-dashboard-data.ts` với alert logic
- [ ] Implement `teacher-overview-stats.tsx`
- [ ] Implement `new-submissions-alert.tsx`
- [ ] Implement `students-need-attention.tsx`
- [ ] Implement `class-summary-card.tsx`
- [ ] Implement `teacher-quick-actions.tsx`
- [ ] Wire tất cả vào `dashboard.tsx`
- [ ] Mobile layout + bottom nav

## Success Criteria
- [ ] Stats hiển thị đúng số lớp và tổng HS
- [ ] Alert HS cần chú ý tính đúng theo 3 criteria
- [ ] Empty state alerts hiện "✅ Tất cả học sinh đang học tốt!"
- [ ] Class summary cards link đúng đến class detail
- [ ] Quick actions navigate đúng
- [ ] Responsive: desktop 2-col, mobile single-col

## Unresolved Questions
- API `/mentor/student-exam` có filter `?new=true` hay phải client-side filter?
- "Bài làm mới" = chưa xem hay chưa chấm? (Submission status: `Đã nộp` vs `Đã chấm điểm`)
- API nào để GV lấy được alert criteria threshold đã được admin/PH config?
- Người bảo trợ config alert cho con — teacher có xem được config đó không?
