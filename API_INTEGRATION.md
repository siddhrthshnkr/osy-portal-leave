# API Integration Documentation

This document outlines the backend API endpoints and integration requirements for the Leaves Management component.

## Required API Endpoints

### 1. Get Leave Requests
```
GET /api/leave-requests
```

**Query Parameters:**
- `filter` (optional): 'upcoming' | 'past' | 'all' (default: 'all')
- `employee_id` (optional): Filter by specific employee
- `status` (optional): 'pending' | 'approved' | 'rejected'

**Response:**
```json
{
  "data": [
    {
      "id": 1,
      "employee_id": "EMP001",
      "employee_name": "Sarah Johnson",
      "leave_type": "Annual Leave",
      "billing_type": "Paid",
      "start_date": "2025-09-15",
      "end_date": "2025-09-19",
      "duration": 5,
      "duration_type": "days",
      "status": "Pending",
      "approved_by": null,
      "rejected_by": null,
      "approved_at": null,
      "rejected_at": null,
      "reason": "Family vacation",
      "team_members": ["1", "2"],
      "created_at": "2025-08-20T10:30:00Z",
      "updated_at": "2025-08-20T10:30:00Z"
    }
  ],
  "meta": {
    "total": 25,
    "per_page": 50,
    "current_page": 1
  }
}
```

### 2. Approve Leave Request
```
POST /api/leave-requests/{id}/approve
```

**Request Body:**
```json
{
  "approver_id": 1,
  "approved_at": "2025-08-25T10:30:00Z"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Leave request approved successfully",
  "data": {
    "id": 1,
    "status": "Approved",
    "approved_by": "Admin User",
    "approved_at": "2025-08-25T10:30:00Z",
    "zoho_people_updated": true,
    "email_sent": true
  }
}
```

**Required Actions:**
1. Update leave request status in database
2. Call Zoho People API to approve leave
3. Log approval action with user ID and timestamp
4. Send email notification to operations team
5. Return updated leave request data

### 3. Reject Leave Request
```
POST /api/leave-requests/{id}/reject
```

**Request Body:**
```json
{
  "rejector_id": 1,
  "rejected_at": "2025-08-25T10:30:00Z",
  "rejection_reason": "Insufficient coverage during requested period"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Leave request rejected successfully",
  "data": {
    "id": 1,
    "status": "Rejected",
    "rejected_by": "Admin User",
    "rejected_at": "2025-08-25T10:30:00Z",
    "rejection_reason": "Insufficient coverage during requested period",
    "zoho_people_updated": true,
    "email_sent": true
  }
}
```

## Laravel Implementation Example

### 1. Route Definition
```php
// routes/api.php
Route::middleware(['auth:sanctum'])->group(function () {
    Route::get('/leave-requests', [LeaveRequestController::class, 'index']);
    Route::post('/leave-requests/{id}/approve', [LeaveRequestController::class, 'approve']);
    Route::post('/leave-requests/{id}/reject', [LeaveRequestController::class, 'reject']);
});
```

