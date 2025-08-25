<template>
  <div class="leaves-management">
    <!-- Header -->
    <div class="header-section">
      <h1 class="page-title">Leaves</h1>
      
      <div class="header-controls">
        <!-- Filter Buttons -->
        <div class="filter-buttons">
          <button 
            @click="currentFilter = 'all'"
            :class="currentFilter === 'all' 
              ? 'btn-filter active' 
              : 'btn-filter'"
          >
            All Requests
          </button>
          <button 
            @click="currentFilter = 'upcoming'"
            :class="currentFilter === 'upcoming' 
              ? 'btn-filter active' 
              : 'btn-filter'"
          >
            Upcoming
          </button>
          <button 
            @click="currentFilter = 'past'"
            :class="currentFilter === 'past' 
              ? 'btn-filter active' 
              : 'btn-filter'"
          >
            Past
          </button>
        </div>
      </div>
    </div>

    <!-- Content Section -->
    <div class="content-section">
      <!-- Desktop & Tablet Table View -->
      <div class="desktop-table">
        <div class="table-container">
          <!-- Table Header -->
          <div class="table-header">
            <div class="table-header-grid">
              <div>Team Member</div>
              <div>Leave Type</div>
              <div class="hide-on-tablet">Billing Type</div>
              <div>Leave Period</div>
              <div class="hide-on-tablet">Days/Hours Taken</div>
              <div>Status</div>
              <div>Action</div>
            </div>
          </div>

          <!-- Table Body -->
          <div class="table-body">
            <div 
              v-for="request in filteredRequests" 
              :key="request.id"
              class="table-row"
            >
              <div class="table-row-grid">
                <!-- Team Member -->
                <div class="employee-info">
                  <div class="employee-avatar">
                    {{ getInitials(request.employee_name) }}
                  </div>
                  <div class="employee-details">
                    <div class="employee-name">{{ request.employee_name }}</div>
                    <div class="employee-id">{{ request.employee_id }}</div>
                  </div>
                </div>

                <!-- Leave Type -->
                <div class="cell-content">{{ request.leave_type }}</div>

                <!-- Billing Type -->
                <div class="hide-on-tablet">
                  <span :class="getBillingTypeClass(request.billing_type)" class="status-badge">
                    {{ request.billing_type }}
                  </span>
                </div>

                <!-- Leave Period -->
                <div class="date-range">
                  <div class="date-start">{{ formatDate(request.start_date) }}</div>
                  <div class="date-to">to {{ formatDate(request.end_date) }}</div>
                </div>

                <!-- Days/Hours Taken -->
                <div class="duration hide-on-tablet">
                  {{ request.duration_type === 'days' ? `${request.duration} days` : `${request.duration} hours` }}
                </div>

                <!-- Status -->
                <div>
                  <span :class="getStatusClass(request.status)" class="status-badge">
                    {{ request.status }}
                  </span>
                </div>

                <!-- Action -->
                <div class="action-buttons">
                  <template v-if="request.status === 'Pending' && canApproveRequest(request)">
                    <button 
                      @click="approveRequest(request.id)"
                      :disabled="processingRequests.has(request.id)"
                      class="btn-approve"
                    >
                      <span v-if="processingRequests.has(request.id)">Processing...</span>
                      <span v-else>Approve</span>
                    </button>
                    <button 
                      @click="showDeclineModal(request)"
                      :disabled="processingRequests.has(request.id)"
                      class="btn-decline"
                    >
                      <span v-if="processingRequests.has(request.id)">Processing...</span>
                      <span v-else>Decline</span>
                    </button>
                  </template>
                  <template v-else-if="request.status === 'Approved'">
                    <div class="action-status approved">
                      ✓ Approved by {{ request.approved_by }}
                    </div>
                  </template>
                  <template v-else-if="request.status === 'Rejected'">
                    <div class="action-status rejected">
                      ✗ Declined by {{ request.rejected_by }}
                    </div>
                  </template>
                  <template v-else>
                    <span class="action-status disabled">No actions available</span>
                  </template>
                </div>
              </div>
            </div>
          </div>

          <!-- Empty State -->
          <div v-if="filteredRequests.length === 0" class="empty-state">
            <div class="empty-state-icon">📄</div>
            <div class="empty-state-title">No {{ currentFilter === 'all' ? '' : currentFilter }} leave requests</div>
            <div class="empty-state-description">
              {{ getEmptyStateMessage() }}
            </div>
          </div>
        </div>
      </div>

      <!-- Mobile Card View -->
      <div class="mobile-cards">
        <div 
          v-for="request in filteredRequests" 
          :key="request.id"
          class="mobile-card"
        >
          <div class="mobile-card-header">
            <div class="employee-info">
              <div class="employee-avatar">
                {{ getInitials(request.employee_name) }}
              </div>
              <div class="employee-details">
                <div class="employee-name">{{ request.employee_name }}</div>
                <div class="employee-id">{{ request.employee_id }}</div>
              </div>
            </div>
            <span :class="getStatusClass(request.status)" class="status-badge">
              {{ request.status }}
            </span>
          </div>

          <div class="mobile-card-body">
            <div class="info-row">
              <span class="info-label">Leave Type:</span>
              <span class="info-value">{{ request.leave_type }}</span>
            </div>
            <div class="info-row">
              <span class="info-label">Billing:</span>
              <span :class="getBillingTypeClass(request.billing_type)" class="status-badge">
                {{ request.billing_type }}
              </span>
            </div>
            <div class="info-row">
              <span class="info-label">Duration:</span>
              <span class="info-value">{{ request.duration_type === 'days' ? `${request.duration} days` : `${request.duration} hours` }}</span>
            </div>
            <div class="info-row">
              <span class="info-label">Period:</span>
              <span class="info-value">{{ formatDate(request.start_date) }} to {{ formatDate(request.end_date) }}</span>
            </div>
          </div>

          <div class="mobile-card-actions" v-if="request.status === 'Pending' && canApproveRequest(request)">
            <button 
              @click="approveRequest(request.id)"
              :disabled="processingRequests.has(request.id)"
              class="btn-approve mobile"
            >
              <span v-if="processingRequests.has(request.id)">Processing...</span>
              <span v-else>Approve</span>
            </button>
            <button 
              @click="showDeclineModal(request)"
              :disabled="processingRequests.has(request.id)"
              class="btn-decline mobile"
            >
              <span v-if="processingRequests.has(request.id)">Processing...</span>
              <span v-else>Decline</span>
            </button>
          </div>

          <div class="mobile-card-status" v-else-if="request.status !== 'Pending'">
            <div v-if="request.status === 'Approved'" class="action-status approved">
              ✓ Approved by {{ request.approved_by }}
            </div>
            <div v-else-if="request.status === 'Rejected'" class="action-status rejected">
              ✗ Declined by {{ request.rejected_by }}
            </div>
          </div>
        </div>

        <!-- Mobile Empty State -->
        <div v-if="filteredRequests.length === 0" class="mobile-empty-state">
          <div class="empty-state-icon">📄</div>
          <div class="empty-state-title">No {{ currentFilter === 'all' ? '' : currentFilter }} leave requests</div>
          <div class="empty-state-description">
            {{ getEmptyStateMessage() }}
          </div>
        </div>
      </div>
    </div>

    <!-- Decline Modal -->
    <div v-if="showDeclinePopup" class="modal-overlay" @click="closeDeclineModal">
      <div class="modal-content" @click.stop>
        <div class="modal-header">
          <h3 class="modal-title">Decline Leave Request</h3>
          <button @click="closeDeclineModal" class="modal-close">
            <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M18 6L6 18M6 6l12 12"></path>
            </svg>
          </button>
        </div>
        
        <div class="modal-body">
          <div v-if="selectedRequest" class="request-summary">
            <div class="summary-item">
              <span class="summary-label">Employee:</span>
              <span class="summary-value">{{ selectedRequest.employee_name }}</span>
            </div>
            <div class="summary-item">
              <span class="summary-label">Leave Type:</span>
              <span class="summary-value">{{ selectedRequest.leave_type }}</span>
            </div>
            <div class="summary-item">
              <span class="summary-label">Period:</span>
              <span class="summary-value">{{ formatDate(selectedRequest.start_date) }} to {{ formatDate(selectedRequest.end_date) }}</span>
            </div>
          </div>

          <div class="form-group">
            <label for="declineReason" class="form-label">
              Reason for declining <span class="optional">(optional)</span>
            </label>
            <textarea 
              id="declineReason"
              v-model="declineReason"
              placeholder="Provide a reason for declining this leave request..."
              class="form-textarea"
              rows="4"
            ></textarea>
          </div>
        </div>

        <div class="modal-footer">
          <button @click="closeDeclineModal" class="btn-secondary">
            Cancel
          </button>
          <button 
            @click="confirmDecline"
            :disabled="processingDecline"
            class="btn-decline-confirm"
          >
            <span v-if="processingDecline">Declining...</span>
            <span v-else>Decline Request</span>
          </button>
        </div>
      </div>
    </div>

    <!-- Success/Error Messages -->
    <div v-if="successMessage" class="toast-message success">
      {{ successMessage }}
    </div>
    
    <div v-if="errorMessage" class="toast-message error">
      {{ errorMessage }}
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue';

