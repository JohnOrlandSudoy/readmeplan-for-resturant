# Waste, Spillage & Spoilage Reporting Feature Plan
**Kitchen Dashboard Enhancement**

---

## 📋 Executive Summary

Add comprehensive waste tracking capabilities to the Kitchen Dashboard enabling kitchen staff to:
- **Report ingredient spoilage** (expired, discolored, odor issues)
- **Report spillage/accidents** (dropped items, broken containers)
- **Log waste amounts** with reason and responsible staff member
- **Trigger low-stock alerts** when waste causes inventory drops
- **Export waste reports** for cost analysis and waste reduction planning

---

## 🎯 Feature Overview

### Core Capabilities

| Capability | Description | Users | Frequency |
|------------|-------------|-------|-----------|
| **Report Spoilage** | Log expired/unusable ingredients | Kitchen staff | Ad-hoc, real-time |
| **Report Spillage** | Document accidental waste | Kitchen staff | Ad-hoc, real-time |
| **Waste Tracking** | Maintain waste inventory ledger | Kitchen staff | Continuous |
| **Auto-Reorder** | Trigger purchase alerts when waste depletes stock | Admin/Manager | Auto-triggered |
| **Analytics** | View waste trends, costs, hotspots | Admin/Manager | Daily/Weekly/Monthly |
| **Compliance** | Document waste for food safety audit | Admin/Manager | On-demand |

---

## 📐 Architecture & Data Model

### 1. Database Schema

#### **New Table: `waste_logs`**
```sql
CREATE TABLE waste_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  restaurant_id UUID NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
  
  -- Waste Details
  ingredient_id UUID NOT NULL REFERENCES ingredients(id),
  waste_type ENUM ('spoilage', 'spillage', 'other') NOT NULL,
  quantity_wasted DECIMAL(10, 3) NOT NULL,
  unit VARCHAR(50) NOT NULL,  -- kg, L, pieces, etc.
  
  -- Reason & Description
  reason VARCHAR(255) NOT NULL,  -- 'expired', 'damaged_container', 'dropped', 'discolored', 'odor', 'other'
  description TEXT,
  notes TEXT,
  
  -- Tracking
  reported_by_user_id UUID NOT NULL REFERENCES auth.users(id),
  reported_by_name VARCHAR(255),
  timestamp_reported TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  -- Recovery & Resolution
  reported_to_manager_id UUID REFERENCES auth.users(id),
  manager_notes TEXT,
  is_resolved BOOLEAN DEFAULT false,
  resolved_at TIMESTAMP,
  
  -- Cost Impact
  estimated_cost DECIMAL(10, 2),  -- Calculate from ingredient cost_per_unit * quantity_wasted
  batch_id VARCHAR(50),  -- If part of batch spoilage
  
  -- Metadata
  location_in_kitchen VARCHAR(100),  -- Fridge, Freezer, Dry Storage, Prep Area
  shift ENUM ('morning', 'afternoon', 'night') DEFAULT 'morning',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Index for fast queries
CREATE INDEX idx_waste_logs_timestamp ON waste_logs(timestamp_reported DESC);
CREATE INDEX idx_waste_logs_ingredient ON waste_logs(ingredient_id);
CREATE INDEX idx_waste_logs_restaurant ON waste_logs(restaurant_id);
```

#### **New Table: `waste_categories`**
```sql
CREATE TABLE waste_categories (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  restaurant_id UUID NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
  
  category_name VARCHAR(100) NOT NULL,  -- 'Vegetable Prep', 'Meat Processing', 'Sauce Prep'
  category_type ENUM ('prep', 'cooking', 'storage', 'holding', 'other'),
  target_waste_percentage DECIMAL(5, 2),  -- e.g., 5.5% expected waste
  description TEXT,
  
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  UNIQUE(restaurant_id, category_name)
);
```

#### **Enhanced: `inventory_movements`** (Already exists with spoilage support)
```sql
-- Existing table now used for waste tracking
-- movement_type: 'in', 'out', 'adjustment', 'spoilage'
-- When waste_log is created → create inventory_movement record with type='spoilage'
```

---