### 2. Controller Implementation
```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\LeaveRequest;
use App\Services\ZohoPeopleService;
use App\Services\EmailNotificationService;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;

class LeaveRequestController extends Controller
{
    protected $zohoPeopleService;
    protected $emailService;

    public function __construct(
        ZohoPeopleService $zohoPeopleService,
        EmailNotificationService $emailService
    ) {
        $this->zohoPeopleService = $zohoPeopleService;
        $this->emailService = $emailService;
    }

    public function index(Request $request)
    {
        $user = $request->user();
        $query = LeaveRequest::with(['employee', 'approver', 'rejector']);

        // Apply role-based filtering
        if (!$user->has_admin_access) {
            if ($user->is_representative) {
                // Representatives see only their assigned team members
                $teamMemberIds = $user->teamMembers->pluck('id');
                $query->whereIn('employee_id', $teamMemberIds);
            } else {
                // Regular users see only their own requests
                $query->where('employee_id', $user->id);
            }
        }

        // Apply filters
        if ($request->has('filter')) {
            $filter = $request->get('filter');
            $now = now();
            
            if ($filter === 'upcoming') {
                $query->where(function ($q) use ($now) {
                    $q->where('start_date', '>', $now)
                      ->orWhere(function ($subQ) use ($now) {
                          $subQ->where('start_date', '<=', $now)
                               ->where('end_date', '>', $now);
                      });
                });
            } elseif ($filter === 'past') {
                $query->where('end_date', '<', $now);
            }
        }

        if ($request->has('status')) {
            $query->where('status', $request->get('status'));
        }

        $leaveRequests = $query->orderBy('start_date', 'desc')->paginate(50);

        return response()->json($leaveRequests);
    }

    public function approve(Request $request, $id)
    {
        $request->validate([
            'approver_id' => 'required|integer',
            'approved_at' => 'required|date'
        ]);

        $user = $request->user();
        $leaveRequest = LeaveRequest::findOrFail($id);

        // Check permissions
        if (!$this->canUserApproveRequest($user, $leaveRequest)) {
            return response()->json([
                'success' => false,
                'message' => 'Insufficient permissions to approve this request'
            ], 403);
        }

        DB::beginTransaction();
        
        try {
            // Update leave request
            $leaveRequest->update([
                'status' => 'Approved',
                'approved_by' => $request->approver_id,
                'approved_at' => $request->approved_at
            ]);

            // Update in Zoho People
            $zohoResponse = $this->zohoPeopleService->approveLeave(
                $leaveRequest->zoho_leave_id,
                $request->approver_id
            );

            if (!$zohoResponse['success']) {
                throw new \Exception('Failed to update leave in Zoho People: ' . $zohoResponse['message']);
            }

            // Send email notification
            $this->emailService->sendLeaveApprovalNotification($leaveRequest, $user);

            // Log the action
            activity()
                ->causedBy($user)
                ->performedOn($leaveRequest)
                ->withProperties([
                    'action' => 'approved',
                    'approver_id' => $request->approver_id,
                    'approved_at' => $request->approved_at
                ])
                ->log('Leave request approved');

            DB::commit();

            return response()->json([
                'success' => true,
                'message' => 'Leave request approved successfully',
                'data' => [
                    'id' => $leaveRequest->id,
                    'status' => $leaveRequest->status,
                    'approved_by' => $user->firstname . ' ' . $user->lastname,
                    'approved_at' => $leaveRequest->approved_at,
                    'zoho_people_updated' => true,
                    'email_sent' => true
                ]
            ]);

        } catch (\Exception $e) {
            DB::rollback();
            
            return response()->json([
                'success' => false,
                'message' => 'Failed to approve leave request: ' . $e->getMessage()
            ], 500);
        }
    }

    public function reject(Request $request, $id)
    {
        $request->validate([
            'rejector_id' => 'required|integer',
            'rejected_at' => 'required|date',
            'rejection_reason' => 'nullable|string|max:500'
        ]);

        $user = $request->user();
        $leaveRequest = LeaveRequest::findOrFail($id);

        // Check permissions
        if (!$this->canUserApproveRequest($user, $leaveRequest)) {
            return response()->json([
                'success' => false,
                'message' => 'Insufficient permissions to reject this request'
            ], 403);
        }

        DB::beginTransaction();
        
        try {
            // Update leave request
            $leaveRequest->update([
                'status' => 'Rejected',
                'rejected_by' => $request->rejector_id,
                'rejected_at' => $request->rejected_at,
                'rejection_reason' => $request->rejection_reason
            ]);

            // Update in Zoho People
            $zohoResponse = $this->zohoPeopleService->rejectLeave(
                $leaveRequest->zoho_leave_id,
                $request->rejector_id,
                $request->rejection_reason
            );

            if (!$zohoResponse['success']) {
                throw new \Exception('Failed to update leave in Zoho People: ' . $zohoResponse['message']);
            }

            // Send email notification
            $this->emailService->sendLeaveRejectionNotification($leaveRequest, $user);

            // Log the action
            activity()
                ->causedBy($user)
                ->performedOn($leaveRequest)
                ->withProperties([
                    'action' => 'rejected',
                    'rejector_id' => $request->rejector_id,
                    'rejected_at' => $request->rejected_at,
                    'rejection_reason' => $request->rejection_reason
                ])
                ->log('Leave request rejected');

            DB::commit();

            return response()->json([
                'success' => true,
                'message' => 'Leave request rejected successfully',
                'data' => [
                    'id' => $leaveRequest->id,
                    'status' => $leaveRequest->status,
                    'rejected_by' => $user->firstname . ' ' . $user->lastname,
                    'rejected_at' => $leaveRequest->rejected_at,
                    'rejection_reason' => $leaveRequest->rejection_reason,
                    'zoho_people_updated' => true,
                    'email_sent' => true
                ]
            ]);

        } catch (\Exception $e) {
            DB::rollback();
            
            return response()->json([
                'success' => false,
                'message' => 'Failed to reject leave request: ' . $e->getMessage()
            ], 500);
        }
    }

    private function canUserApproveRequest($user, $leaveRequest)
    {
        // Admins can approve all requests
        if ($user->has_admin_access) {
            return true;
        }
        
        // Representatives can approve requests from their team members
        if ($user->is_representative) {
            return $user->teamMembers->contains('id', $leaveRequest->employee_id);
        }
        
        return false;
    }
}
```

