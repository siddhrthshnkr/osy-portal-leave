<template>
  <Head title="Leaves Management" />
  
  <!-- @vue-ignore -->
  <AuthenticatedLayout>
    <div class="flex justify-between items-center">
      <h1 class="text-4xl font-bold text-[#0a0a0a] tracking-[-0.88px] leading-[52px]">Leaves</h1>
      
      <div class="flex items-center gap-6">
        <!-- Filter Buttons -->
        <div class="flex items-center gap-2">
          <button 
            @click="currentFilter = 'upcoming'"
            :class="currentFilter === 'upcoming' 
              ? 'bg-[#014c85] text-white' 
              : 'bg-gray-100 text-gray-600 hover:bg-gray-200'"
            class="px-4 py-2 rounded-lg font-medium text-sm transition-colors"
          >
            Upcoming
          </button>
          <button 
            @click="currentFilter = 'past'"
            :class="currentFilter === 'past' 
              ? 'bg-[#014c85] text-white' 
              : 'bg-gray-100 text-gray-600 hover:bg-gray-200'"
            class="px-4 py-2 rounded-lg font-medium text-sm transition-colors"
          >
            Past
          </button>
        </div>
      </div>
    </div>

    <PageHeader>
      <div class="mt-8">
        <div class="bg-white rounded-lg border border-[#e8e8e8] shadow-sm">
          <!-- Table Header -->
          <div class="bg-[#f9f9f9] border-b-2 border-[#d0d0d0] p-4 rounded-t-lg">
            <div class="grid grid-cols-7 gap-4 font-medium text-[#969696] text-sm">
              <div>Team Member</div>
              <div>Leave Type</div>
              <div>Billing Type</div>
              <div>Leave Period</div>
              <div>Days/Hours Taken</div>
              <div>Status</div>
              <div>Action</div>
            </div>
          </div>

          <!-- Table Body -->
          <div class="divide-y divide-[#d0d0d0]">
            <div 
              v-for="request in filteredRequests" 
              :key="request.id"
              class="p-4 hover:bg-gray-50 transition-colors"
            >
              <div class="grid grid-cols-7 gap-4 items-center">
                <!-- Team Member -->
                <div class="flex items-center gap-3">
                  <div class="w-10 h-10 rounded-full bg-gray-200 flex items-center justify-center text-sm font-semibold text-gray-600">
                    {{ getInitials(request.employee_name) }}
                  </div>
                  <div>
                    <div class="font-medium text-[#0a0a0a]">{{ request.employee_name }}</div>
                    <div class="text-xs text-gray-500">{{ request.employee_id }}</div>
                  </div>
                </div>

                <!-- Leave Type -->
                <div class="text-sm text-[#0a0a0a]">{{ request.leave_type }}</div>

                <!-- Billing Type -->
                <div>
                  <span :class="getBillingTypeClass(request.billing_type)" class="px-3 py-1 rounded-full text-xs font-medium">
                    {{ request.billing_type }}
                  </span>
                </div>

                <!-- Leave Period -->
                <div class="text-sm text-[#0a0a0a]">
                  <div>{{ formatDate(request.start_date) }}</div>
                  <div class="text-xs text-gray-500">to {{ formatDate(request.end_date) }}</div>
                </div>

                <!-- Days/Hours Taken -->
                <div class="text-sm text-[#0a0a0a] font-medium">
                  {{ request.duration_type === 'days' ? `${request.duration} days` : `${request.duration} hours` }}
                </div>

                <!-- Status -->
                <div>
                  <span :class="getStatusClass(request.status)" class="px-3 py-1 rounded-full text-xs font-medium">
                    {{ request.status }}
                  </span>
                </div>

                <!-- Action -->
                <div class="flex items-center gap-2">
                  <template v-if="request.status === 'Pending' && canApproveRequest(request)">
                    <button 
                      @click="approveRequest(request.id)"
                      :disabled="processingRequests.has(request.id)"
                      class="bg-[#d4edda] text-[#155724] px-3 py-1 rounded-lg text-xs font-medium hover:bg-[#c3e6cb] transition-colors disabled:opacity-50 disabled:cursor-not-allowed"
                    >
                      <span v-if="processingRequests.has(request.id)">Processing...</span>
                      <span v-else>Approve</span>
                    </button>
                    <button 
                      @click="rejectRequest(request.id)"
                      :disabled="processingRequests.has(request.id)"
                      class="bg-[#f8d7da] text-[#721c24] px-3 py-1 rounded-lg text-xs font-medium hover:bg-[#f1b0b7] transition-colors disabled:opacity-50 disabled:cursor-not-allowed"
                    >
                      <span v-if="processingRequests.has(request.id)">Processing...</span>
                      <span v-else>Reject</span>
                    </button>
                  </template>
                  <template v-else-if="request.status === 'Approved'">
                    <div class="text-xs text-green-600">
                      ✓ Approved by {{ request.approved_by }}
                    </div>
                  </template>
                  <template v-else-if="request.status === 'Rejected'">
                    <div class="text-xs text-red-600">
                      ✗ Rejected by {{ request.rejected_by }}
                    </div>
                  </template>
                  <template v-else>
                    <span class="text-xs text-gray-400">No actions available</span>
                  </template>
                </div>
              </div>
            </div>
          </div>

          <!-- Empty State -->
          <div v-if="filteredRequests.length === 0" class="p-12 text-center text-gray-500">
            <div class="text-lg font-medium mb-2">No {{ currentFilter }} leave requests</div>
            <div class="text-sm">{{ currentFilter === 'upcoming' ? 'There are no upcoming leave requests to display.' : 'There are no past leave requests to display.' }}</div>
          </div>
        </div>
      </div>
    </PageHeader>

    <!-- Success/Error Messages -->
    <div v-if="successMessage" class="fixed top-4 right-4 bg-green-100 border border-green-400 text-green-700 px-4 py-3 rounded-lg shadow-lg z-50">
      {{ successMessage }}
    </div>
    
    <div v-if="errorMessage" class="fixed top-4 right-4 bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded-lg shadow-lg z-50">
      {{ errorMessage }}
    </div>
  </AuthenticatedLayout>
