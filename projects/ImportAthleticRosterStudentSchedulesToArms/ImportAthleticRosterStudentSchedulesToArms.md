## About
Imports student schedules for athletic roster (current year term) from J1 to ARMS.

### Questions
- Can you pull rosters from ARMS?
- Can both ARMS jobs be consolidated into this one?
- What kind of term codes should the job process?
	- Just "FA"/"SP"? (traditional year term codes)
	- Or "SU" as well?(supplementary terms)

### Notes
- Current year term logic (from Chelsea)
	- Fall window (from end of spring to end of fall)
		- EG: May 5th (example spring end date) to December 3rd (example fall end date)
	- Spring window (from end of fall to end of spring)
		- EG: December 3rd (example fall end date) to May 5th (example spring end date)
	- J-term window (from end of fall to end of J term)
		- EG: December 3rd (example fall end date) to Jan 14th (example J term end date)
### Tips
- API allows dry runs for testing

### Resources
- [API Documentation](https://teamworksapp.com/docs/retain)
- API Credentials in 1Password ("Teamworks Credentials")

### Implementation

```php
public function handle()
{
	$currentYearTerm = $j1DataService->getCurrentYearTerm(); // OR
	$currentYearTerm = $j1DataService->getCurrentTraditionalYearTerm();
	
	$roster = $j1DataService->getAthleticRosterForYearTerm("2425", "FA")
	$ids = // convert roster response to id nums here
	$schedules = $j1DataService->getStudentSchedulesForYearTerm($ids, "2425", "FA")
	$csv = // convert schedule to CSV (may require more data than just schedules)
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

### Tests
- Appropriately converts J1 schedule query response to CSV
- Appropriately calls student schedule query based on roster query response