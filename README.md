# Frontistirio — Bootstrap Brain

> **Full-stack reference** for the bootstrap endpoint — the single API call that loads all initial data after login. Documents the exact backend implementation, what each role receives, the frontend dispatch sequence, and common failure modes.

---

## Change log

### 2026-09-21 — corrections (verified against API `93b4079`)
This README had three statements that were no longer true:
- **The `parent` branch IS implemented** (since 2026-07-14). It previously said "Not yet implemented". See [Branch: `parent`](#branch-parent).
- **The `student` branch no longer reads only `student.period_courses`.** Since BS-05 (2026-07-17) the courses are the union of the
  student's own courses and their τμήμα's (`syllabusHandler.getStudentPeriodCourseIds`). The lookups are also null-safe now.
- **The period is `resolveActivePeriod(req)`, not `getStoreDefaultPeriod`.** It is the store default for every role, except a store-user
  who has entered another period (admin active-period override, see the authentication brain repo).

Student and parent responses now also carry `store_settings` = **branding only** (`GetStoreBranding`), so they see the store's logo
and colours. The store-user and the teacher get the full settings (`GetStoreSettings`).

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

  // Active period for all non-superadmin roles. resolveActivePeriod returns the
  // store default, except for a store-user with a personal override
  // (User.active_period / JWT claim) — teachers/students/parents always get the default.
  let default_period;
  if (userRole !== 'superadmin') {
    default_period = await teachinPeriodHandler.resolveActivePeriod(req);
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

// 2. Class + courses for this period — both may be missing (assignment is optional),
//    so everything is null-safe (`student?.period_class`, empty id lists).
const period_classes = _.find(student?.period_class, pc => pc.period.toString() === periodId.toString());
// Courses = the UNION of the student's own period_courses (ιδιαίτερα) and the courses
// of their τμήμα (ClassModel.period_courses, never copied onto the student). BS-05.
const courseIds = student ? await syllabusHandler.getStudentPeriodCourseIds(student, periodId) : [];
const classIds = period_classes?.classes ? [period_classes.classes] : [];

// 3. 8 parallel queries for only what the student needs
promises.push(classesHandler.getClassesByIds(classIds));                         // 0
promises.push(coursesHandler.getCoursesById(courseIds, storeId));                // 1
promises.push(userHandler.findUserById(userId));                                 // 2
promises.push(storeHandler.getStore(storeId));                                   // 3
promises.push(student ? educationalMaterialHandler.getStudentEducationalMaterial(storeId, student, periodId) : []); // 4
promises.push(teachinPeriodHandler.fetchStoreTeachingPeriods(storeId));          // 5
promises.push(gradersHandler.fetchStoreGrades(storeId));                         // 6
promises.push(storeSettingsHandler.GetStoreBranding(storeId));                   // 7 — branding only
```

Response: `{ success, students: [student] | [], courses, classes, educational_materials, grades, user, stores, teaching_periods, store_settings }`

Note: the student only gets their **own** courses and classes, not the store's full list. Educational materials are filtered by
`getStudentEducationalMaterial`, which checks `period_permissions`. Its course-level check still reads only `student.period_courses`
(BUG-013, open). Material attached to a course syllabus is reachable anyway; see the course-syllabus brain repo.

### Branch: `parent` (implemented 2026-07-14)
The parent sees the **same shapes as the student**, scoped to their **children**:
```javascript
const parentIds = await parentsHandler.getParentIdsByUserId(userId);      // Parent docs of this login
const kids = await studentHandler.getStudentsByParentIds(parentIds, storeId); // reverse lookup on Student.parents.father/mother
// → kidIdSet

// Fetch the SAME period-scoped store collections as student/teacher, then FILTER:
fetchStorePeriodStudents / fetchStorePeriodCourses / fetchStorePeriodClasses,
fetchStoreGrades, findUserById, fetchStoreTeachingPeriods, getStore, GetStoreBranding
```
- `students` = only the children.
- `courses` = the union over the children of **`getStudentPeriodCourseIds`** (own ∪ τμήμα). `classes` = the children's τμήματα.
- `educational_materials` = the union of each child's `getStudentEducationalMaterial`, deduplicated by id.
- `parents` = `getParentsByUserId(userId)`. It is **not dispatched to any NgRx slice**. The frontend resolves the parent's name
  from the children's populated `parents.father/mother` (matching `user_id`).

Response: `{ success, students, courses, classes, grades, user, teaching_periods, educational_materials, stores, parents, store_settings }`

**Resolution is a reverse lookup** on `Student.parents.father/.mother`. The `Parent.kids` array was historically empty. It is now
`$addToSet` when a parent login is linked, but reads do not rely on it. A `Parent` document is created **per student**, so today one parent
login maps to one child. See the student-management brain repo, «Parent portal».

### Branch: `teacher` (implemented 2026-07-06)
Scoped to the teacher's own assignments for the default period. Steps:
```javascript
const teacher = await teachersHandler.getTeacherByUserId(userId);            // "me"
const { courseIds, classIds, studentIds } =                                 // sync, from period_courses
  teachersHandler.getTeacherPeriodAssignmentIds(teacher, periodId);

// Fetch the SAME period-scoped store collections the store-user gets, then FILTER
// to the teacher's subset (guarantees identical shape for the shared components/gradebook):
fetchStorePeriodStudents / fetchStorePeriodCourses / fetchStorePeriodClasses,
fetchStoreGrades, getGradeCategories, getGradeScenarios, getGradeScales,
fetchStoreTeachingPeriods, getStore, GetStoreSettings, findUserById,
educationalMaterialHandler.getTeacherEducationalMaterials(storeId, userId, periodId); // own uploads
```
Filtering: `courses`/`classes` kept if their `_id ∈ courseIds/classIds`; `students` kept if
`_id ∈ studentIds` (private-lesson students) **OR** their active-period `period_class` ∈ `classIds`.
Returns `teachers: [teacher]` so the frontend resolves "me" by `user_id`. Test cycles are NOT in
bootstrap — the teacher home loads upcoming ones lazily (mirrors student), see test-cycles brain repo.

Response keys: `{ success, students, courses, classes, grades, user, gradeCategories, teachers,
teaching_periods, educational_materials, stores, gradeScenarios, gradeScales, store_settings }`.
See the **teacher-management** brain repo for the full teacher-portal picture.

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

**Everything in bootstrap is scoped to `TeachingPeriod.default === true` for the store**, except a store-user who has entered
another period (`resolveActivePeriod` then returns that period).

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
| `parent` sees no children | The login's `User._id` is not on any `Parent.user_id`, or no `Student.parents.father/mother` points to that Parent (the link is made by `register-parent-user`) |
| Student/parent sees no courses although the τμήμα has them | Code that reads only `student.period_courses`; use `getStudentPeriodCourseIds` (BS-05) |
| `teacher` sees empty lists | Teacher has no `period_courses` for the default period (nothing assigned yet), OR their login isn't linked to a Teacher doc (`getTeacherByUserId` null) |
| Student sees no materials | `getStudentEducationalMaterial` found no materials matching student's grade/class in `period_permissions` |