// Mock store for standalone functionality
const store = {
  getUser: {
    id: 1,
    firstname: 'Admin',
    lastname: 'User',
    has_admin_access: true,
    is_representative: false
  }
};

// --- TYPES ---
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
  team_members?: string[]; 
}

// --- SAMPLE DATA ---
const sampleLeaveRequests: LeaveRequest[] = [
  {
    id: 1,
    employee_id: "EMP001",
    employee_name: "Sarah Johnson",
    leave_type: "Annual Leave",
    billing_type: "Paid",
    start_date: "2025-09-15",
    end_date: "2025-09-19",
    duration: 5,
    duration_type: "days",
    status: "Pending",
    reason: "Family vacation",
    team_members: ["1", "2"]
  },
  {
    id: 2,
    employee_id: "EMP002",  
    employee_name: "Mike Chen",
    leave_type: "Sick Leave",
    billing_type: "Paid",
    start_date: "2025-08-20",
    end_date: "2025-08-22",
    duration: 3,
    duration_type: "days",
    status: "Approved",
    approved_by: "Admin User",
    approved_at: "2025-08-19T10:30:00Z",
    reason: "Medical appointment"
  },
  {
    id: 3,
    employee_id: "EMP003",
    employee_name: "Emily Davis",  
    leave_type: "Personal Leave",
    billing_type: "Unpaid",
    start_date: "2025-09-01",
    end_date: "2025-09-01",
    duration: 4,
    duration_type: "hours",
    status: "Pending",
    reason: "Personal matters",
    team_members: ["1"]
  },
  {
    id: 4,
    employee_id: "EMP004",
    employee_name: "James Wilson",
    leave_type: "Annual Leave",
    billing_type: "Paid",  
    start_date: "2025-07-10",
    end_date: "2025-07-12",
    duration: 3,
    duration_type: "days",
    status: "Rejected",
    rejected_by: "Team Lead",
    rejected_at: "2025-07-05T14:20:00Z",
    reason: "Summer vacation"
  },
  {
    id: 5,
    employee_id: "EMP005",
    employee_name: "Lisa Thompson",
    leave_type: "Maternity Leave",
    billing_type: "Paid",
    start_date: "2025-10-01",  
    end_date: "2025-12-31",
    duration: 92,
    duration_type: "days",
    status: "Pending",
    reason: "Maternity leave",
    team_members: ["1", "2"]
  }
];

