## What changed between version pr-1403 and the current version of the codebase ##
#### Keep in mind: the current version of the codebase's uploadMeters.js file is behind pr-1403's because of the changes of past contributors (pr-1403 is 23 commits ahead) ####
This document will be a way for me to keep track of what I changed.

## Added: ##
1. `const moment = require('moment-timezone');`
2. Refactored `isValidGPSInput(input)` from:

Oldest version in development (main) branch.
```Js
function isValidGPSInput(input) {
	let message = '';
	let validGps = true;
	if (input.indexOf(',') === -1) { // if there is no comma
		message = 'GPS Input is missing a comma';
		validGps = false;
	} else if (input.indexOf(',') !== input.lastIndexOf(',')) { // if there are multiple commas
		message = 'GPS Input has too many commas';
		validGps = false;
	}
	if (validGps) {
		// Works if value is not a number since parseFloat returns a NaN so treated as invalid later.
		const array = input.split(',').map((value) => parseFloat(value));
		const latitudeIndex = 0;
		const longitudeIndex = 1;
		const latitudeConstraint = array[latitudeIndex] >= -90 && array[latitudeIndex] <= 90;
		const longitudeConstraint = array[longitudeIndex] >= -180 && array[longitudeIndex] <= 180;
		const result = latitudeConstraint && longitudeConstraint;
		if (!result) {
			validGps = false;
			message = 'Invalid GPS coordinate, latitude must be an integer between -90 and 90, longitude must be an integer between -180 and 180. You input: ' + input;
		}
	}
	return { validGps, message };
}
```

Version wrote by SageMar & Other in Pr-1403:
```Js
function isValidGPSInput(input) {
	if (input.indexOf(',') === -1) { // if there is no comma
		return false;
	} else if (input.indexOf(',') !== input.lastIndexOf(',')) { // if there are multiple commas
		return false;
	}
	// Works if value is not a number since parseFloat returns a NaN so treated as invalid later.
	const array = input.split(',').map(value => parseFloat(value));
	const latitudeIndex = 0;
	const longitudeIndex = 1;
	const latitudeConstraint = array[latitudeIndex] >= -90 && array[latitudeIndex] <= 90;
	const longitudeConstraint = array[longitudeIndex] >= -180 && array[longitudeIndex] <= 180;
	const result = latitudeConstraint && longitudeConstraint;
	return result;
}
```

Version written by me (trying to clean up the code & improve readability):
```Js
function isValidGPSInput(input) {
	// if there is no comma or there are multiple commas return false 
	if (input.indexOf(',') === -1 || input.indexOf(',') !== input.lastIndexOf(',')) {
		return false;
	}
	
	// Works if value is not a number since parseFloat returns a NaN so treated as invalid later.
	const array = input.split(',').map((value) => parseFloat(value));
	const latitudeIndex = 0;
	const longitudeIndex = 1;
	const latitudeConstraint = array[latitudeIndex] >= -90 && array[latitudeIndex] <= 90;
	const longitudeConstraint = array[longitudeIndex] >= -180 && array[longitudeIndex] <= 180;

	return latitudeConstraint && longitudeConstraint;	
}
```
**What changed?**
- redid how function returns
- redid if statement

3. `isValidArea(areaInput)`
New check by SageMar & Partner:

Pr-1403:
```Js
/**
 * Checks if the area provided is a number and if it is larger than zero.
 * @param areaInput the provided area for the meter
 * @returns true or false
 */
function isValidArea(areaInput) {
	// check for non-number input, which is not allowed
	if (Number.isNaN(areaInput)){
		return false;
	}

	// must be a number and must be non-negative
	if (areaInput > 0) {
		return true;
	} else {
		return false;
	}
}
```

My version:
```Js
/**
 * Checks if the area provided is a number and if it is larger than zero.
 * @param areaInput the provided area for the metere
 * @returns boolean
 */
function isValidArea(areaInput) {
	// check for non-number inputs, which are not allowed
	if (Number.isNaN(areaInput)) return false;

	// must be a non-negative number
	if (areaInput <= 0) {
		return false;
	}	
	return true;
}
```
**What changed?**
- redid the logic

Question for mentor: <br>
How do I tell if that number `areaInput` can have a value of 0?

When rewriting the code I just followed the same logic as the previous contributors but I think that since 0 is a number it may be able to be handled. -Future kyle: I discovered in the javadoc that they want to exclude 0's but I don't know why.


