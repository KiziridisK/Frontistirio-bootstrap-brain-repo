# Frontistirio — Bootstrap Brain

> **Purpose:** Documents the bootstrap endpoint — the single API call the frontend makes on app startup to load all initial data in one shot. Reference this when debugging initial load issues or understanding what data each role receives on login.

---

## Overview

The **bootstrap endpoint** (`GET /bootstrap`) is a role-aware "fat GET" that returns all data the frontend needs to render the initial UI — in a single request instead of many parallel calls.

What data is returned depends entirely on the **user's role**.

---

## Endpoint

```
GET /bootstrap
Authorization: Bearer <token>
```

Response shape varies by role (see below).

---

## Role-Based Responses

### SuperAdmin

```javascript
{
  success: true,
  user: { ...userDoc },
  groups: [ ...allGroups ],
  stores: [ ...allStores ]
}
```

The superadmin sees the full platform topology (groups + stores) to manage the network of tutoring centers.

---

### Store-User (Tutoring Center Admin)

The largest bootstrap response. Loads everything needed for the store management dashboard:

```javascript
{
  success: true,
  user: { ...userDoc },
  store: { ...storeDoc },
  students: [ ...periodStudents ],
  courses: [ ...periodCourses ],
  classes: [ ...periodClasses ],
  grades: [ ...storeGrades ],
  teachers: [ ...periodTeachers ],
  teaching_periods: [ ...storePeriods ],
  grade_categories: [ ...storeGradeCategories ],
  grade_scenarios: [ ...storeGradeScenarios ],
  grade_scales: [ ...storeGradeScales ],
  educational_materials: [ ...storeMaterials ],
  test_cycles: [ ...periodTestCycles ],
  classrooms: [ ...storeClassrooms ],
  store_settings: { ...storeSettings },
  hourly_rates: [ ...storeRates ],
  student_hourly_rates: [ ...studentRates ]
}
```

All entities are scoped to the **default (active) teaching period**.

---

### Student

```javascript
{
  success: true,
  user: { ...userDoc },
  student: { ...studentDoc with period_courses, period_class, period_grade },
  courses: [ ...enrolledCourses ],
  teachers: [ ...studentTeachers ],
  educational_materials: [ ...accessibleMaterials ],
  test_cycles: [ ...relevantTestCycles ]
}
```

Only data relevant to the student's own enrollment and grade level.

---

### Teacher

```javascript
{
  success: true,
  user: { ...userDoc },
  teacher: { ...teacherDoc },
  courses: [ ...assignedCourses ],
  classes: [ ...assignedClasses ],
  students: [ ...teacherStudents ],
  educational_materials: [ ...storeMaterials (visibleToTeachers: true) ]
}
```

---

### Parent

```javascript
{
  success: true,
  user: { ...userDoc },
  children: [ ...studentDocs ],    // their registered children
  educational_materials: [ ...visibleToParents: true ]
}
```

---

## Implementation (`controllers/bootstrap.js`)

The `getSingle` function:
1. Reads `req.user.role` from the JWT
2. Branches to role-specific data fetching
3. Uses `Promise.all()` for parallel fetches to minimize latency
4. All store-level queries use `getStoreDefaultPeriod(store_id)` to scope data

```javascript
const promises = [];
promises.push(studentHandler.fetchStorePeriodStudents(storeId, periodId));
promises.push(coursesHandler.fetchStorePeriodCourses(storeId, periodId));
promises.push(classesHandler.fetchStorePeriodClasses(storeId, periodId));
// ... more parallel fetches
const results = await Promise.all(promises);
```

---

## Why Bootstrap Exists

Instead of the frontend making 10–15 separate API calls on load (causing waterfalls and flicker), the bootstrap endpoint:
- **Eliminates N+1 load waterfalls** — one round trip to the server
- **Returns role-filtered data** — no overfetching
- **Guarantees period consistency** — all data is from the same active period

---

## Period Dependency

The entire bootstrap response depends on the store's **default teaching period**. If the default period is not set or is wrong:
- The bootstrap will return empty arrays for all period-scoped entities
- The frontend will appear to have no students/courses/etc.

Always check `TeachingPeriod.default === true` for the store if bootstrap data looks empty.