// --- REACTIVE STATE ---
const currentFilter = ref<'all' | 'upcoming' | 'past'>('all');
const processingRequests = ref<Set<number>>(new Set());
const successMessage = ref<string>('');
const errorMessage = ref<string>('');
const showDeclinePopup = ref<boolean>(false);
const selectedRequest = ref<LeaveRequest | null>(null);
const declineReason = ref<string>('');
const processingDecline = ref<boolean>(false);
const leaveRequests = ref<LeaveRequest[]>(sampleLeaveRequests);

// --- COMPUTED PROPERTIES ---
const isAdmin = computed(() => store.getUser.has_admin_access);
const isRepresentative = computed(() => store.getUser.is_representative || false);

const filteredRequests = computed(() => {
  const now = new Date();
  
  return leaveRequests.value.filter(request => {
    const startDate = new Date(request.start_date);
    const endDate = new Date(request.end_date);
    
    if (currentFilter.value === 'all') {
      return true;
    } else if (currentFilter.value === 'upcoming') {
      return startDate > now || (startDate <= now && endDate >= now);
    } else { // 'past'
      return endDate < now;
    }
  });
});

// --- HELPER FUNCTIONS ---
function getInitials(name: string): string {
  if (!name) return '';
  return name
    .split(' ')
    .map(part => part.charAt(0).toUpperCase())
    .join('')
    .substring(0, 2);
}