### 2. TypeScript Types

#### **`src/types/waste.ts`** (New File)
```typescript
export interface WasteLog {
  id: string;
  restaurant_id: string;
  ingredient_id: string;
  ingredient_name?: string;
  waste_type: 'spoilage' | 'spillage' | 'other';
  quantity_wasted: number;
  unit: string;
  reason: 'expired' | 'damaged_container' | 'dropped' | 'discolored' | 'odor' | 'other';
  description?: string;
  notes?: string;
  reported_by_user_id: string;
  reported_by_name: string;
  timestamp_reported: string;
  reported_to_manager_id?: string;
  manager_notes?: string;
  is_resolved: boolean;
  resolved_at?: string;
  estimated_cost?: number;
  batch_id?: string;
  location_in_kitchen: 'Fridge' | 'Freezer' | 'Dry Storage' | 'Prep Area' | 'Other';
  shift: 'morning' | 'afternoon' | 'night';
  created_at: string;
  updated_at: string;
}

export interface WasteCategory {
  id: string;
  restaurant_id: string;
  category_name: string;
  category_type: 'prep' | 'cooking' | 'storage' | 'holding' | 'other';
  target_waste_percentage: number;
  description?: string;
  is_active: boolean;
  created_at: string;
  updated_at: string;
}

export interface WasteStats {
  total_waste_today: number;
  total_waste_week: number;
  total_waste_month: number;
  total_cost_today: number;
  total_cost_week: number;
  total_cost_month: number;
  most_wasted_ingredient: string;
  waste_by_type: {
    spoilage: number;
    spillage: number;
    other: number;
  };
  waste_by_reason: Record<string, number>;
}

export interface WasteReportRequest {
  ingredient_id: string;
  waste_type: 'spoilage' | 'spillage' | 'other';
  quantity_wasted: number;
  unit: string;
  reason: string;
  description?: string;
  notes?: string;
  location_in_kitchen: string;
  shift?: 'morning' | 'afternoon' | 'night';
}

export interface ApiResponse<T> {
  success: boolean;
  message: string;
  data?: T;
  error?: string;
}
```

---

## 🎨 UI Components

### 1. **WasteReportModal** (New Component)
**File:** `src/components/Kitchen/WasteReportModal.tsx`

```typescript
interface WasteReportModalProps {
  isOpen: boolean;
  onClose: () => void;
  onSubmit: (data: WasteReportRequest) => Promise<void>;
  ingredients: Ingredient[];
  isLoading?: boolean;
}

// Features:
// - Form with ingredient selector
// - Waste type selector (spoilage/spillage/other)
// - Quantity input with unit dropdown
// - Reason dropdown with predefined options
// - Optional description textarea
// - Kitchen location selector
// - Shift indicator
// - Estimated cost display (auto-calculated)
// - Submit button with loading state
```

**Form Fields:**
```
┌─ Waste Report ─────────────────────────────┐
│                                             │
│ Ingredient: [Dropdown - search & select]   │
│ Current Stock: 12.5 kg                      │
│                                             │
│ Waste Type: ○ Spoilage ○ Spillage ○ Other │
│                                             │
│ Quantity Wasted: [5.0]                      │
│ Unit: [kg         ▼]                        │
│                                             │
│ Reason: [expired           ▼]              │
│ • expired                                   │
│ • damaged_container                         │
│ • dropped                                   │
│ • discolored                                │
│ • odor                                      │
│ • other                                     │
│                                             │
│ Location: [Fridge          ▼]              │
│                                             │
│ Shift: ○ Morning ○ Afternoon ○ Night       │
│                                             │
│ Description: [textarea                    ]│
│                                             │
│ Estimated Cost: ₱125.00 (5kg × ₱25/kg)    │
│                                             │
│ [Cancel]                    [Report Waste] │
└─────────────────────────────────────────────┘
```

### 2. **WasteReportCard** (New Component)
**File:** `src/components/Kitchen/WasteReportCard.tsx`

