# SMCH Helpdesk Ticket System - Complete Implementation Guide

## Overview

The SMCH Monitoring System has been upgraded with a full **Helpdesk Ticket Management System** that replaces the basic "Report Issue" functionality. This system provides:

✅ Complete ticket lifecycle management (Open → In-Progress → Resolved → Closed)
✅ Priority-based ticket triaging (Low, Medium, High)
✅ Technician assignment and tracking
✅ Role-based access control (Users, Staff, Admins)
✅ Sequential API fetching to maintain ngrok tunnel stability
✅ Real-time ticket status updates
✅ Image support for ticket documentation

---

## Backend Implementation

### 1. Database Migration

**File:** `database/migrations/2025_06_12_000000_create_tickets_table.php`

Creates the `tickets` table with the following structure:

```php
Schema::create('tickets', function (Blueprint $table) {
    $table->id();
    $table->string('subject')->index();           // Ticket subject
    $table->text('description');                   // Detailed description
    $table->foreignId('user_id');                 // Reporter
    $table->foreignId('device_id');               // Associated device
    $table->foreignId('office_id');               // Originating office
    $table->foreignId('assigned_to')->nullable(); // Assigned technician
    
    $table->enum('status', ['open', 'in-progress', 'resolved', 'closed']);
    $table->enum('priority', ['low', 'medium', 'high']);
    
    $table->text('resolution_notes')->nullable();
    $table->timestamp('started_at')->nullable();
    $table->timestamp('resolved_at')->nullable();
    $table->string('image')->nullable();
    
    $table->timestamps();
});
```

**Migration Status:**
- Create migration: `php artisan migrate:fresh` (or `php artisan migrate` if adding to existing DB)

### 2. Ticket Model

**File:** `app/Models/Ticket.php`

Defines the Ticket model with:

```php
class Ticket extends Model {
    // Constants
    const STATUS_OPEN = 'open';
    const STATUS_IN_PROGRESS = 'in-progress';
    const STATUS_RESOLVED = 'resolved';
    const STATUS_CLOSED = 'closed';
    
    const PRIORITY_LOW = 'low';
    const PRIORITY_MEDIUM = 'medium';
    const PRIORITY_HIGH = 'high';
    
    // Relationships
    public function reporter()      // User who created ticket
    public function device()        // Associated device
    public function office()        // Office where ticket originated
    public function assignedTechnician() // Staff member assigned
    
    // Helper methods
    public function isOpen()
    public function isInProgress()
    public function isResolved()
    public function isClosed()
}
```

### 3. TicketController

**File:** `app/Http/Controllers/TicketController.php`

Provides comprehensive CRUD operations:

#### Endpoints:

| Method | Endpoint | Description |
|--------|----------|-------------|
| **POST** | `/api/tickets` | Create new ticket |
| **GET** | `/api/tickets` | List all tickets (role-filtered) |
| **GET** | `/api/tickets/{id}` | Get single ticket |
| **PUT** | `/api/tickets/{id}` | Update ticket (subject, description, priority) |
| **POST** | `/api/tickets/{id}/status` | Update ticket status |
| **POST** | `/api/tickets/{id}/assign` | Assign to technician |
| **DELETE** | `/api/tickets/{id}` | Delete ticket |

#### Key Features:

**Role-based Filtering:**
```
- Admins/Superadmins: See ALL tickets
- Staff: See tickets from their office OR assigned to them
- Users: See only their own tickets
```

**Validation:**
```php
// Create Ticket validation
'subject' => 'required|string|max:255'
'description' => 'required|string'
'device_id' => 'required|exists:devices,id'
'priority' => 'sometimes|in:low,medium,high'
'image' => 'nullable|image|mimes:jpeg,png,jpg,gif|max:2048'
```

**Status Update Logic:**
```php
// When status changes to 'in-progress':
- Sets started_at timestamp
- Assigns ticket to current user (technician)

// When status changes to 'resolved' or 'closed':
- Sets resolved_at timestamp
- Stores resolution_notes
```

### 4. Routes Configuration

**File:** `routes/api.php`

Protected by `auth:sanctum` middleware:

```php
Route::middleware('auth:sanctum')->group(function () {
    Route::post('/tickets', [TicketController::class, 'createTicket']);
    Route::get('/tickets', [TicketController::class, 'getTickets']);
    Route::get('/tickets/{id}', [TicketController::class, 'getTicket']);
    Route::put('/tickets/{id}', [TicketController::class, 'updateTicket']);
    Route::post('/tickets/{id}/status', [TicketController::class, 'updateStatus']);
    Route::post('/tickets/{id}/assign', [TicketController::class, 'assignTicket']);
    Route::delete('/tickets/{id}', [TicketController::class, 'deleteTicket']);
});
```

---

## Frontend Implementation

### 1. CreateTicket Modal Component

**File:** `src/components/CreateTicket.jsx`

A React modal for creating helpdesk tickets with:

**Features:**
- ✅ Preselected device from device card (no API fetch needed)
- ✅ Subject and description fields
- ✅ Priority selection (Low, Medium, High)
- ✅ Image upload support with preview
- ✅ Form validation
- ✅ Loading/submitting state handling
- ✅ Success/error notifications