function getBillingTypeClass(billingType: string): string {
  switch (billingType) {
    case 'Paid': return 'badge-paid';
    case 'Unpaid': return 'badge-unpaid';
    default: return 'badge-neutral';
  }
}

function getStatusClass(status: string): string {
  switch (status) {
    case 'Approved': return 'badge-approved';
    case 'Rejected': return 'badge-rejected';
    case 'Pending': return 'badge-pending';
    default: return 'badge-neutral';
  }
}

function formatDate(dateString: string): string {
  return new Date(dateString).toLocaleDateString('en-US', {
    month: 'short',
    day: 'numeric',
    year: 'numeric'
  });
}

function getEmptyStateMessage(): string {
  switch (currentFilter.value) {
    case 'upcoming': return 'There are no upcoming leave requests to display.';
    case 'past': return 'There are no past leave requests to display.';
    default: return 'There are no leave requests to display.';
  }
}

function canApproveRequest(request: LeaveRequest): boolean {
  if (isAdmin.value) {
    return true;
  }
  if (isRepresentative.value && request.team_members) {
    return request.team_members.includes(store.getUser.id.toString());
  }
  return false;
}

function showMessage(message: string, type: 'success' | 'error') {
  if (type === 'success') {
    successMessage.value = message;
    errorMessage.value = '';
  } else {
    errorMessage.value = message;
    successMessage.value = '';
  }
  
  setTimeout(() => {
    successMessage.value = '';
    errorMessage.value = '';
  }, 5000);
}

// --- MODAL FUNCTIONS ---
function showDeclineModal(request: LeaveRequest) {
  selectedRequest.value = request;
  declineReason.value = '';
  showDeclinePopup.value = true;
}

function closeDeclineModal() {
  showDeclinePopup.value = false;
  selectedRequest.value = null;
  declineReason.value = '';
  processingDecline.value = false;
}

// --- API CALLS (MOCKED) ---
async function approveRequest(requestId: number) {
  processingRequests.value.add(requestId);
  try {
    // Simulate API call
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    // Update local state
    const request = leaveRequests.value.find(r => r.id === requestId);
    if (request) {
      request.status = 'Approved';
      request.approved_by = `${store.getUser.firstname} ${store.getUser.lastname}`;
      request.approved_at = new Date().toISOString();
    }
    
    showMessage('Leave request approved successfully.', 'success');
  } catch (error) {
    console.error('Error approving request:', error);
    showMessage('Failed to approve leave request. Please try again.', 'error');
  } finally {
    processingRequests.value.delete(requestId);
  }
}

async function confirmDecline() {
  if (!selectedRequest.value) return;
  
  const requestId = selectedRequest.value.id;
  processingDecline.value = true;
  processingRequests.value.add(requestId);
  
  try {
    // Simulate API call
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    // Update local state
    const request = leaveRequests.value.find(r => r.id === requestId);
    if (request) {
      request.status = 'Rejected';
      request.rejected_by = `${store.getUser.firstname} ${store.getUser.lastname}`;
      request.rejected_at = new Date().toISOString();
      if (declineReason.value.trim()) {
        request.reason = declineReason.value.trim();
      }
    }
    
    showMessage('Leave request declined successfully.', 'success');
    closeDeclineModal();
  } catch (error) {
    console.error('Error declining request:', error);
    showMessage('Failed to decline leave request. Please try again.', 'error');
  } finally {
    processingDecline.value = false;
    processingRequests.value.delete(requestId);
  }
}
</script>