```typescript
interface WasteReportCardProps {
  report: WasteLog;
  onEdit?: () => void;
  onResolve?: () => void;
  showActions?: boolean;
}

// Displays:
// - Ingredient name with icon
// - Waste type & reason badges
// - Quantity wasted
// - Reported by & timestamp
// - Estimated cost (red highlight if high)
// - Status badge (pending/resolved)
// - Resolve button (if pending)
```

### 3. **WasteDashboardTab** (New Component - Part of Kitchen Dashboard)
**File:** `src/components/Kitchen/WasteDashboardTab.tsx`

Integrate into KitchenDashboard as 4th tab.

**Features:**
- **Summary Stats Cards:**
  - Total waste today (qty & cost)
  - Total waste this week
  - Total waste this month
  - Top wasted ingredient

- **Waste Reports List:**
  - Searchable/filterable by ingredient, reason, shift
  - Time range selector (Today, This Week, This Month)
  - Sortable by date, cost, ingredient
  - Pagination (20 per page)
  - Quick resolve button on pending reports

- **Waste Breakdown Charts:**
  - Pie chart: Waste by type (spoilage vs spillage)
  - Bar chart: Top wasted ingredients (by quantity)
  - Bar chart: Waste by reason
  - Line chart: Daily waste trend

- **Export Report:**
  - Export as Excel (date range selectable)
  - Generate PDF waste summary for management

---

## 🔌 API Endpoints

### Waste Reporting Endpoints

```typescript
// POST - Report waste incident
POST /api/waste/report
RequestBody: WasteReportRequest
Response: { success, data: WasteLog, message }

// GET - List waste reports (paginated)
GET /api/waste/reports?page=1&limit=20&ingredient_id=&reason=&from_date=&to_date=
Response: { success, data: WasteLog[], pagination, stats }

// GET - Waste statistics
GET /api/waste/stats?period=day|week|month
Response: { success, data: WasteStats }

// GET - Waste report by ID
GET /api/waste/reports/:id
Response: { success, data: WasteLog }

// PUT - Resolve waste report
PUT /api/waste/reports/:id/resolve
RequestBody: { manager_notes?: string }
Response: { success, data: WasteLog, message }

// PUT - Update waste report
PUT /api/waste/reports/:id
RequestBody: Partial<WasteReportRequest>
Response: { success, data: WasteLog }

// DELETE - Delete waste report (admin only)
DELETE /api/waste/reports/:id
Response: { success, message }

// GET - Export waste reports
GET /api/waste/export?format=excel|pdf&from_date=&to_date=
Response: File (Excel/PDF)

// GET - Waste trends
GET /api/waste/trends?metric=quantity|cost&period=week|month|quarter
Response: { success, data: ChartData[] }
```

---

## 🔄 Integration Points

### 1. Kitchen Dashboard Enhancement
**File:** `src/components/Kitchen/KitchenDashboard.tsx`

```typescript
// Add new state
const [wasteTab, setWasteTab] = useState(false);

// Add 4th tab to tab navigation
{
  id: 'waste',
  label: 'Waste & Spoilage',
  icon: AlertTriangle,
  badge: wasteReportsPending  // Show count of unresolved reports
}

// Add "Report Waste" button in sticky header (next to Refresh)
<button onClick={() => setShowWasteModal(true)}>
  <Trash2 className="h-4 w-4" />
  Report Waste
</button>

// Add WasteReportModal
{showWasteModal && (
  <WasteReportModal
    isOpen={showWasteModal}
    onClose={() => setShowWasteModal(false)}
    onSubmit={handleReportWaste}
    ingredients={ingredients}
  />
)}

// Add WasteDashboardTab content
{activeTab === 'waste' && (
  <WasteDashboardTab 
    onReportWaste={() => setShowWasteModal(true)}
  />
)}
```

### 2. Inventory Deduction Logic
When waste is reported:
```typescript
// Auto-create inventory_movement record
const movement = await api.inventory.createMovement({
  ingredient_id: wasteReport.ingredient_id,
  movement_type: 'spoilage',
  quantity_moved: wasteReport.quantity_wasted,
  reference_id: wasteReport.id,
  notes: `${wasteReport.waste_type}: ${wasteReport.reason}`
});

// This will automatically update ingredient.current_stock
// Trigger low-stock alert if stock falls below threshold
```

