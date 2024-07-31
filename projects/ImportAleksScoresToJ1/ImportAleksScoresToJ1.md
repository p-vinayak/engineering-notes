### About
- Import active ALEKS cohort scores to J1 that meets Amy Benenati's criteria for import.

### Notes
- Active ALEKS cohorts are stored in an integration table within GriffinDB.

### Pseudocode
```php
$classes = $integrationsDatabaseDataService->getAllActiveAleksClasses();
$scoresToImport = [];
foreach ($classes as $class) {
	$scores = $aleksApiResourceService->getAllPlacementScoreReportsForClass($class, $accessToken, null, null, 'last');
	foreach ($scores as $score) {
		if ($this->shouldImportAleksScore($score)) {
			$scoresToImport[] = $score;
		}
	}
}
$idNums = // $idNums from $scoresToImport
$j1AleksScores = $j1DataService->getAllAleksScoresWhereIdNum($idNums);
$j1AleksScoresToIdNumMap = [
	'111111' => [// ALEKS TEST_SCORES row]
];

$scoresToCreate = [];
$scoresUpdatedRowAppIds = [];
foreach ($scoresToImport as $score) {
	if (! array_key_exists($j1AleksScoresToIdNumMap, $score['sis_id'])) {
		$scoresToCreate[] = $score;
	}
	else {
		$scoresUpdatedRowAppIds[] = $j1DataService->updateAleksScore($score); // This method will do the diffs
	}
}

$scoresCreatedRowAppIds = $j1DataService->createAleksScores($scoresToCreate); // Do whatever contract conversion you need

$scoresUpdatedOrCreatedRowAppIds = array_merge($scoresCreatedRowAppIds, $scoresUpdatedRowAppIds);

DB::connection('j1')->table('TEST_SCORES')->where('TST_CODE', '=', 'ALEKS')->whereNotIn('AppID', $scoresUpdatedOrCreatedRowAppIds)->delete();
```

#### Amy's Criteria for a Valid Score
- A score is valid for 2 registration periods after it has been taken
- If an ALEKS exam was taken before the end of the add/drop period for fall 2024, then that score will be valid for fall 2024 and spring 2025.
- If an ALEKS exam was taken after the add/drop period for fall 2024, then that score will be valid for spring 2025 and fall 2025.
- The start date of the semester isn't important, only the add/drop date.
- Example:

| Semester    | Start Date | Add Drop End |
| ----------- | ---------- | ------------ |
| Fall 2023   | 2023-08-21 | 2023-08-25   |
| Spring 2024 | 2024-01-16 | 2024-01-22   |
| Fall 2024   | 2024-08-26 | 2024-08-30   |

| Score Taken Date | Valid From | Valid Until |
| ---------------- | ---------- | ----------- |
| 2023-08-20       | 2023-08-20 | 2023-01-22  |
| 2023-08-22       | 2023-08-22 | 2023-01-22  |
| 2023-08-26       | 2023-08-26 | 2024-08-30  |