### 3. Zoho People Service
```php
<?php

namespace App\Services;

use GuzzleHttp\Client;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Log;

class ZohoPeopleService
{
    protected $client;
    protected $baseUrl;
    protected $clientId;
    protected $clientSecret;
    protected $refreshToken;

    public function __construct()
    {
        $this->client = new Client();
        $this->baseUrl = config('services.zoho.people.base_url');
        $this->clientId = config('services.zoho.people.client_id');
        $this->clientSecret = config('services.zoho.people.client_secret');
        $this->refreshToken = config('services.zoho.people.refresh_token');
    }

    public function approveLeave($zohoLeaveId, $approverId)
    {
        try {
            $accessToken = $this->getAccessToken();
            
            $response = $this->client->post("{$this->baseUrl}/leave/{$zohoLeaveId}/approve", [
                'headers' => [
                    'Authorization' => "Zoho-oauthtoken {$accessToken}",
                    'Content-Type' => 'application/json',
                ],
                'json' => [
                    'approver_id' => $approverId,
                    'approved_at' => now()->toISOString(),
                    'status' => 'Approved'
                ]
            ]);

            $responseData = json_decode($response->getBody(), true);
            
            Log::info('Zoho People leave approval', [
                'zoho_leave_id' => $zohoLeaveId,
                'approver_id' => $approverId,
                'response' => $responseData
            ]);

            return [
                'success' => $response->getStatusCode() === 200,
                'data' => $responseData
            ];

        } catch (\Exception $e) {
            Log::error('Zoho People leave approval failed', [
                'zoho_leave_id' => $zohoLeaveId,
                'error' => $e->getMessage()
            ]);

            return [
                'success' => false,
                'message' => $e->getMessage()
            ];
        }
    }

    public function rejectLeave($zohoLeaveId, $rejectorId, $reason = null)
    {
        try {
            $accessToken = $this->getAccessToken();
            
            $response = $this->client->post("{$this->baseUrl}/leave/{$zohoLeaveId}/reject", [
                'headers' => [
                    'Authorization' => "Zoho-oauthtoken {$accessToken}",
                    'Content-Type' => 'application/json',
                ],
                'json' => [
                    'rejector_id' => $rejectorId,
                    'rejected_at' => now()->toISOString(),
                    'status' => 'Rejected',
                    'reason' => $reason
                ]
            ]);

            $responseData = json_decode($response->getBody(), true);
            
            Log::info('Zoho People leave rejection', [
                'zoho_leave_id' => $zohoLeaveId,
                'rejector_id' => $rejectorId,
                'reason' => $reason,
                'response' => $responseData
            ]);

            return [
                'success' => $response->getStatusCode() === 200,
                'data' => $responseData
            ];

        } catch (\Exception $e) {
            Log::error('Zoho People leave rejection failed', [
                'zoho_leave_id' => $zohoLeaveId,
                'error' => $e->getMessage()
            ]);

            return [
                'success' => false,
                'message' => $e->getMessage()
            ];
        }
    }

    private function getAccessToken()
    {
        return Cache::remember('zoho_access_token', 3500, function () {
            $response = $this->client->post('https://accounts.zoho.com/oauth/v2/token', [
                'form_params' => [
                    'refresh_token' => $this->refreshToken,
                    'client_id' => $this->clientId,
                    'client_secret' => $this->clientSecret,
                    'grant_type' => 'refresh_token'
                ]
            ]);

            $data = json_decode($response->getBody(), true);
            return $data['access_token'];
        });
    }
}
```