### 3. Notification System
```typescript
// When waste reported
addNotification({
  type: 'waste_reported',
  message: `${ingredient.name}: ${qty} ${unit} wasted (${reason})`,
  severity: 'warning',
  icon: AlertTriangle
});

// If stock drops critical
addNotification({
  type: 'low_stock_alert',
  message: `${ingredient.name} stock now CRITICAL: ${newStock} ${unit}`,
  severity: 'error',
  icon: AlertCircle
});
```

---

## 📊 Implementation Phases

### Phase 1: Core Reporting (Week 1)
**Priority:** HIGH
- [x] Database schema creation (spoilage already in schema)
- [ ] Create `waste.ts` types file
- [ ] Create `WasteReportModal` component
- [ ] Create API endpoints for reporting waste
- [ ] Integrate "Report Waste" button into Kitchen Dashboard
- [ ] Test waste reporting end-to-end
- **Deliverable:** Kitchen staff can report waste incidents

### Phase 2: Dashboard & Analytics (Week 2)
**Priority:** HIGH
- [ ] Create `WasteDashboardTab` component
- [ ] Add waste stats calculations
- [ ] Create summary stat cards
- [ ] Implement waste reports list with pagination
- [ ] Add filtering by ingredient, reason, shift, date range
- [ ] Test pagination and filtering
- **Deliverable:** Real-time waste tracking dashboard

### Phase 3: Reporting & Export (Week 3)
**Priority:** MEDIUM
- [ ] Create waste trend charts (Recharts or Chart.js)
- [ ] Implement Excel export functionality
- [ ] Implement PDF report generation
- [ ] Create waste comparison views (vs. targets)
- [ ] Test export functionality
- **Deliverable:** Management reports for cost analysis

### Phase 4: Advanced Features (Week 4)
**Priority:** MEDIUM
- [ ] Implement waste category management UI
- [ ] Add bulk resolve functionality
- [ ] Create waste alerts/notifications system
- [ ] Implement manager approval workflow
- [ ] Create waste trend predictions
- **Deliverable:** Full waste management suite

### Phase 5: Optimization & Polish (Week 5)
**Priority:** LOW
- [ ] Performance optimization (query indexing)
- [ ] Mobile UI refinement
- [ ] Accessibility audit
- [ ] Documentation
- [ ] User training materials
- **Deliverable:** Production-ready feature

---

## 🛠️ Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Frontend** | React + TypeScript | UI components |
| **State** | useState/useCallback | Local component state |
| **Styling** | Tailwind CSS | Responsive design |
| **Icons** | Lucide React | Visual indicators |
| **Charts** | Recharts | Waste trend visualization |
| **Export** | XLSX library | Excel export |
| **PDF** | html2pdf | PDF report generation |
| **Backend** | Node.js + Express | API endpoints |
| **Database** | PostgreSQL | Data persistence |
| **Validation** | Joi/Zod | Form validation |

---

## 📝 File Structure

```
src/
├── components/
│   └── Kitchen/
│       ├── KitchenDashboard.tsx (enhanced)
│       ├── WasteReportModal.tsx (new)
│       ├── WasteReportCard.tsx (new)
│       ├── WasteDashboardTab.tsx (new)
│       └── WasteChart.tsx (new)
│
├── hooks/
│   └── useWasteManagement.ts (new)
│
├── types/
│   └── waste.ts (new)
│
├── utils/
│   ├── wasteApi.ts (new)
│   └── wasteCalculations.ts (new)
│
└── data/
    └── wasteReasons.ts (new)

Database/
├── migrations/
│   └── 001_create_waste_tables.sql
└── functions/
    └── calculate_waste_impact.sql
```

---

## 🔐 Security & Permissions

### Role-Based Access Control

| Action | Kitchen Staff | Manager | Admin |
|--------|---------------|---------|-------|
| Report Waste | ✅ | ✅ | ✅ |
| View Own Reports | ✅ | ✅ | ✅ |
| View All Reports | ❌ | ✅ | ✅ |
| Resolve Reports | ❌ | ✅ | ✅ |
| Edit Reports | ❌ | ✅ | ✅ |
| Delete Reports | ❌ | ❌ | ✅ |
| Export Data | ❌ | ✅ | ✅ |
| Manage Categories | ❌ | ❌ | ✅ |

