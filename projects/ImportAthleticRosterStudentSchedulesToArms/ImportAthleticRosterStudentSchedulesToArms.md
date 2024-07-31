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
		- **There are also cases where a course has a fourth hour. For example SMA 130 meets on MWF at (arbitrary time) but also R (with a different arbitrary time). In that case, a query from J1 would return us two rows for enrollment for a user in SMA 130. One of MWF and the other for R. So, when sending that student to ARMS, we would need to send their two rows as SMA 130 01 and SMA 130 01R. The latter having a random letter that we would need to hold a mapping for. I don't think we need to hold a mapping for this because we can have the letter represent something that's obvious (like the days of the course).**
	- There are a maximum of 2 meeting patterns per course, is this going to cause an issue?
		- 
- API Behavior
	- Can we import schedules for multiple semesters?
		- 
	- What if multiple semesters have courses with 3 different meeting patterms?
		- 
	- Can we get rid of schedules? What if someone's schedule changes?
		- 
	- What happens if we push up student schedules for students who don't currently exist in teamworks? Should we provide optional data fields just in case that happens?
		- Ideally this shouldn't happen because the schedules come from users within SPORTS_TRACKING, which gets its users from ARMS, where users are initially put manually. So everyone whose schedules that are coming from the query should already exist in arms.

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

### Implementation Pseudocode

```php
public function handle()
{
	$currentYearTerm = DB::connection("j1")->query()->...
	$schedules = $j1DataService->getStudentSchedulesForYearTerm("2425", "FA")
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
#### Query to API CSV Conversion Mapping

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

### Scratch Pad
#### "Fourth Hour" Course Logic
```php
$sampleUser = [
	"School ID*" => 330704,
	"Alternate ID" => null,
	"Groups" => null,
	"Student First Name" => null,
	"Student Last Name" => null,
	"Preferred Name" => null,
	"Student Email" => null,
	"Academic Year" => null,
	"College Code" => null,
	"Primary Major Code" => null,
	"Primary Major Description" => null,
	"Cumulative GPA" => null,
	"Race or Ethnicity" => null,
	"Gender" => null,
	"Subject Code*" => "SMA",
	"Subject Name" => null,
	"Class Code*" => "130",
	"Class Section Code*" => "01",
	"Class Description*" => "Calculus I with Analytical Geometry",
	"Credits Attempted" => null,
	"Grade" => null,
	"Score" => null,
	"Class Building/Room" => "BOYLE 370",
	"Monday?*" => "Y",
	"Tuesday?*" => "N",
	"Wednesday?*" => "Y",
	"Thursday?*" => "N",
	"Friday?*" => "Y",
	"Saturday?*" => "N",
	"Sunday?*" => "N",
	"Start Time" => null,
	"End Time" => null,
	"Term ID*" => "Fall 2024",
	"Term Start Date*" => "2024-08-26",
	"Term End Date*" => "2024-12-12",
	"Professor First Name" => null,
	"Professor Last Name" => null,
	"Professor Email" => null,
	"Professor Unique ID" => null,
	"Professor Phone" => null,
	"Professor Office" => null
]
```