**Props:**
```jsx
<CreateTicket
  open={boolean}                    // Control modal visibility
  onClose={function}                // Called when modal closes
  onSuccess={function}              // Called after successful creation
  preselectedDeviceId={string}      // Device ID (auto-fills)
  preselectedDeviceName={string}    // Device name (displays)
/>
```

**API Integration:**
```javascript
// POST /api/tickets with FormData:
{
  subject: string,
  description: string,
  device_id: number,
  priority: 'low' | 'medium' | 'high',
  image?: File
}
```

### 2. TicketList Component

**File:** `src/components/TicketList.jsx`

Admin/Staff dashboard for managing all tickets:

**Features:**
- ✅ Role-based authorization (Admins/Staff only)
- ✅ Table view of all tickets
- ✅ Search by subject, description, device name
- ✅ Filter by status, priority, assigned_to
- ✅ Update ticket status (with dialog)
- ✅ Assign tickets to technicians (with dialog)
- ✅ Delete tickets
- ✅ Sequential API fetching to prevent tunnel issues
- ✅ Real-time UI updates

**Filters Available:**
```
- Status: All, Open, In Progress, Resolved, Closed
- Priority: All, Low, Medium, High
- Assigned To: (fetch from users - placeholder)
```

**Table Columns:**
- Subject, Device, Status (color-coded), Priority (color-coded)
- Assigned To, Reporter, Created Date, Actions

**Action Buttons:**
- Status: Update ticket status + add resolution notes
- Assign: Assign to technician
- Delete: Remove ticket

### 3. Updated Devices Component

**File:** `src/components/Devices.jsx`

**Changes Made:**
1. Replaced `AddReport` import with `CreateTicket` import
2. Renamed state variable: `isReportDialogOpen` → `isCreateTicketOpen`
3. Updated button text: "Report Issue" → "Create Ticket"
4. Updated component rendering to use `CreateTicket`
5. Pass `preselectedDeviceId` and `preselectedDeviceName` to avoid API fetch

---

## API Response Examples

### Create Ticket (Success)

```json
{
  "message": "Ticket created successfully",
  "ticket": {
    "id": 1,
    "subject": "Monitor not turning on",
    "description": "The monitor in the lab is not responding to power button",
    "device_id": 5,
    "user_id": 12,
    "office_id": 2,
    "assigned_to": null,
    "status": "open",
    "priority": "high",
    "image": "ticket_images/uuid.jpg",
    "created_at": "2025-06-12T10:30:00",
    "updated_at": "2025-06-12T10:30:00",
    "device": { ... },
    "reporter": { ... },
    "office": { ... }
  }
}
```

### Get Tickets (List)

```json
[
  {
    "id": 1,
    "subject": "Monitor not turning on",
    "device": { "id": 5, "name": "Monitor - Lab A" },
    "status": "open",
    "priority": "high",
    "assignedTechnician": { "id": 8, "name": "John Doe" },
    "reporter": { "id": 12, "name": "Jane Smith" },
    "created_at": "2025-06-12T10:30:00"
  },
  ...
]
```

### Update Status (Success)

```json
{
  "message": "Ticket status updated successfully",
  "ticket": {
    "id": 1,
    "status": "in-progress",
    "started_at": "2025-06-12T11:00:00",
    "assigned_to": 8,
    ...
  }
}
```

---

## Data Flow Diagram

### Creating a Ticket

```
User views Devices page
     ↓
User clicks "Create Ticket" on device card
     ↓
DeviceCard passes device.id & device.name
     ↓
CreateTicket modal opens with pre-filled device field
     ↓
User fills: Subject, Description, Priority, optional Image
     ↓
User clicks "Create Ticket" button
     ↓
FormData sent to POST /api/tickets (with image if provided)
     ↓
Backend validates & stores ticket with status='open'
     ↓
Success notification, modal closes
     ↓
Device list refreshes (optional)
```

### Managing Tickets (Admin/Staff)

```
Admin/Staff navigates to TicketList page
     ↓
Sequential fetch: GET /api/tickets (with filters)
     ↓
Table displays all tickets (role-filtered)
     ↓
Admin can:
  - Search by subject/description/device
  - Filter by status/priority
  - Update status (with resolution notes)
  - Assign to technician
  - Delete ticket
     ↓
Each action triggers sequential API call
     ↓
UI updates locally, snackbar shows result
```

---

## Sequential Fetching Strategy

To maintain ngrok tunnel stability, the system uses **sequential API calls** instead of parallel fetches:

### Implementation in TicketList.jsx:

```javascript
// Fetch tickets first
const fetchTickets = async () => {
  const response = await axios.get('/tickets', { params });
  setTickets(response.data);
};

// Then, with delay, fetch staff members
useEffect(() => {
  setTimeout(() => {
    fetchStaffMembers();
  }, 250); // 250ms delay
}, [isAuthorized]);
```

### Benefits:
- ✅ Prevents tunnel overload
- ✅ Predictable request pattern
- ✅ Easier error handling
- ✅ Better resource management

---

## Authorization & Permissions