</template>

<script setup lang="ts">
import AuthenticatedLayout from '@/layouts/AuthenticatedLayout.vue';
import PageHeader from '@/shared/PageHeader.vue';
import { useGlobalStore } from '@/stores/globalStore';
import { Head } from '@inertiajs/inertia-vue3';
import axios from 'axios';
import dayjs from 'dayjs';
import { PropType, computed, defineProps, ref } from 'vue';

const store = useGlobalStore();

axios.defaults.withCredentials = true;

// Types
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
  team_members?: string[]; // For representatives to check if they manage this employee
}

const props = defineProps({
  leave_requests: {
    type: Array as PropType<LeaveRequest[]>,
    required: true
  },
  user: {
    type: Object,
    required: true
  }
});

// Reactive state
const currentFilter = ref<'upcoming' | 'past'>('upcoming');
const processingRequests = ref<Set<number>>(new Set());
const successMessage = ref<string>('');
const errorMessage = ref<string>('');

// Computed properties
const isAdmin = computed(() => store.getUser.has_admin_access);
const isRepresentative = computed(() => store.getUser.is_representative || false);

const filteredRequests = computed(() => {
  const now = dayjs();
  
  return props.leave_requests.filter(request => {
    const startDate = dayjs(request.start_date);
    const endDate = dayjs(request.end_date);
    
    if (currentFilter.value === 'upcoming') {
      return startDate.isAfter(now) || (startDate.isBefore(now) && endDate.isAfter(now));
    } else {
      return endDate.isBefore(now);
    }
  });
});

// Helper functions
function getInitials(name: string): string {
  return name
    .split(' ')
    .map(part => part.charAt(0).toUpperCase())
    .join('')
    .substring(0, 2);
}

function getBillingTypeClass(billingType: string): string {
  switch (billingType) {
    case 'Paid':
      return 'bg-[#d4edda] text-[#155724]';
    case 'Unpaid':
      return 'bg-[#fff3cd] text-[#856404]';
    default:
      return 'bg-[#e2e3e5] text-[#6c757d]';
  }
}