<style scoped>
/* --- BASE & TYPOGRAPHY --- */
.leaves-management {
  font-family: 'DM Sans', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  background-color: #ffffff;
  color: #0a0a0a;
  line-height: 1.6;
  padding: 40px 60px 60px;
  max-width: 1400px;
  margin: 0 auto;
}

.page-title {
  font-family: 'DM Sans', sans-serif;
  font-weight: 700;
  font-size: 44px;
  line-height: 52px;
  color: #0a0a0a;
  letter-spacing: -0.88px;
  margin: 0;
}

/* --- HEADER --- */
.header-section {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 32px;
  flex-wrap: wrap;
  gap: 20px;
}

.header-controls {
  display: flex;
  align-items: center;
  gap: 24px;
}

.filter-buttons {
  display: flex;
  align-items: center;
  gap: 8px;
}

.btn-filter {
  padding: 10px 16px;
  border-radius: 8px;
  font-size: 14px;
  font-weight: 500;
  border: none;
  cursor: pointer;
  transition: all 0.2s ease;
  background-color: #f3f4f6;
  color: #6b7280;
}

.btn-filter:hover {
  background-color: #e5e7eb;
}

.btn-filter.active {
  background-color: #014c85;
  color: white;
}

/* --- CONTENT & TABLE --- */
.content-section {
  margin-top: 8px;
}

.desktop-table {
  display: block;
}

.table-container {
  background-color: #ffffff;
  border-radius: 8px;
  border: 1px solid #e8e8e8;
  overflow: hidden;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
}

.table-header {
  background-color: #f9fafb;
  border-bottom: 1px solid #e8e8e8;
  padding: 16px 24px;
}

.table-header-grid {
  display: grid;
  grid-template-columns: 2fr 1fr 1fr 1.5fr 1fr 1fr 1.5fr;
  gap: 20px;
  font-weight: 500;
  color: #6b7280;
  font-size: 12px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  font-family: 'Inter', sans-serif;
}

.table-body {
  background-color: white;
}

.table-row {
  border-bottom: 1px solid #e8e8e8;
  transition: background-color 0.2s ease;
}

.table-row:last-child {
  border-bottom: none;
}

.table-row:hover {
  background-color: #f9fafb;
}

.table-row-grid {
  display: grid;
  grid-template-columns: 2fr 1fr 1fr 1.5fr 1fr 1fr 1.5fr;
  gap: 20px;
  align-items: center;
  padding: 16px 24px;
}

/* --- TABLE CELL CONTENT --- */
.employee-info {
  display: flex;
  align-items: center;
  gap: 12px;
}

.employee-avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background-color: #e5e7eb;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 14px;
  font-weight: 600;
  color: #4b5563;
}

.employee-name {
  font-weight: 500;
  color: #111827;
  font-size: 14px;
}

.employee-id {
  color: #6b7280;
  font-size: 12px;
}

.cell-content, .duration, .date-range {
  font-size: 14px;
  color: #374151;
}

.date-to {
  color: #6b7280;
  font-size: 12px;
}

