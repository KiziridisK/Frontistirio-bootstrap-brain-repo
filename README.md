# Frontistirio — Bootstrap Brain

> **Full-stack reference** for the bootstrap endpoint — the single API call that loads all initial data after login. Documents the exact backend implementation, what each role receives, the frontend dispatch sequence, and common failure modes.

---

## What Bootstrap Does

Instead of 15 separate API calls on app load, a single `GET /bootstrap/getSingle` loads everything the frontend needs and populates the entire NgRx store. The response shape is **role-dependent** — the backend reads the JWT, detects the role, and returns only what that role needs.

---

## Backend: Route

```
GET /bootstrap/getSingle
Middleware: authMiddleware (JWT required)
Controller: controllers/bootstrap.js → exports.getSingle
No role middleware — branching is done inside the controller
```

---

## Backend: `getSingle` — Full Logic

```javascript
exports.getSingle = async (req, res) => {
  const userId = req.user.id;
  const userRole = req.user.role;

  // Get default period for all non-superadmin roles
  let default_period;
  if (userRole !== 'superadmin') {
    default_period = await teachinPeriodHandler.getStoreDefaultPeriod(req.user?.store);
  }
  const periodId = default_period?._id;
  // ...
};
```

### Branch: `superadmin`
Uses `Promise.all` for 3 parallel queries:
```javascript
promises.push(userHandler.findUserById(userId));
promises.push(groupHandler.getAll(true));        // all groups with stores
promises.push(storeHandler.getAllStores());       // all stores
```
Response: `{ success, user, groups, stores }`

### Branch: `store-user`
Uses `Promise.all` for **15 parallel queries**:
```javascript
promises.push(studentHandler.fetchStorePeriodStudents(storeId, periodId));   // [0]
promises.push(coursesHandler.fetchStorePeriodCourses(storeId, periodId));    // [1]
promises.push(classesHandler.fetchStorePeriodClasses(storeId, periodId));    // [2]
promises.push(gradersHandler.fetchStoreGrades(storeId));                     // [3]
promises.push(userHandler.findUserById(userId));                             // [4]
promises.push(gradeCategoiesHandler.getGradeCategories());                   // [5] — all categories, not store-scoped
promises.push(teachersHandler.fetchStorePeriodTeachers(storeId, periodId));  // [6]
promises.push(teachingPeriodsHandler.fetchStoreTeachingPeriods(storeId));    // [7]
promises.push(educationalMaterialHandler.getStoreEducationalMaterials(storeId)); // [8]
promises.push(storeHandler.getStore(storeId));                               // [9] — wrapped in array: [store]
promises.push(gradeScenariosHandler.getGradeScenarios());                    // [10] — all scenarios
promises.push(gradeScalesHandler.getGradeScales());                          // [11] — all scales
promises.push(testCycleHandler.getStoreTestCycles(storeId, periodId));       // [12]
promises.push(classroomsHandler.fetchStoreClassRooms(storeId));              // [13]
promises.push(storeSettingsHandler.GetStoreSettings(storeId));               // [14]
```

Response:
```json
{
  "success": true,
  "students": [],
  "courses": [],
  "classes": [],
  "grades": [],
  "user": {},
  "gradeCategories": [],
  "teachers": [],
  "teaching_periods": [],
  "educational_materials": [],
  "stores": [{}],
  "gradeScenarios": [],
  "gradeScales": [],
  "test_cycles": [],
  "classrooms": [],
  "store_settings": {}
}
```

### Branch: `student`
```javascript
// 1. Fetch the student document by user_id, filtered to current period
const student = await studentHandler.getStudentByUserId(userId, default_period._id);

// 2. Find the student's courses and class for this period
const period_courses = student.period_courses.find(
  pc => pc.period.toString() === default_period._id.toString()
);
const period_classes = student.period_class.find(
  pc => pc.period.toString() === default_period._id.toString()
);

// 3. 6 parallel queries for only what the student needs
promises.push(classesHandler.getClassesByIds([period_classes.classes]));
promises.push(coursesHandler.getCoursesById(period_courses.courses, storeId));
promises.push(userHandler.findUserById(userId));
promises.push(storeHandler.getStore(storeId));
promises.push(educationalMaterialHandler.getStudentEducationalMaterial(storeId, student, periodId));
promises.push(teachinPeriodHandler.fetchStoreTeachingPeriods(storeId));
promises.push(gradersHandler.fetchStoreGrades(storeId));
```

Response: `{ success, students: [student], courses, classes, educational_materials, grades, user, stores, teaching_periods }`

