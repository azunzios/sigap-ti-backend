# Kartu Kendali - Bug Fix & Testing Guide

## 🐛 Bug yang Diperbaiki

### **Problem: Pagination Sebelum Filter**

**Sebelum fix:**
```php
// ❌ BAD: Pagination dilakukan SEBELUM filter work order
$tickets = $query->paginate($perPage); // Ambil 15 tiket

$data = $tickets->map(function ($ticket) {
    if ($ticket->workOrders->isEmpty()) {
        return null; // Skip tiket tanpa work order
    }
    // ...
})->filter();
```

**Hasil bug:**
- Backend ambil 15 tiket perbaikan per page
- Dari 15 tiket, mungkin hanya 3 yang punya work order completed
- 12 tiket lainnya di-skip
- **Frontend cuma dapat 3 data per page** ❌

### **Solusi: Filter SEBELUM Pagination**

**Setelah fix:**
```php
// ✅ GOOD: Filter work order SEBELUM pagination
$query = Ticket::where('type', 'perbaikan')
    ->whereHas('workOrders', function ($q) {
        $q->where('status', 'completed');
    })
    ->with(['workOrders' => function ($q) {
        $q->where('status', 'completed');
    }]);

$tickets = $query->paginate($perPage); // Ambil 15 tiket yang SUDAH terfilter
```

**Hasil fix:**
- Backend filter dulu tiket yang punya work order completed
- Pagination 15 item dilakukan setelah filter
- **Frontend dapat 15 data penuh per page** ✅

---

## 📋 Filter Kartu Kendali

**Kartu Kendali menampilkan tiket dengan kriteria:**

1. ✅ **Type:** `perbaikan`
2. ✅ **Status:** Bukan `rejected`
3. ✅ **Repair Type:** Bukan `direct_repair` (dari diagnosis)
4. ✅ **Work Order:** Harus ada work order dengan status `completed`

**Tiket yang TIDAK muncul di kartu kendali:**
- ❌ Tiket selain perbaikan (zoom, konsultasi, instalasi, dll)
- ❌ Tiket dengan status `rejected`
- ❌ Tiket dengan diagnosis `direct_repair`
- ❌ Tiket yang belum punya work order completed

---

## 🧪 Testing dengan Seeder

### **1. Setup Database**

```bash
# Reset database (HATI-HATI: Menghapus semua data!)
cd backend
php artisan migrate:fresh

# Seed user default
php artisan db:seed

# Seed 50 kartu kendali test data
php artisan db:seed --class=KartuKendaliTestSeeder
```

### **2. Data yang Dibuat Seeder**

**50 Kartu Kendali (Valid):**
- 50 tiket perbaikan dengan work order completed
- Variasi status: in_progress, on_hold, closed, waiting_for_submitter
- Variasi asset: PC, Laptop, Printer, Scanner
- Random dates dalam 6 bulan terakhir
- **Expected: 4 pages (15 per page)**

**10 Tiket Excluded (Tidak Valid):**
- 10 tiket dengan diagnosis `direct_repair`
- Work order status: requested/pending (BUKAN completed)
- **Expected: TIDAK muncul di kartu kendali**

---

## 🔍 Testing Scenario

### **Test 1: Pagination Penuh**

**API Request:**
```bash
GET /api/kartu-kendali?page=1&per_page=15
```

**Expected Result:**
```json
{
  "data": [...15 items...],
  "pagination": {
    "total": 50,
    "per_page": 15,
    "current_page": 1,
    "last_page": 4,
    "from": 1,
    "to": 15
  }
}
```

✅ **PASS:** Page 1 harus punya 15 data penuh (bukan 3-5 data)

---

### **Test 2: Semua Page Konsisten**

**API Requests:**
```bash
GET /api/kartu-kendali?page=1&per_page=15  # Expected: 15 items
GET /api/kartu-kendali?page=2&per_page=15  # Expected: 15 items
GET /api/kartu-kendali?page=3&per_page=15  # Expected: 15 items
GET /api/kartu-kendali?page=4&per_page=15  # Expected: 5 items (50 % 15 = 5)
```

✅ **PASS:** Total data = 50 (bukan lebih karena filter)

---

### **Test 3: Filter Excludes Direct Repair**

**Query Database:**
```bash
php artisan tinker
```

```php
// Count tiket perbaikan total
$totalPerbaikan = \App\Models\Ticket::where('type', 'perbaikan')->count();
// Expected: 60 (50 valid + 10 invalid)

// Count kartu kendali (dengan filter)
$kartuKendali = \App\Models\Ticket::where('type', 'perbaikan')
    ->whereHas('workOrders', function($q) {
        $q->where('status', 'completed');
    })
    ->count();
// Expected: 50 (hanya yang valid)
```

✅ **PASS:** 10 tiket dengan direct_repair TIDAK masuk kartu kendali

---

### **Test 4: Search Functionality**

**API Request:**
```bash
GET /api/kartu-kendali?search=PC-001&per_page=15
```

**Expected:**
- Hanya tiket dengan kode asset `PC-001`
- Pagination tetap bekerja (jika hasil > 15)

✅ **PASS:** Search filter data yang sudah terfilter

---

### **Test 5: Status Filter**

**API Request:**
```bash
GET /api/kartu-kendali?status=closed&per_page=15
```

**Expected:**
- Hanya tiket dengan status `closed`
- Total data < 50 (karena filter tambahan)

✅ **PASS:** Status filter bekerja pada data yang sudah terfilter

---

## 🔧 Manual Testing Commands

### **Check Total Kartu Kendali**
```bash
cd backend
php artisan tinker

# Count kartu kendali
\App\Models\Ticket::where('type', 'perbaikan')
    ->where('status', '!=', 'rejected')
    ->whereHas('diagnosis', function($q) {
        $q->where('repair_type', '!=', 'direct_repair');
    })
    ->whereHas('workOrders', function($q) {
        $q->where('status', 'completed');
    })
    ->count();
```

### **Test API Response**
```bash
# Test dengan curl
curl -X GET "http://localhost:8000/api/kartu-kendali?page=1&per_page=15" \
  -H "Authorization: Bearer YOUR_TOKEN"

# Test dengan httpie (install: pip install httpie)
http GET localhost:8000/api/kartu-kendali page==1 per_page==15 \
  "Authorization: Bearer YOUR_TOKEN"
```

---

## 📊 Expected Database State

**After seeding:**

| Table | Count | Notes |
|-------|-------|-------|
| `tickets` (type=perbaikan) | 60 | 50 valid + 10 invalid |
| `diagnosis` | 60 | Semua tiket punya diagnosis |
| `work_orders` (status=completed) | 50 | Hanya valid yang punya WO completed |
| **Kartu Kendali Result** | **50** | **Yang muncul di API** |

---

## 🎯 Key Takeaways

1. **Filter SEBELUM pagination** untuk hasil konsisten
2. **whereHas()** untuk filter relasi sebelum ambil data
3. **with()** untuk eager load relasi (performance)
4. **Pagination total** harus sesuai dengan data terfilter

---

## 🐛 Debugging Tips

### **Problem: Data masih terpotong**
```bash
# Check query log
cd backend
tail -f storage/logs/laravel.log

# Enable query log di controller
\DB::enableQueryLog();
$tickets = $query->paginate($perPage);
dd(\DB::getQueryLog());
```

### **Problem: Work order tidak ter-filter**
```php
// Check work order status
\App\Models\WorkOrder::where('ticket_id', TICKET_ID)
    ->where('status', 'completed')
    ->exists(); // Harus true
```

---

Last Updated: December 11, 2025
