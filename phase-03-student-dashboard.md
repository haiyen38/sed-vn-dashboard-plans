# Phase 03: Student Dashboard (`/study/dashboard`)

**Status:** pending  
**Priority:** high  
**Depends on:** Phase 01

## Context
- Role color: `#F97316` (cam)
- URL: `/study/dashboard`
- Current state: H1 "Dashboard với Widgets" + Empty state "Không có widget nào"
- User: học sinh — người trực tiếp làm bài
- Style: Gamification nhẹ (streak, progress, confetti)

## Layout Design

### Mobile (single column)
```
1. Header: Logo + tên HS + 🔔
2. Greeting + ngày hôm nay
3. 🔥 Streak counter (N ngày liên tiếp)
4. Mục tiêu hôm nay: progress bar [2/3 đề] + CTA "▶ Làm bài tiếp theo"
5. Bài tập từ lớp (assigned, có deadline)
6. Thống kê tuần: [Số đề] [% đúng] [Xếp hạng lớp]
7. Điểm yếu → [Luyện ngay]
```

### Desktop (3 columns)
```
Col trái (30%):          Col giữa (40%):       Col phải (30%):
- Streak counter          - Bài tập từ lớp      - Stats tuần
- Mục tiêu hôm nay       - Đề gợi ý tiếp theo  - Điểm yếu
- CTA chính                                     - Xếp hạng lớp
```

## Related Code Files
- **Tạo mới:**
  - `src/pages/study/dashboard.tsx`
  - `src/components/student/streak-counter.tsx`
  - `src/components/student/daily-goal-progress.tsx`
  - `src/components/student/class-assignments.tsx`
  - `src/components/student/student-weekly-stats.tsx`
  - `src/components/student/student-weakness-quick.tsx`
  - `src/hooks/student/use-student-dashboard-data.ts`
- **Đọc để hiểu:**
  - `src/pages/study/exam/` (hiểu exam flow và navigate)
  - `src/pages/study/workspace/my-class.tsx`
  - `src/pages/study/weakness.tsx`
  - Student services/hooks hiện có

## Key UX Rules
- **CTA "Làm bài tiếp theo" phải là element nổi bật nhất trang** — size lớn, màu `#F97316`
- Streak counter dùng animation: đếm từ 0 → actual (800ms, ease-out)
- Progress bar color-coded: cam → vàng → xanh theo %
- Confetti khi hoàn thành 100% mục tiêu ngày

## Implementation Steps

### Step 1: Scout student pages
Đọc existing pages để hiểu:
- Exam navigation pattern (làm sao navigate sang `/study/exam/{id}`)
- Submission data types
- Class/assignment data shape

### Step 2: Data Hook
```tsx
// src/hooks/student/use-student-dashboard-data.ts
interface StudentDashboardData {
  streak: number;           // ngày học liên tiếp
  streakBroken: boolean;    // hôm qua không học
  todayGoal: { done: number; total: number }; // 0/3, 2/3, 3/3
  nextExamId?: string;      // đề tiếp theo để làm
  classAssignments: Assignment[];
  weeklyStats: { exams: number; avgScore: number; classRank?: number };
  weaknesses: WeaknessTopic[];
}

function useStudentDashboardData(): {
  data: StudentDashboardData;
  isLoading: boolean;
}
```

**Streak logic (CONFIRMED):**
- Lấy lịch sử bài làm, group by date
- Đếm ngày liên tiếp tính từ hôm nay ngược lại
- **Grace period 1 ngày**: nếu hôm qua không học → `streakBroken = true` nhưng streak chưa reset
- Streak chỉ reset về 0 nếu **2 ngày liên tiếp** không có submission
- `streakBroken = true` → hiện banner nhẹ, nhắc nhở không phạt ngay

**Mục tiêu hôm nay (CONFIRMED):**
- Số đề/ngày do **Người bảo trợ hoặc Giáo viên config** (không hardcode)
- Nếu chưa config: default = 3 đề/ngày
- Cần API/setting: `GET /student/goal` hoặc lấy từ profile settings

### Step 3: StreakCounter Component
```tsx
// src/components/student/streak-counter.tsx
interface StreakCounterProps {
  streak: number;
  broken: boolean;
}

// Khi broken=true: hiện banner nhẹ "😢 Bắt đầu lại hôm nay!"
// Khi streak >= 7: glow effect vàng quanh badge
// Animation: useEffect với interval đếm 0 → streak (800ms total)
```

```tsx
// Streak animation pattern
const [displayed, setDisplayed] = useState(0);
useEffect(() => {
  if (streak === 0) return;
  const step = Math.ceil(streak / 20);
  const timer = setInterval(() => {
    setDisplayed(prev => {
      if (prev + step >= streak) { clearInterval(timer); return streak; }
      return prev + step;
    });
  }, 40); // 40ms * 20 steps = 800ms
  return () => clearInterval(timer);
}, [streak]);
```

### Step 4: DailyGoalProgress Component
```tsx
// src/components/student/daily-goal-progress.tsx
interface DailyGoalProgressProps {
  done: number;      // 0, 1, 2, 3
  total: number;     // 3
  nextExamId?: string;
  onStartExam: (id?: string) => void;
  onGoalComplete: () => void; // trigger confetti
}
```