function getStatusClass(status: string): string {
  switch (status) {
    case 'Approved':
      return 'bg-[#d4edda] text-[#155724]';
    case 'Rejected':
      return 'bg-[#f8d7da] text-[#721c24]';
    case 'Pending':
      return 'bg-[#fff3cd] text-[#856404]';
    default:
      return 'bg-[#e2e3e5] text-[#6c757d]';
  }
}

function formatDate(dateString: string): string {
  return dayjs(dateString).format('MMM DD, YYYY');
}

function canApproveRequest(request: LeaveRequest): boolean {
  // Admins can approve all requests
  if (isAdmin.value) {
    return true;
  }
  
  // Representatives can only approve requests from their assigned team members
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

// API calls
async function approveRequest(requestId: number) {
  try {
    processingRequests.value.add(requestId);
    
    const response = await axios.post(`/api/leave-requests/${requestId}/approve`, {
      approver_id: store.getUser.id,
      approved_at: new Date().toISOString()
    });
    
    if (response.status === 200) {
      // Update the request in the local state
      const request = props.leave_requests.find(r => r.id === requestId);
      if (request) {
        request.status = 'Approved';
        request.approved_by = `${store.getUser.firstname} ${store.getUser.lastname}`;
        request.approved_at = new Date().toISOString();
      }
      
      showMessage('Leave request approved successfully. Zoho People updated and operations team notified.', 'success');
    }
  } catch (error) {
    console.error('Error approving request:', error);
    showMessage('Failed to approve leave request. Please try again.', 'error');
  } finally {
    processingRequests.value.delete(requestId);
  }
}

async function rejectRequest(requestId: number) {
  try {
    processingRequests.value.add(requestId);
    
    const response = await axios.post(`/api/leave-requests/${requestId}/reject`, {
      rejector_id: store.getUser.id,
      rejected_at: new Date().toISOString()
    });
    
    if (response.status === 200) {
      // Update the request in the local state
      const request = props.leave_requests.find(r => r.id === requestId);
      if (request) {
        request.status = 'Rejected';
        request.rejected_by = `${store.getUser.firstname} ${store.getUser.lastname}`;
        request.rejected_at = new Date().toISOString();
      }
      
      showMessage('Leave request rejected successfully. Zoho People updated and operations team notified.', 'success');
    }
  } catch (error) {
    console.error('Error rejecting request:', error);
    showMessage('Failed to reject leave request. Please try again.', 'error');
  } finally {
    processingRequests.value.delete(requestId);
  }
}
</script>

<style scoped>
/* Custom styles following the attendance dashboard design */
.container {
  padding: 65px 88px 88px 88px;
  max-width: 1400px;
  margin: 0 auto;
}

/* Mobile responsive */
@media (max-width: 1200px) {
  .container {
    padding: 40px 20px 60px 20px;
  }
  
  .grid-cols-7 {
    grid-template-columns: repeat(4, 1fr);
    gap: 2rem;
  }
  
  .grid-cols-7 > div:nth-child(n+5) {
    display: none;
  }
}

@media (max-width: 768px) {
  .container {
    padding: 20px 16px 40px 16px;
  }
  
  .grid-cols-7 {
    grid-template-columns: 1fr;
    gap: 1rem;
  }
  
  .grid-cols-7 > div {
    display: block !important;
    border-bottom: 1px solid #e5e7eb;
    padding-bottom: 0.5rem;
  }
  
  .grid-cols-7 > div:last-child {
    border-bottom: none;
  }
}

/* Smooth transitions */
.transition-colors {
  transition: background-color 0.2s ease, color 0.2s ease;
}

/* Loading states */
.disabled\:opacity-50:disabled {
  opacity: 0.5;
}

.disabled\:cursor-not-allowed:disabled {
  cursor: not-allowed;
}

/* Message animations */
.fixed {
  animation: slideInFromRight 0.3s ease-out;
}

@keyframes slideInFromRight {
  from {
    transform: translateX(100%);
    opacity: 0;
  }
  to {
    transform: translateX(0);
    opacity: 1;
  }
}
</style>