---

## ✅ Acceptance Criteria

### Core Features
- [ ] Kitchen staff can report waste with ingredient, quantity, reason
- [ ] Waste reports appear in new "Waste & Spoilage" dashboard tab
- [ ] Reported waste automatically deducts from ingredient inventory
- [ ] Low-stock alerts trigger when waste causes depletion
- [ ] Reports show: what, when, who, estimated cost

### Analytics & Reporting
- [ ] Dashboard displays waste statistics (daily, weekly, monthly)
- [ ] Charts show waste by type, ingredient, and reason
- [ ] Excel export includes all waste data with calculations
- [ ] PDF reports available for management

### User Experience
- [ ] "Report Waste" button visible in Kitchen Dashboard header
- [ ] Modal form is intuitive and mobile-friendly
- [ ] Estimated cost auto-calculated from ingredient data
- [ ] Real-time updates when new waste is reported
- [ ] Notifications alert relevant staff

### Performance
- [ ] Waste report submission < 1s
- [ ] Dashboard loads within 2s
- [ ] Export of 1000 records < 5s
- [ ] No API timeouts

### Security
- [ ] Only authenticated users can report waste
- [ ] Staff can only resolve own reports or manager permission
- [ ] Audit trail maintained (who, what, when, why)
- [ ] Data encryption for cost information

---

## 📈 Future Enhancements

1. **Predictive Analytics:**
   - ML model to predict waste rates per ingredient
   - Anomaly detection for unusual waste patterns

2. **IoT Integration:**
   - Real-time temperature monitoring (spoilage detection)
   - Weight sensors on storage to auto-detect spillage

3. **Supplier Management:**
   - Link waste reports to supplier quality issues
   - Auto-notify supplier of quality concerns

4. **Cost Optimization:**
   - Waste cost analysis vs. profit margins
   - Supplier quality metrics for procurement decisions

5. **Sustainability Reporting:**
   - Carbon footprint calculation from waste
   - Sustainability compliance reports

6. **Mobile App:**
   - Native iOS/Android app for kitchen staff
   - Push notifications for low-stock alerts

---

## 🚀 Getting Started

### Prerequisites
- [ ] Database migration files created
- [ ] API endpoints implemented
- [ ] TypeScript types defined
- [ ] Tailwind CSS configured

### Setup Commands
```bash
# 1. Run database migration
npm run migrate:up

# 2. Create types file
touch src/types/waste.ts

# 3. Create component directory files
touch src/components/Kitchen/WasteReportModal.tsx
touch src/components/Kitchen/WasteDashboardTab.tsx
touch src/components/Kitchen/WasteReportCard.tsx

# 4. Create hook
touch src/hooks/useWasteManagement.ts

# 5. Install dependencies (if needed)
npm install xlsx html2pdf recharts

# 6. Start development server
npm run dev
```

---

## 📞 Support & Questions

**Questions during implementation?**
- Refer to kitchen.ts types for order/ingredient structure
- Check existing API patterns in api.ts
- Review Kitchen Dashboard for UI patterns
- Check Inventory components for similar workflows

**Blockers?**
- API endpoints not ready → mock data in useWasteManagement hook
- Database migration issues → refer to SUPABASE_COMPLETE_SCHEMA.sql
- UI/UX questions → reference OrderHistory.tsx for patterns

---

## 📋 Checklist for Implementation

- [ ] Phase 1: Core Reporting
- [ ] Phase 2: Dashboard & Analytics
- [ ] Phase 3: Reporting & Export
- [ ] Phase 4: Advanced Features
- [ ] Phase 5: Optimization & Polish
- [ ] Testing & QA
- [ ] User Training
- [ ] Production Deployment

---

**Document Version:** 1.0  
**Last Updated:** November 13, 2025  
**Created By:** GitHub Copilot  
**Status:** Ready for Implementation
