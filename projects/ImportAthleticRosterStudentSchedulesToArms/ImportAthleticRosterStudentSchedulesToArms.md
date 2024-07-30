## About
Imports student schedules for athletic roster (current year term) from J1 to ARMS.

### Questions
- Can you pull rosters from ARMS?
	- **No, but it would have been ideal to because `SPORTS_TRACKING` in theory could have rows other than the ones imported from ARMS in the other ARMS integration.**
- Can both ARMS jobs be consolidated into this one?
	- **Yes, but we should keep them separate anyway.**
- What kind of term codes should the job process?
	- Just "FA"/"SP"? (traditional year term codes)
	- Or "SU" as well?(supplementary terms)
	- **Custom logic in the notes section**
- Teamworks engineer said:
	- We need to maintain a mapping of term id/descriptor, what did he mean?
		- **Term ID in arms should be sent over as the term description (not the term ID from the SIS). We are expected to maintain a mapping between term id of the SIS and term description that we provide to ARMS for internal use cases. But, the term description we are planning to send to ARMS is already mapped in J1 to the appropriate year term code, so there's no need to do anything.**
	- There are a maximum of 2 meeting patterns per course, is this going to cause an issue?
		- **According to others, there should never be a course with more than 2 meeting patterns. This is a guess, but a course with two meeting patterns means a course like "SCS 100 01" that has two different meeting times.**
- API Behavior
	- Can we import schedules for multiple semesters?
		- 
	- What if multiple semesters have courses with 3 different meeting patterms?
		- 
	- Can we get rid of schedules? What if someone's schedule changes?
		- 

### Notes
- Current year term logic (from Chelsea)
	- Fall window (from end of spring to end of fall)
		- EG: May 5th (example spring end date) to December 3rd (example fall end date)
	- Spring window (from end of fall to end of spring)
		- EG: December 3rd (example fall end date) to May 5th (example spring end date)
	- J-term window (from end of fall to end of J term)
		- EG: December 3rd (example fall end date) to Jan 14th (example J term end date)
- Teamworks meeting notes (their API has some nuances)

![teamworks_meeting_notes](./teamworks_meeting_notes.png)

### Tips
- API allows dry runs for testing

### Resources
- [API Documentation](https://teamworksapp.com/docs/retain)
- API Credentials in 1Password ("Teamworks Credentials")

### Implementation

```php
public function handle()
{
	$currentYearTerm = DB::connection("j1")->query()->...
	$schedules = $j1DataService->getStudentSchedulesForYearTerm($ids, "2425", "FA")
	$csv = // convert schedule to CSV (may require more data than just schedules)
	// Skip online courses (those will null days/starttime/endtime in the query)
	try {
		$importId = $teamworksApiService->importEnrollments() // internally calls async endpoint and polls until fail or complete
	} catch (\Throwable $exception) {
		$this->error($exception->getMessage())
		return Command::Failure;
	}
	$this->info("Some success message")
	return Command::Success
}
```

### Queries/Sub Logic
#### Query to API CSV Conversion Logic
| API CSV Column            | Query Column                         |
| ------------------------- | ------------------------------------ |
| School ID*                | Student ID                           |
| Alternate ID              |                                      |
| Groups                    |                                      |
| Student First Name        |                                      |
| Student Last Name         |                                      |
| Preferred Name            |                                      |
| Student Email             |                                      |
| Academic Year             |                                      |
| College Code              |                                      |
| Primary Major Code        |                                      |
| Primary Major Description |                                      |
| Cumulative GPA            |                                      |
| Race or Ethnicity         |                                      |
| Gender                    |                                      |
| Subject Code*             | Course Code (first 3 letters parsed) |
| Subject Name              |                                      |
| Class Code*               | Course Code (first 6 letters parsed) |
| Class Section Code*       | Course Code (last 2 digits parsed)   |
| Class Description*        | Course Title                         |
| Credits Attempted         |                                      |
| Grade                     |                                      |
| Score                     |                                      |
| Class Building/Room       | Building + Room                      |
| Monday?*                  | Parsed from Days                     |
| Tuesday?*                 | Parsed from Days                     |
| Wednesday?*               | Parsed from Days                     |
| Thursday?*                | Parsed from Days                     |
| Friday?*                  | Parsed from Days                     |
| Saturday?*                | Parsed from Days                     |
| Sunday?*                  | Parsed from Days                     |
| Start Time                | Start Time                           |
| End Time                  | End Time                             |
| Term ID*                  | *Ask Amanda to add to query*         |
| Term Start Date*          | Start Date                           |
| Term End Date*            | End Date                             |
| Professor First Name      |                                      |
| Professor Last Name       |                                      |
| Professor Email           |                                      |
| Professor Unique ID       |                                      |
| Professor Phone           |                                      |
| Professor Office          |                                      |

#### $currentYearterm
```sql
SELECT * FROM YEAR_TERM_TABLE WHERE TRM_END_DTE > GETDATE() AND TRM_CDE IN ('FA', 'SP') AND TRM_BEGIN_DTE IS NOT NULL ORDER BY TRM_BEGIN_DTE
```

```php
$currentAndFollowingYearTerms | //use query above
if ($currentAndFollowingYearTerms[0]) 
```
#### getStudentSchedulesForYearTerm()
```sql
SELECT*FROM dbo.shu_GetAthleteCourseEnrollmentForARMS('2425', 'FA') WHERE STATUS = 'Enrolled';
```

### Tests
- Appropriately converts J1 schedule query response to CSV
- Appropriately calls student schedule query based on roster query response