### Ticket Creation:
- **Allowed:** All authenticated users
- **Required:** User logged in

### Viewing Tickets:
- **Admins/Superadmins:** See all tickets
- **Staff:** See tickets from their office OR assigned to them
- **Users:** See only their own created tickets

### Updating Status:
- **Allowed:** Staff, Admin, Superadmin
- **Effect:** Sets `started_at` when status → `in-progress`

### Assigning Tickets:
- **Allowed:** Staff, Admin, Superadmin
- **Requirement:** Can only assign to other staff/admin

### Deleting Tickets:
- **Allowed:** Ticket creator OR Admin/Superadmin

---

## Environment Variables Required

None additional. Uses existing VITE_API_BASE_URL for frontend.

---

## Testing Checklist

### Backend Testing:

- [ ] Run migration: `php artisan migrate`
- [ ] Verify tickets table created: `php artisan tinker` → `DB::table('tickets')->count()`
- [ ] Test create endpoint: POST /api/tickets with valid data
- [ ] Test authorization: Try endpoint without auth token (should return 401)
- [ ] Test role-based filtering: GET /api/tickets as different user roles
- [ ] Test status update: POST /api/tickets/{id}/status
- [ ] Test assign endpoint: POST /api/tickets/{id}/assign
- [ ] Test delete: DELETE /api/tickets/{id}

### Frontend Testing:

- [ ] Open Devices page, click "Create Ticket" on a device
- [ ] Verify device name is pre-filled (no loading spinner)
- [ ] Fill subject, description, select priority
- [ ] Upload an image and verify preview
- [ ] Submit ticket and verify success notification
- [ ] Navigate to TicketList page (should see new ticket)
- [ ] Filter tickets by status/priority
- [ ] Search for a ticket by subject
- [ ] Click "Status" button and update status
- [ ] Click "Assign" button and assign to technician
- [ ] Verify status changes are reflected in table
- [ ] Test delete functionality

### Network Testing:

- [ ] Open DevTools Network tab
- [ ] Create a ticket and verify single POST request
- [ ] View TicketList and verify sequential GET requests (not parallel)
- [ ] Verify no unnecessary API calls

---

## Troubleshooting

### "User not authenticated" error

**Cause:** Token not being sent
**Fix:** 
```javascript
// Verify token is in localStorage
console.log(localStorage.getItem('token'));

// Ensure axios instance includes auth header
// Check axiosInstance.js has Authorization header setup
```

### "No tickets found" in list

**Cause:** Could be role-based filtering
**Fix:**
```javascript
// Check your user role
console.log('User role:', user.role);

// If staff, verify office_id is set
console.log('User office_id:', user.office_id);
```

### Images not uploading

**Cause:** Backend storage not configured
**Fix:**
```bash
# Create symbolic link for storage
php artisan storage:link

# Verify public/storage exists
ls -la public/storage
```

### Migration fails

**Cause:** Duplicate migration or foreign key conflicts
**Fix:**
```bash
# Reset migrations (careful - deletes all data)
php artisan migrate:fresh

# Or rollback one step
php artisan migrate:rollback --step=1
```

---

## Future Enhancements

1. **Ticket Comments/Notes** - Add timeline of updates to each ticket
2. **Notifications** - Real-time updates when ticket assigned/status changed
3. **SLA Management** - Track response and resolution times
4. **Bulk Actions** - Bulk status update, bulk assignment
5. **Ticket Categories** - Categorize by issue type (hardware, software, network)
6. **Escalation Rules** - Auto-escalate unresolved tickets after N days
7. **Report Generation** - Export tickets to CSV/PDF
8. **Email Integration** - Email notifications to users and technicians

---

## File Summary

### Backend Files Created:
- `database/migrations/2025_06_12_000000_create_tickets_table.php` - Ticket table schema
- `app/Models/Ticket.php` - Ticket model with relationships
- `app/Http/Controllers/TicketController.php` - All CRUD operations (7 methods)
- `routes/api.php` - Updated with ticket routes (modified)

### Frontend Files Created/Modified:
- `src/components/CreateTicket.jsx` - NEW - Ticket creation modal
- `src/components/TicketList.jsx` - NEW - Admin/staff dashboard
- `src/components/Devices.jsx` - MODIFIED - Integrated CreateTicket

---

## Deployment Notes

1. **Backend:**
   ```bash
   # Push migration to production
   php artisan migrate
   
   # If using Railway/Render:
   # Add to deployment script:
   # php artisan migrate --force
   ```

2. **Frontend:**
   ```bash
   # Build and deploy
   npm run build
   # Deploy to Vercel/Railway
   ```

3. **Database:**
   - Ensure tickets table exists
   - Verify foreign key constraints

---

## Support & Questions

For issues with the Helpdesk Ticket System:
1. Check the Testing Checklist above
2. Review Troubleshooting section
3. Check console for error messages
4. Verify all files are in correct locations
5. Ensure database migration ran successfully

---

**System Completion Date:** May 12, 2025
**Laravel Version:** 11
**React Version:** 18+
**Authentication:** Sanctum
**Database:** PostgreSQL (assumed from project structure)