4. `isValidAreaUnit(areaUnit)`
Unique to Pr-1403. SageMar's version: 
```Js
/**
 * Checks if the area unit provided is an option
 * @param areaUnit the provided area for the meter
 * @returns true or false
 */
function isValidAreaUnit(areaUnit) { 
    const validTypes = Object.values(Unit.areaUnitType); 
    // must be one of the three values 
    if (validTypes.includes(areaUnit)) { 
        return true; 
    } else { 
        return false; 
    } 
}
```

My version:
```Js
/**
 * Checks if the area unit provided is an option
 * @param areaUnit the provided area for the meter
 * @returns boolean
 */
function isValidAreaUnit(areaUnit) {
	const validTypes = Object.values(Unit.areaUnitType);

	if (!validTypes.includes(areaUnit)) {
		return false;
	}
	return true;
}
```
**What changed?**
- redid logic

Question for mentor:
Would it be faster to use SageMar's version since it's using a search algorithm and returning true when it finds the item? Or should I be going for readability over some small time saving thing like that?

5. `isValidTimeSort(timeSortValue)`

Unique to pr-1403. SageMar's version:
```Js
/**
 * Checks if the time sort value provided is accurate (should be increasing or decreasing)
 * @param timeSortValue the provided time sort
 * @returns true or false
 */
function isValidTimeSort(timeSortValue) {
	const validTimes = Object.values(MeterTimeSortTypesJS);
	// must be one of the three values
	if (validTimes.includes(timeSortValue)) {
		return true;
	} else {
		return false;
	}
}
```

My version:
```Js
/**
 * Checks if the time sort value provided is accurate (should be increasing or decreasing)
 * @param timeSortValue the provided time sort
 * @returns boolean
 */
function isValidTimeSort(timeSortValue) {
	const validTimes = Object.values(MeterTimeSortTypesJS);
	
	// must be one of three values
	if (!validTimes.includes(timeSortValue)) {
		return false;
	}
	return true;
}
```
**What changed?**
- redid the logic

I have the same question as 5.

6. `isValidMeterType(meterTypeString)`
Unique to pr-1403. SageMar's version:
```Js
/**
 * Checks if the meter type provided is one of the 5 options allowed when creating a meter.
 * @param meterTypeString the string for the meter type
 * @returns true or false
 */
function isValidMeterType(meterTypeString) {
	const validTypes = Object.values(Meter.type);
	if (validTypes.includes(meterTypeString)) {
		return true;
	} else {
		return false;
	}
}
```

My version: 
```Js 
/**
 * Checks if the meter type provided is one of the 5 options allowed when creating a meter.
 * @param meterTypeString the string for the meter type
 * @returns boolean
 */
function isValidMeterType(meterTypeString) {
	const validTypes = Object.values(Meter.type);

	if (!validTypes.includes(meterTypeString)) {
		return false;
	}
	return true;
}
```
**What changed?**
- redid the logic

Same questions as before :)

7. `isValidTimeZone(timeZone)`
Unique to pr-1403. SageMar's version:
```Js
/**
 * Checks the provided time zone and if it is a real time zone.
 * @param zone the provided time zone from the csv
 * @returns true or false
 */
function isValidTimeZone(zone) {
	const validZones = moment.tz.names();
	if (validZones.includes(zone)) {
		return true;
	} else {
		return false;
	}
}
```

My version:
Changed the parameter for readability.
```Js
/**
 * Checks the provided time zone and if it's a real time zone
 * @param timeZone the provided time zone from the csv
 * @returns boolean
 */
function isValidTimeZone(timeZone) {
	const validZones = moment.tz.names();

	if (!validZones.includes(timeZone)) {
		return false;
	}
	return true;
}
```
**What changed?**
- redid the logic

Same question as before :) 

8. `validateBooleanFields(meter, rowIndex)`
Unique to pr-1403. SageMar's version:
```Js
/**
 * Validates all boolean-like fields for a given meter row.
 * @param {Array} meter - A single row from the CSV file.
 * @param {number} rowIndex - The current row index for error reporting.
 */
function validateBooleanFields(meter, rowIndex) {
	// all inputs that involve a true or false all being validated together.
	const booleanFields = {
		2: 'enabled',
		3: 'displayable',
		10: 'cumulative',
		11: 'reset',
		18: 'end only',
		32: 'disableChecks'
	};

	// this array has values which may be left empty
	const booleanUndefinedAcceptable = [
		'cumulative', 'reset', 'end only', 'disableChecks'
	];

	for (const [index, name] of Object.entries(booleanFields)) {
		let value = meter[index];

		// allows upper/lower case.
		if ((value === '' || value === undefined) && booleanUndefinedAcceptable.includes(name)) {
			// skip if the value is undefined
			continue;
		} else {
			if (typeof value === 'string') {
				value = value.toLowerCase();
			}
		}

		// Validates read values to either false or true
		if (value !== 'true' && value !== 'false' && value !== true && value !== false
			&& value !== 'yes' && value !== 'no') {
			throw new CSVPipelineError(
				`Invalid input for '${name}' in row ${rowIndex + 1}: "${meter[index]}". Expected 'true' or 'false'.`,
				undefined,
				500
			);
		}
	}
}
```