Note: student only gets their **own** courses and classes, not the store's full list. Educational materials are filtered by `getStudentEducationalMaterial` which checks `period_permissions`.

### Branch: `parent`
Not yet implemented — only logs "fetching bootstrap data of parent".

### Branch: `teacher`
Not yet implemented — only logs "fetching bootstrap data of teacher".

---

## Frontend: Bootstrap Trigger (`login.component.ts`)

Called inside `handleBootstrapAndRedirect()`:
```typescript
// AppStateService:
public getBootstrapData() {
  return this.http.get<any>(`${environment.apiUrl}/bootstrap/getSingle`);
}
```

### Full NgRx Dispatch Sequence
```typescript
if (data.groups?.length > 0)
  store.dispatch(setGroups({ groups: data.groups }));
if (data.stores?.length > 0)
  store.dispatch(setStores({ stores: data.stores }));
if (data.user)
  store.dispatch(setUser({ user: data.user }));
if (data.gradeCategories?.length > 0)
  store.dispatch(setGradeCategories({ gradeCategories: data.gradeCategories }));
if (data.gradeScenarios?.length > 0)
  store.dispatch(setGradeScenarios({ gradeScenarios: data.gradeScenarios }));
if (data.gradeScales?.length > 0)
  store.dispatch(setGradeScales({ gradeScales: data.gradeScales }));
if (data.grades?.length > 0)
  store.dispatch(setGrades({ grades: data.grades }));
if (data.classes?.length > 0)
  store.dispatch(setClasses({ classes: data.classes }));
if (data.courses?.length > 0)
  store.dispatch(setCourses({ courses: data.courses }));
if (data.teachers?.length > 0)
  store.dispatch(setTeachers({ teachers: data.teachers }));
if (data.students?.length > 0)
  store.dispatch(setStudents({ students: data.students }));
if (data.teaching_periods?.length > 0)
  store.dispatch(setTeachingPeriods({ teaching_periods: data.teaching_periods }));
if (data.educational_materials?.length > 0)
  store.dispatch(setEducationalMaterial({ materials: data.educational_materials }));
if (data.test_cycles?.length > 0)
  store.dispatch(setTestCycles({ test_cycles: data.test_cycles }));
if (data.classrooms?.length > 0)
  store.dispatch(setClassrooms({ classrooms: data.classrooms }));
if (data.store_settings)
  store.dispatch(setStoreSettings({ settings: data.store_settings }));
```

Each `if` check means **empty arrays are not dispatched** — reducers keep their initial empty state for missing data.

After `.complete()`:
```typescript
.complete: () => {
  this.router.navigate(['/home']);
}
```

---

## Frontend: Home Page Uses Bootstrap Data

`home.page.ts` uses `combineLatest` of NgRx selectors to reactively build the dashboard:
```typescript
combineLatest([
  store.select(selectUser),
  store.select(selectTeachingPeriods),
  store.select(selectAllStudents),
  store.select(selectAllCourses),
  store.select(selectAllClasses),
  store.select(selectAllGrades)
]).pipe(
  filter(([user]) => !!user && !!user._id),
  map(([user, periods, students, courses, classes, grades]) => {
    const defaultPeriod = periods?.find(p => p.default === true) || null;
    // For student role: find student by user_id, then find period_class and period_grade
    // For store-user: stats/counts from loaded arrays
    // ...
  })
)
```

For students: also fetches upcoming test cycles via `studentsService.getStoreStudentTestCycleTests(student._id)`.

---

## Critical: Default Period Dependency

**Everything in bootstrap is scoped to `TeachingPeriod.default === true` for the store.**

If a store-user logs in and sees empty lists:
1. Check if the store has a period where `default: true` — if not, `getStoreDefaultPeriod` returns null and the bootstrap will return empty arrays
2. `getStoreDefaultPeriod(storeId)` in `handlers/teaching_periods.js`: `TeachingPeriod.findOne({ store_id, default: true, isDeleted: false })`
3. Even if periods exist, if none is marked default, all 15 parallel queries get `periodId = undefined` and return nothing

---

## Common Failures

| Symptom | Likely cause |
|---|---|
| Empty students/courses/classes after login | No default teaching period set |
| `error getting bootstrap data` in logs | Unhandled exception in one of the 15 parallel queries; check server logs |
| `parent`/`teacher` role gets empty response | Not implemented — falls through all if-branches |
| Student sees no materials | `getStudentEducationalMaterial` found no materials matching student's grade/class in `period_permissions` |