Progress bar color:
```tsx
const getBarColor = (pct: number) => {
  if (pct >= 1) return '#7C3AED'; // completed — purple shimmer
  if (pct >= 0.67) return '#22C55E'; // xanh
  if (pct >= 0.34) return '#F59E0B'; // vàng
  return '#F97316'; // cam
};
```

CTA Button states:
- `done < total`: "▶ Làm bài tiếp theo" (primary, large)
- `done >= total`: "🎉 Hoàn thành! Làm thêm" (secondary)

### Step 5: Confetti on Goal Complete
```tsx
// Dùng canvas-confetti (3KB gzip)
// Cài: npm install canvas-confetti @types/canvas-confetti

import confetti from 'canvas-confetti';

const triggerConfetti = () => {
  confetti({
    particleCount: 30,
    spread: 60,
    colors: ['#F97316', '#C9A227', '#FFF'],
    origin: { y: 0.6 },
  });
};

// Gọi khi done === total (useEffect, chỉ trigger 1 lần)
useEffect(() => {
  if (done >= total && total > 0) {
    triggerConfetti();
    onGoalComplete?.();
  }
}, [done, total]);
```

### Step 6: ClassAssignments Component
```tsx
// src/components/student/class-assignments.tsx
// Hiển thị bài tập được giáo viên giao, có nút "Làm bài"
// Lấy từ lớp học đang tham gia
// Empty state: "Chưa có bài tập mới"

interface Assignment {
  id: string;
  examTitle: string;
  className: string;
  assignedBy: string; // tên giáo viên
  dueDate?: string;
}
```

### Step 7: StudentWeeklyStats
```tsx
// 3 stat cards: Số đề tuần | % đúng trung bình | Xếp hạng lớp
<div className="grid grid-cols-3 gap-2">
  <StatCard value={weeklyStats.exams} label="đề tuần này" accentColor="#F97316" />
  <StatCard value={`${weeklyStats.avgScore}%`} label="câu đúng" accentColor="#F97316" />
  <StatCard value={`#${weeklyStats.classRank}`} label="xếp hạng lớp" accentColor="#F97316" />
</div>
```

### Step 8: StudentWeaknessQuick
```tsx
// Top 2 điểm yếu với nút [Luyện ngay] → navigate to /study/weakness
<StudentWeaknessQuick
  items={weaknesses.slice(0, 2)}
  onPractice={(topic) => navigate('/study/weakness', { search: { topic } })}
/>
```

### Step 9: Empty States
- **Ngày đầu (streak=0, todayGoal.done=0):**
  ```tsx
  <EmptyDashboardState
    illustration="🚀"
    title="Bắt đầu hành trình học tập!"
    description="Làm bài đầu tiên để bắt đầu streak của bạn"
    action={{ label: "▶ Làm bài đầu tiên", onClick: () => navigate('/study/exam/list') }}
  />
  ```
- **Streak broken:**
  ```
  Banner nhẹ (không màu đỏ):
  "😢 Streak bị gián đoạn. Hôm nay bắt đầu lại nhé!"
  ```

### Step 10: Mobile Bottom Navigation
```tsx
const studentBottomNav = [
  { icon: <HomeIcon />, label: 'Trang chủ', href: '/study/dashboard' },
  { icon: <PenLineIcon />, label: 'Làm bài', href: '/study/exam/list' },
  { icon: <BarChart2Icon />, label: 'Điểm yếu', href: '/study/weakness' },
  { icon: <SchoolIcon />, label: 'Lớp học', href: '/study/workspace/my-class' },
]
```

## Todo List
- [ ] Scout student pages, hiểu exam navigation và data types
- [ ] Implement `use-student-dashboard-data.ts` với streak logic
- [ ] Implement `streak-counter.tsx` với count-up animation
- [ ] Implement `daily-goal-progress.tsx` với color-coded progress bar
- [ ] Cài canvas-confetti, implement confetti trigger
- [ ] Implement `class-assignments.tsx`
- [ ] Implement `student-weekly-stats.tsx`
- [ ] Implement `student-weakness-quick.tsx`
- [ ] Wire tất cả vào `dashboard.tsx`
- [ ] Handle empty states (new user, streak broken)
- [ ] Mobile layout + bottom nav

## Success Criteria
- [ ] Streak counter đếm animation hoạt động
- [ ] CTA "Làm bài tiếp theo" navigate đúng sang exam
- [ ] Progress bar đổi màu theo tiến độ
- [ ] Confetti trigger đúng 1 lần khi hoàn thành 3/3
- [ ] Streak broken state hiển thị banner nhẹ nhàng
- [ ] Empty state (ngày đầu) hiển thị đúng
- [ ] Desktop 3-col layout, mobile single-col

## Unresolved Questions
- API endpoint nào để lấy mục tiêu ngày đã được PH/GV set cho HS?
- Xếp hạng lớp: API nào? Lớp nào nếu HS tham gia nhiều lớp?
- `nextExamId` (đề tiếp theo gợi ý): logic chọn đề theo curriculum đã chọn?
- Grace period streak: có lưu vào DB không hay chỉ tính client-side?
