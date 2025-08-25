# Leaves Management Component

A Vue.js component for managing employee leave requests in the Outsourcey Portal. This component follows the same design structure as the attendance dashboard and provides functionality for viewing, filtering, and approving/rejecting leave requests.

## Features

- **Role-based access control**: Admins can see all requests, Representatives see only their assigned team members
- **Filtering**: View "Upcoming" and "Past" leave requests
- **Approve/Reject functionality**: With API integration to Zoho People and email notifications
- **Responsive design**: Follows the attendance dashboard design patterns
- **Real-time updates**: Processing states and success/error feedback

## Component Structure

### Props

```typescript
interface LeaveRequest {
  id: number;
  employee_id: string;
  employee_name: string;
  leave_type: string;
  billing_type: 'Paid' | 'Unpaid';
  start_date: string;
  end_date: string;
  duration: number;
  duration_type: 'days' | 'hours';
  status: 'Pending' | 'Approved' | 'Rejected';
  approved_by?: string;
  rejected_by?: string;
  approved_at?: string;
  rejected_at?: string;
  reason?: string;
  team_members?: string[]; // For representatives
}
```

### Required Props

- `leave_requests`: Array of leave request objects
- `user`: Current user object with access permissions

## API Endpoints Expected

The component expects these API endpoints to be implemented:

### 1. Approve Leave Request
```
POST /api/leave-requests/{id}/approve
Body: {
  approver_id: number,
  approved_at: string
}
```

**Functionality:**
- Approve the leave in Zoho People
- Log the approver's user ID and timestamp
- Send email notification to operations team
- Return success/failure response

### 2. Reject Leave Request
```
POST /api/leave-requests/{id}/reject
Body: {
  rejector_id: number,
  rejected_at: string
}
```

**Functionality:**
- Reject the leave in Zoho People
- Log the rejector's user ID and timestamp
- Send email notification to operations team
- Return success/failure response

## Usage

### 1. In Laravel/Inertia.js Application

```php
// Controller
use Inertia\Inertia;

public function index(Request $request)
{
    $user = $request->user();
    $leaveRequests = $this->getLeaveRequestsForUser($user);
    
    return Inertia::render('Leaves/Management', [
        'leave_requests' => $leaveRequests,
        'user' => $user
    ]);
}
```

```vue
<!-- In your Vue application -->
<template>
  <LeavesManagement 
    :leave_requests="leave_requests"
    :user="user"
  />
</template>

<script setup>
import LeavesManagement from './LeavesManagement.vue'

const props = defineProps(['leave_requests', 'user'])
</script>
```

### 2. Standalone Testing

You can test the component using the included `index.html` file:

```bash
# Install dependencies
npm install

# Start development server
npm run dev

# Or open index.html directly in a browser for static testing
```

## Design Guidelines

The component follows the same design patterns as the attendance dashboard:

- **Typography**: DM Sans for headers, Inter for content
- **Colors**: 
  - Primary: #014c85
  - Background: #ffffff
  - Text: #0a0a0a
  - Secondary: rgba(10, 10, 10, 0.6)
- **Status Colors**:
  - Approved: #d4edda (bg), #155724 (text)
  - Rejected: #f8d7da (bg), #721c24 (text)
  - Pending: #fff3cd (bg), #856404 (text)

## Responsive Behavior

- **Desktop (>1200px)**: Full table with all columns
- **Tablet (768-1200px)**: Condensed table with hidden columns
- **Mobile (<768px)**: Single column stack layout

## Access Control

- **Admins**: Can see and approve/reject all leave requests
- **Representatives**: Can only see and approve/reject requests from their assigned team members
- **Regular Users**: View-only access (if implemented)

## Integration Notes

### Zoho People Integration

The approve/reject API endpoints should:

1. Make API calls to Zoho People to update leave status
2. Handle authentication with Zoho People API
3. Log all approval/rejection actions with timestamps
4. Send email notifications to the operations team

### Email Notifications

When a leave request is approved or rejected:
- Send notification to operations team
- Include employee details, leave dates, and action taken
- Include approver/rejector information

### Error Handling

The component handles:
- Network errors during API calls
- Processing states during requests
- Success/failure feedback to users
- Disabled states during processing

## Deployment

For Vercel deployment:

1. Ensure all dependencies are installed
2. Build the application: `npm run build`
3. Deploy the `dist` folder to Vercel
4. Configure environment variables for API endpoints

The component is designed to be easily integrated into existing Laravel/Inertia.js applications while also being testable as a standalone component.