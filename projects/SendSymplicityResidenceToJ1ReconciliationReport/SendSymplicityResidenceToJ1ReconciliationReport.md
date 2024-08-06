## About

This job reconciles data between symplicity residence and j1 by:

- Grabbing all meal and room assignments (archived and unarchived) from Symplicity using their REST API
- Grabbing rows from J1 that Stacy uses to determine billing charges for housing and meals
- Comparing the two to ensure data is correct in both places

## Data in Symplicity

Before reconciliation, the job validates data within Symplicity, because certain types of bad data, for a given user, within Symplicity prevents that user from be reconciled. The script checks the following:

- A user does has no more than 1 unarchived room/meal assignment row per session code. There is an exception to this rule. If the additional unarchived rows are 'pending' rows, the job does not treat it as an error, but as a warning. This is because 'pending' rows do not show up in the CSV export that the GriffinsLair integration uses to process housing and meal imports.
    - More than 1 unarchived row per session code is problematic because both rows show up in the CSV export processed by GriffinsLair, resulting in possibilities where a student is added to a room and removed immediately after (because they might have a placed and a departed row one after the other). More than 1 unarchived row per session (as long as those rows aren't pending) causes too many issues.
- A user has exactly 1 meal assignment and 1 room assignment.
    - This is because a user cannot have a meal without a room and vice versa.

## Data in J1

Data in J1 is retrieved using this query

```sql
DB::connection('j1')->table('STUD_SESS_ASSIGN')
        ->select(['STUD_SESS_ASSIGN.ID_NUM', 'STUD_LIFE_CHGS.BLDG_CDE', 'STUD_LIFE_CHGS.ROOM_CDE', 'STUD_SESS_ASSIGN.MEAL_PLAN', 'STUD_SESS_ASSIGN.RESID_COMMUTER_STS', 'STUD_SESS_ASSIGN.ROOM_ASSIGN_STS'])
        ->leftJoin('STUD_LIFE_CHGS', function ($join) {
            $join->on('STUD_LIFE_CHGS.ID_NUM', '=', 'STUD_SESS_ASSIGN.ID_NUM');
            $join->on('STUD_LIFE_CHGS.SESS_CDE', '=', 'STUD_SESS_ASSIGN.SESS_CDE');
        })
        ->where(['STUD_SESS_ASSIGN.SESS_CDE' => $this->currentResidenceSessionCode])
        ->get()
        ->toArray();
```

## Reconciliation

The main data the script tries to reconcile are:

- BLDG_CDE
- ROOM_CDE
- MEAL_PLAN
- RESID_COMMUTER_STS
- ROOM_ASSIGN_STS

### Comparison Process

- BLDG_CDE from J1 is compared to the building code of the unarchived room assignment row from Symplicity (unless that unarchived row is a 'cancelled' row, then it ensures that BLDG_CDE is null)
- ROOM_CDE from J1 is compared to the room code of the unarchived room assignment row from Symplicity (unless that unarchived row is a 'cancelled' row, then it ensures that ROOM_CDE is null)
- MEAL_PLAN from J1 is compared to the meal code of the unarchived meal assignment row from Symplicity (unless that unarchived row is a 'cancelled' row, then it ensures that MEAL_PLAN is null)
- RESID_COMMUTER_STS from J1 is compared based on building and room values from symplicity. if building and room are cancelled in symplicity, the recon script ensures that RESID_COMMUTER_STS is 'C'. if building and room exist (placed, checkedin, etc) it ensures that RESID_COMMUTER_STS is 'R'.
- ROOM_ASSIGN_STS room J1 is compared based on building and room values from symplicity. if building and room are cancelled in symplicity, the recon script ensures that ROOM_ASSIGN_STS is 'U'. if building and room exist (placed, checkedin, etc) it ensures that ROOM_ASSIGN_STS is 'A'.