/* --- STATUS BADGES --- */
.status-badge {
  padding: 4px 12px;
  border-radius: 9999px;
  font-size: 12px;
  font-weight: 500;
  display: inline-block;
  text-transform: capitalize;
}
.badge-paid { background-color: #def7ec; color: #047857; }
.badge-unpaid { background-color: #fef3c7; color: #92400e; }
.badge-approved { background-color: #def7ec; color: #047857; }
.badge-rejected { background-color: #fee2e2; color: #991b1b; }
.badge-pending { background-color: #fef3c7; color: #92400e; }
.badge-neutral { background-color: #e5e7eb; color: #4b5563; }

/* --- ACTION BUTTONS & STATUS --- */
.action-buttons {
  display: flex;
  align-items: center;
  gap: 8px;
}

.btn-approve, .btn-decline {
  border: none;
  padding: 8px 12px;
  border-radius: 6px;
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
}
.btn-approve { background-color: #d1fae5; color: #065f46; }
.btn-approve:hover:not(:disabled) { background-color: #a7f3d0; }
.btn-decline { background-color: #fee2e2; color: #991b1b; }
.btn-decline:hover:not(:disabled) { background-color: #fecaca; }
.btn-approve:disabled, .btn-decline:disabled { opacity: 0.6; cursor: not-allowed; }

.action-status { font-size: 13px; font-weight: 500; }
.action-status.approved { color: #16a34a; }
.action-status.rejected { color: #dc2626; }
.action-status.disabled { color: #9ca3af; }

/* --- EMPTY STATE --- */
.empty-state, .mobile-empty-state {
  padding: 60px 20px;
  text-align: center;
  color: #6b7280;
}
.empty-state-icon { font-size: 48px; margin-bottom: 16px; }
.empty-state-title { font-size: 18px; font-weight: 500; margin-bottom: 8px; color: #374151; }
.empty-state-description { font-size: 14px; }

/* --- MOBILE CARDS --- */
.mobile-cards { display: none; }
.mobile-card {
  background-color: white;
  border: 1px solid #e8e8e8;
  border-radius: 12px;
  margin-bottom: 16px;
  box-shadow: 0 1px 2px rgba(0,0,0,0.05);
  overflow: hidden;
  transition: box-shadow 0.2s ease;
}
.mobile-card:hover { box-shadow: 0 4px 6px -1px rgba(0,0,0,0.1); }
.mobile-card-header { display: flex; align-items: center; justify-content: space-between; padding: 16px; border-bottom: 1px solid #f3f4f6; }
.mobile-card-body { padding: 16px; }
.info-row { display: flex; align-items: center; justify-content: space-between; margin-bottom: 12px; }
.info-row:last-child { margin-bottom: 0; }
.info-label { font-size: 14px; color: #6b7280; }
.info-value { font-size: 14px; color: #111827; font-weight: 500; text-align: right; }
.mobile-card-actions { padding: 8px 16px 16px; display: flex; gap: 8px; }
.mobile-card-actions .btn-approve.mobile, .mobile-card-actions .btn-decline.mobile { flex: 1; padding: 12px; font-size: 14px; }
.mobile-card-status { padding: 16px; border-top: 1px solid #f3f4f6; }

/* --- MODAL --- */
.modal-overlay { position: fixed; inset: 0; background-color: rgba(0, 0, 0, 0.5); display: flex; align-items: center; justify-content: center; z-index: 50; padding: 16px; }
.modal-content { background-color: white; border-radius: 12px; box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1); width: 100%; max-width: 480px; max-height: 90vh; overflow-y: auto; }
.modal-header { display: flex; align-items: center; justify-content: space-between; padding: 24px 24px 0; }
.modal-title { font-size: 18px; font-weight: 600; color: #111827; }
.modal-close { background: none; border: none; padding: 4px; cursor: pointer; color: #6b7280; border-radius: 50%; transition: all 0.2s ease; }
.modal-close:hover { background-color: #f3f4f6; color: #111827; }
.modal-body { padding: 24px; }
.request-summary { background-color: #f9fafb; border: 1px solid #e5e7eb; border-radius: 8px; padding: 16px; margin-bottom: 24px; }
.summary-item { display: flex; justify-content: space-between; margin-bottom: 8px; }
.summary-item:last-child { margin-bottom: 0; }
.summary-label { font-size: 14px; color: #6b7280; }
.summary-value { font-size: 14px; color: #111827; font-weight: 500; }
.form-label { display: block; font-size: 14px; font-weight: 500; color: #374151; margin-bottom: 8px; }
.optional { color: #9ca3af; font-weight: 400; }
.form-textarea { width: 100%; padding: 12px; border: 1px solid #d1d5db; border-radius: 6px; font-size: 14px; font-family: inherit; resize: vertical; transition: all 0.2s ease; }
.form-textarea:focus { outline: none; border-color: #014c85; box-shadow: 0 0 0 3px rgba(1, 76, 133, 0.1); }
.modal-footer { display: flex; justify-content: flex-end; gap: 12px; padding: 0 24px 24px; }
.btn-secondary { background-color: #f9fafb; border: 1px solid #d1d5db; color: #374151; padding: 10px 16px; border-radius: 6px; font-size: 14px; font-weight: 500; cursor: pointer; transition: all 0.2s ease; }
.btn-secondary:hover { background-color: #f3f4f6; }
.btn-decline-confirm { background-color: #dc2626; color: white; border: none; padding: 10px 16px; border-radius: 6px; font-size: 14px; font-weight: 500; cursor: pointer; transition: background-color 0.2s ease; }
.btn-decline-confirm:hover:not(:disabled) { background-color: #b91c1c; }
.btn-decline-confirm:disabled { opacity: 0.6; cursor: not-allowed; }

/* --- TOAST MESSAGES --- */
.toast-message { position: fixed; top: 20px; right: 20px; padding: 16px 20px; border-radius: 8px; box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1); z-index: 60; animation: slideInFromRight 0.3s ease-out; max-width: 400px; font-size: 14px; font-weight: 500; }
.toast-message.success { background-color: #f0fdf4; border: 1px solid #4ade80; color: #15803d; }
.toast-message.error { background-color: #fef2f2; border: 1px solid #f87171; color: #b91c1c; }
@keyframes slideInFromRight { from { transform: translateX(100%); opacity: 0; } to { transform: translateX(0); opacity: 1; } }

/* --- RESPONSIVE BREAKPOINTS --- */

/* Tablet (1200px) - Condensed Table */
@media (max-width: 1200px) {
  .leaves-management { padding: 32px 40px 40px; }
  .page-title { font-size: 36px; line-height: 42px; }
  
  .table-header-grid,
  .table-row-grid {
    grid-template-columns: 2fr 1fr 1.5fr 1fr 1.5fr;
    gap: 16px;
  }
  
  .hide-on-tablet {
    display: none;
  }
}

/* Mobile (768px) - Card View */
@media (max-width: 768px) {
  .leaves-management { padding: 20px 16px 40px; }
  .header-section { flex-direction: column; align-items: stretch; gap: 20px; margin-bottom: 24px; }
  .page-title { font-size: 28px; line-height: 32px; text-align: center; }
  .header-controls, .filter-buttons { width: 100%; justify-content: center; }
  .btn-filter { flex: 1; text-align: center; padding: 12px 8px; }
  
  .desktop-table { display: none; }
  .mobile-cards { display: block; }
  
  .modal-footer { flex-direction: column-reverse; gap: 8px; }
  .btn-secondary, .btn-decline-confirm { width: 100%; text-align: center; }
  .toast-message { right: 16px; left: 16px; top: 16px; max-width: none; }
}

/* Extra Small Mobile (480px) - Compact View */
@media (max-width: 480px) {
  .leaves-management { padding: 16px 12px 32px; }
  .page-title { font-size: 24px; line-height: 28px; }
  .filter-buttons { gap: 4px; }
  .btn-filter { font-size: 12px; padding: 10px 6px; }
  
  .employee-avatar { width: 36px; height: 36px; font-size: 12px; }
  .employee-name { font-size: 13px; }
  .employee-id { font-size: 11px; }
  .info-label, .info-value { font-size: 13px; }
}

/* --- ACCESSIBILITY & FOCUS --- */
.btn-filter:focus-visible,
.btn-approve:focus-visible,
.btn-decline:focus-visible,
.btn-secondary:focus-visible,
.btn-decline-confirm:focus-visible,
.modal-close:focus-visible {
  outline: 2px solid #014c85;
  outline-offset: 2px;
}
.form-textarea:focus {
  outline: none;
  border-color: #014c85;
  box-shadow: 0 0 0 3px rgba(1, 76, 133, 0.1);
}
</style>
