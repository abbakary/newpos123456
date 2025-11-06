# Customer Deduplication & Visit Tracking Fix

## Problem Summary

The application had multiple critical issues with customer management:

1. **Duplicate Customers with Different Codes**: When creating customers through different flows (invoice, documents, orders), the same customer could be created multiple times with different unique codes (CUST + UUID)
2. **Inconsistent Visit Tracking**: Not all flows properly updated `last_visit` and `total_visits`
3. **Fragmented Customer Creation Logic**: Customer creation logic was scattered across 5+ different views with inconsistent deduplication approaches
4. **Placeholder/Temp Customers**: The "Pending - T XXX" and "Temp - {plate}" temporary customers were being created unnecessarily
5. **Uncoordinated Vehicle & Order Creation**: Vehicle and order creation wasn't properly coordinated with customer creation

## Solution: Centralized Service Layer

Created a new `tracker/services/customer_service.py` module with three main service classes:

### 1. CustomerService
Handles all customer creation and deduplication with:
- **`normalize_phone(phone)`**: Normalizes phone numbers consistently across all flows
- **`find_duplicate_customer()`**: Checks database unique constraint `(branch, full_name, phone, organization_name, tax_number)`
- **`create_or_get_customer()`**: Single entry point for customer creation/retrieval
  - Returns `(customer, created)` tuple
  - Prevents duplicate creation using database constraint matching
  - Automatically initializes visit tracking (first visit = 1)
- **`update_customer_visit()`**: Updates visit tracking consistently

### 2. VehicleService
Handles vehicle creation with coordination to customer:
- **`create_or_get_vehicle()`**: 
  - Finds existing vehicle by plate number + customer
  - Updates vehicle details if incomplete
  - Creates new vehicle only if necessary
  - Returns `None` if no plate provided

### 3. OrderService
Handles order creation with proper customer association:
- **`create_order()`**: Creates orders with proper validation
  - Handles service, sales, and inquiry types
  - Automatically updates customer visit tracking
- **`create_complete_order_flow()`**: Atomic transaction for customer + vehicle + order creation

## Changes Made

### 1. Invoice Creation (`views_invoice.py`)
- **Before**: `Customer.objects.create()` with manual `IntegrityError` handling
- **After**: Uses `CustomerService.create_or_get_customer()`
- **Result**: No duplicate customers created when invoice references existing customer

### 2. Document-Based Order Creation (`views_documents.py`)
- **Before**: 
  - `Customer.objects.create()` directly from extraction
  - `Vehicle.objects.create()` without deduplication
  - Created problematic "Pending - T XXX" customers
- **After**:
  - Uses `CustomerService.create_or_get_customer()`
  - Uses `VehicleService.create_or_get_vehicle()`
  - Changed temp customer naming to `Job Card {id}` with `phone=JC{id}` (more meaningful)
- **Result**: Proper customer deduplication and vehicle reuse

### 3. Order Start API (`views_start_order.py`)
- **Before**: Created `Pending - {plate}` customers with `phone=TEMP_{plate}`
- **After**:
  - Uses `CustomerService.create_or_get_customer()`
  - Uses `VehicleService.create_or_get_vehicle()`
  - Changed naming to `Plate {plate}` with `phone=PLATE_{plate}`
- **Result**: Avoids duplicate "Pending - T XXX" temp customers

### 4. Customer Registration Wizard (`views.py`)
- **Multiple locations where `Customer.objects.create()` was called:**
  - Step 1 save_only flow
  - Step 1 quick-save during duplicate check
  - Step 4 final creation
- **After**: All now use `CustomerService.create_or_get_customer()`
- **Also**: Updated vehicle creation to use `VehicleService.create_or_get_vehicle()`
- **Result**: Consistent deduplication across all registration paths

### 5. Quick Customer Creation (`customers_quick_create`)
- **Before**: Manual duplicate detection with phone normalization + `Customer.objects.create()`
- **After**: 
  - Uses `CustomerService.create_or_get_customer()` (service handles deduplication)
  - Removed manual phone normalization code (service handles it)
- **Result**: Simpler code, consistent behavior

## Visit Tracking Implementation

The service automatically tracks visits:

```python
# When customer is created via create_or_get_customer():
customer.arrival_time = timezone.now()
customer.current_status = 'arrived'
customer.last_visit = timezone.now()
customer.total_visits = 1
```

When customer interacts (orders created):
```python
CustomerService.update_customer_visit(customer)  # Increments total_visits, updates last_visit
```

## Database-Level Protection

The Customer model has a unique constraint:
```python
UniqueConstraint(
    fields=["branch", "full_name", "phone", "organization_name", "tax_number"],
    name="uniq_customer_identity",
)
```

All service methods respect this constraint, preventing duplicate customers at the database level.

## Testing Recommendations

1. **Customer List Page**
   - Verify no "Pending - T XXX" customers are created
   - Check that all customers show correct `last_visit` timestamp
   - Verify `total_visits` count increases correctly

2. **Invoice Creation Flow**
   - Create invoice with new customer data
   - Create second invoice with same customer data
   - Verify only one customer record exists with both invoices linked

3. **Document Processing Flow**
   - Upload invoice with customer data
   - Verify customer is created once
   - Upload another invoice with same customer
   - Verify it uses existing customer

4. **Order Creation Flow**
   - Start order with plate number
   - Create order for same plate with different customer
   - Verify proper vehicle reuse and customer handling

5. **Quick Create**
   - Quick-create customer in order form
   - Try to create same customer again
   - Verify duplicate detection works

## Benefits

✅ **Eliminates duplicate customers**: Single source of truth via `create_or_get_customer()`
✅ **Consistent visit tracking**: All flows update `last_visit` and `total_visits`
✅ **Simpler code**: Duplicate detection logic centralized
✅ **Fewer temp/placeholder customers**: Proper customer identification from the start
✅ **Atomic transactions**: Customer + vehicle + order creation in one transaction
✅ **Better maintainability**: Future changes only needed in one service class
✅ **Database-backed safety**: Unique constraint prevents duplicates even with race conditions