My version: 
```Js
/**
 * Validates all boolean-like fields for a given meter row.
 * @param {Array} meter - A single row from the CSV file.
 * @param {Number} rowindex - the current row index for error reporting.
 */
function validateBooleanFields(meter, rowIndex) {
	const booleanFields = {
		2: 'enabled',
		3: 'displayable',
		10: 'cumulative',
		11: 'reset',
		18: 'end only',
		32: 'disableChecks'
	};
	
	// this array has values which may be left empty
	const booleanUndefinedAcceptable = [
		'cumulative', 'reset', 'end only', 'disableChecks'
	];

	for (const [index, name] of Object.entries(booleanFields)) {
		let value = meter[index];

		// allows upper/lower case
		if (typeof value === 'string' && !(value === '' || value === undefined) && !booleanUndefinedAcceptable.includes(name)) {
            // if value exists standardize it
            value = value.toLowerCase();
		} else {
			// skip undefined value
			continue;
		}

		// define list of accepted values to increase readability
		const acceptedValues = ['true', 'false', 'yes', 'no', true, false];

		// Validates read values to either true or false
		if (!acceptedValues.includes(value)) {
			throw new CSVPipelineError(
				`Invalid input for '${name}' in row ${rowIndex+1}: "${meter[index]}". Expected 'true' or 'false'.`,
				undefined,
				500
			);
		}
	}
}
```

I have the same runtime based question for using a searching algorithm inside of a for loop.

Also: Should I keep that really long if statement in there or should I go back to how it was originally programmed?


9. `validateMinMaxValues(meter, rowIndex)`
Unique to pr-1403. SageMar version: 
```Js
function validateMinMaxValues(meter, rowIndex) {
	const minValue = Number(meter[27]);
	const maxValue = Number(meter[28]);

	if (isNaN(minValue) || minValue < -9007199254740991 || (!isNaN(maxValue) && minValue > maxValue)) {
		throw new CSVPipelineError(
			`Invalid min/max values in row ${rowIndex + 1}: min="${minValue}", max="${maxValue}". ` +
			`Min or/and max must be a number larger than -9007199254740991, and less then 9007199254740991, and min must be less than max.`,
			undefined,
			500
		);
	}

		// do nothing, pass it through
	if (isNaN(maxValue) || maxValue > 9007199254740991) {
		throw new CSVPipelineError(
			`Invalid min/max values in row ${rowIndex + 1}: min="${minValue}", max="${maxValue}". ` +
			`Min or/and max must be a number larger than -9007199254740991, and less then 9007199254740991, and min must be less than max.`,
			undefined,
			500
		);
	}
}
```

My version (revamped the logic):
## Need to double and triple check this logic to make sure it's the same as the one from Sage's version ##
```Js
/**
 * Checks to see if meter values are out of bounds.
 * @param meter the current meter object
 * @param rowIndex the current row being accessed
 */
function validateMinMaxValues(meter, rowIndex) {
	const minValue = Number(meter[27]);
	const maxValue = Number(meter[28]);	
	
	if ((isNaN(minValue) || isNaN(maxValue)) || (minValue < Number.MIN_SAFE_INTEGER || maxValue > Number.MAX_SAFE_INTEGER) || minValue > maxValue) {
		throw new CSVPipelineError(
			`Invalid min/max values in row ${rowIndex + 1}: min="${minValue}", max="${maxValue}".` +
			`Min and/or max mus be a number in between -900719925470991 and 9007199254740991, and min must be less than max.`
		)
	}
}
```

What changed:
Added javadoc, re-did the logic.

##### The logic could also look like this #####
```Js
if ((isNaN(minValue) || isNaN(maxValue)) || !(minValue >= Number.MIN_SAFE_INTEGER && maxValue <= Number.MAX_SAFE_INTEGER) || minValue > maxValue) 
```
This version would do the same thing (hopefully) as the above version but it excludes a range of numbers from the min safe integer to the max.