### 4. Email Notification Service
```php
<?php

namespace App\Services;

use App\Models\LeaveRequest;
use App\Models\User;
use Illuminate\Support\Facades\Mail;

class EmailNotificationService
{
    public function sendLeaveApprovalNotification(LeaveRequest $leaveRequest, User $approver)
    {
        $operationsTeam = config('mail.operations_team_email');
        
        Mail::send('emails.leave-approved', [
            'employee_name' => $leaveRequest->employee_name,
            'leave_type' => $leaveRequest->leave_type,
            'start_date' => $leaveRequest->start_date,
            'end_date' => $leaveRequest->end_date,
            'duration' => $leaveRequest->duration,
            'duration_type' => $leaveRequest->duration_type,
            'approver_name' => $approver->firstname . ' ' . $approver->lastname,
            'approved_at' => $leaveRequest->approved_at
        ], function ($message) use ($operationsTeam, $leaveRequest) {
            $message->to($operationsTeam)
                   ->subject("Leave Request Approved - {$leaveRequest->employee_name}");
        });
    }

    public function sendLeaveRejectionNotification(LeaveRequest $leaveRequest, User $rejector)
    {
        $operationsTeam = config('mail.operations_team_email');
        
        Mail::send('emails.leave-rejected', [
            'employee_name' => $leaveRequest->employee_name,
            'leave_type' => $leaveRequest->leave_type,
            'start_date' => $leaveRequest->start_date,
            'end_date' => $leaveRequest->end_date,
            'duration' => $leaveRequest->duration,
            'duration_type' => $leaveRequest->duration_type,
            'rejector_name' => $rejector->firstname . ' ' . $rejector->lastname,
            'rejected_at' => $leaveRequest->rejected_at,
            'rejection_reason' => $leaveRequest->rejection_reason
        ], function ($message) use ($operationsTeam, $leaveRequest) {
            $message->to($operationsTeam)
                   ->subject("Leave Request Rejected - {$leaveRequest->employee_name}");
        });
    }
}
```

## Database Schema

### Leave Requests Table
```sql
CREATE TABLE leave_requests (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    employee_id VARCHAR(255) NOT NULL,
    employee_name VARCHAR(255) NOT NULL,
    zoho_leave_id VARCHAR(255) NULL,
    leave_type VARCHAR(255) NOT NULL,
    billing_type ENUM('Paid', 'Unpaid') NOT NULL DEFAULT 'Paid',
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    duration DECIMAL(8,2) NOT NULL,
    duration_type ENUM('days', 'hours') NOT NULL DEFAULT 'days',
    status ENUM('Pending', 'Approved', 'Rejected') NOT NULL DEFAULT 'Pending',
    reason TEXT NULL,
    approved_by BIGINT UNSIGNED NULL,
    rejected_by BIGINT UNSIGNED NULL,
    approved_at TIMESTAMP NULL,
    rejected_at TIMESTAMP NULL,
    rejection_reason TEXT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_employee_id (employee_id),
    INDEX idx_status (status),
    INDEX idx_start_date (start_date),
    INDEX idx_end_date (end_date),
    FOREIGN KEY (approved_by) REFERENCES users(id),
    FOREIGN KEY (rejected_by) REFERENCES users(id)
);
```

## Configuration

### Environment Variables
```env
ZOHO_PEOPLE_BASE_URL=https://people.zoho.com/api
ZOHO_PEOPLE_CLIENT_ID=your_client_id
ZOHO_PEOPLE_CLIENT_SECRET=your_client_secret
ZOHO_PEOPLE_REFRESH_TOKEN=your_refresh_token

OPERATIONS_TEAM_EMAIL=operations@company.com
```

### Config Files
```php
// config/services.php
'zoho' => [
    'people' => [
        'base_url' => env('ZOHO_PEOPLE_BASE_URL'),
        'client_id' => env('ZOHO_PEOPLE_CLIENT_ID'),
        'client_secret' => env('ZOHO_PEOPLE_CLIENT_SECRET'),
        'refresh_token' => env('ZOHO_PEOPLE_REFRESH_TOKEN'),
    ]
],

// config/mail.php
'operations_team_email' => env('OPERATIONS_TEAM_EMAIL', 'operations@company.com'),
```

This implementation provides a complete backend API that integrates with Zoho People and handles email notifications as required by the component specifications.