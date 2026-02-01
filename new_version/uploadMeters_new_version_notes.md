## Notes on the 2026 version of uploadMeters.js to better understand how to/if I should merge the old pr-1403 branch to it ##
I realize that was a long title but whatever. Let's start off with the imports. 

What are they doing?
This is the full chunk of code for the upload/imports. We will be going through each line to examine what they do and to further understand what it is that changed across versions. Does issue 1217 even still exist or have all the checks been handled already? Is there **more** work to be done on it? 

```js
const express = require('express');
const { CSVPipelineError } = require('./CustomErrors');
const Meter = require('../../models/Meter');
const readCsv = require('../../models/Unit');
const Unit = require('../../models/Unit');
const { normalizeBoolean } = require('./validateCsvUploadParams');
```

The first line seems to be importing a library 'express' and allowing an object, `express` to access the features of the library. *I have yet to do any research into this*
`const express = require('express');`

The second line I recognize. CustomErros.js is a custom set of errors defined in the csvPipeline directory. These are errors that are constant referenced throughout the file.
`const { CSVPipelineError } = require('./CustomErrors');`

The third line appears to be mimic-ing the first, but instead of importing a library or what appears to be a library, it's importing predefined models of Meter. 
`const Meter = require('../../models/Meter');`

The fourth line appears to be doing something very similar to the third line. I've found that Unit.js is a model that looks to be either getting data or putting data into a database.
`const readCsv = require('../../models/Unit');`

The fifth line is doing the same as the fourth line but saving it to a different variable. I wonder if it will be used for something else. Or what the difference is at all.
`const Unit = require('../../models/Unit');`

The sixth line is interesting because it's pulling from a file called `validateCsvUploadParams`. Which I feel is the file we should be editing instead of `uploadMeters.js`.
`const { normalizeBoolean } = require('./validateCsvUploadParams');`
validateCsvUploadParams.js appears to not actually validate the readings from the CSV, which, as I'm looking at the title of the file, actually makes a lot more sense. It's checking to see if the CSV file can be uploaded at all, not checking to see if the values contained within the CSV are bad.

This whole piece of code is meant to be 'middleware' that "uploads meters via the pipeline". This appears to be the main part of the code for this file, and it does appear to have changed from the other version.

I'm going to go through each section and figure out what they do.
```Js
async function uploadMeters(req, res, filepath, conn) {
	const temp = (await readCsv(filepath)).map(row => {
		// The Canonical structure of each row in the Meters CSV file is the order of the fields 
		// declared in the Meter constructor. If no headerRow is provided (i.e. headerRow === false),
		// then we assume that the uploaded CSV file follows this Canonical structure.

		// For now, we do not use the header row to remap the ordering of the columns.
		// To Do: Use header row to remap the indices to fit the Meter constructor
		return row.map(val => val === '' ? undefined : val);
	});

	// If there is a header row, we remove and ignore it for now.
	const meters = normalizeBoolean(req.body.headerRow) ? temp.slice(1) : temp;
	// The original code used a Promise.all to run through the meters. The issue is that the promises are run in parallel.
	// If the meters are independent as expected then this works fine. However, in the error case where one CSV file has
	// the same meter name listed twice, the order of the attempts to add to the database was arbitrary. This meant one of them
	// failed due to the duplicate name but you did not know which one. If some of the information on the two meters differed then
	// you did not know which one you would get in the database. The best result would be the first one in the CSV file would be stored
	// as this makes the most logical sense (no update here) and it is consistent. To make this happen a for loop is used as it
	// is sequential. A small negative is the database requests do not run in parallel in the usual case without an error.
	// However, uploading meters is not common so slowing it down slightly seems a reasonable price to get this behavior.

	try {
		for (let i = 0; i < meters.length; i++) {
			let meter = meters[i];
			// First verify GPS is okay
			// This assumes that the sixth column is the GPS as order is assumed for now in a GPS file.
			const gpsInput = meter[6];
			// Skip if undefined.
			if (gpsInput) {
				// Verify GPS is okay values
				const { validGps, message } = isValidGPSInput(gpsInput);
				if (!validGps) {
					let msg = `For meter ${meter[0]} the gps coordinates of ${gpsInput} are invalid with error of "${message}"`;
					throw new CSVPipelineError(msg, undefined, 500);
				}
				// Need to reverse latitude & longitude because standard GPS gives in that order but a GPSPoint for the
				// DB is longitude, latitude.
				meter[6] = switchGPS(gpsInput);
			}

			// Process unit.
			const unitName = meter[23];
			const unitId = await getUnitId(unitName, Unit.unitType.METER, conn);
			if (!unitId) {
				const msg = `For meter ${meter[0]} the unit of ${unitName} is invalid`;
				throw new CSVPipelineError(msg, undefined, 500);
			}
			// Replace the unit's name by its id.
			meter[23] = unitId;

			// Process default graphic unit.
			const defaultGraphicUnitName = meter[24];
			const defaultGraphicUnitId = await getUnitId(defaultGraphicUnitName, Unit.unitType.UNIT, conn);
			if (!defaultGraphicUnitId) {
				const msg = `For meter ${meter[0]} the default graphic unit of ${defaultGraphicUnitName} is invalid`;
				throw new CSVPipelineError(msg, undefined, 500);
			}
			// Replace the default graphic unit's name by its id.
			meter[24] = defaultGraphicUnitId;

			if (normalizeBoolean(req.body.update)) {
				// Updating the new meters.
				// First get its id.
				let identifierOfMeter = req.body.meterIdentifier;
				if (!identifierOfMeter) {
					// Seems no identifier provided so use one in CSV file.
					if (!meter[7]) {
						// There is no identifier given for meter in CSV so use name as identifier since would be automatically set.
						identifierOfMeter = meter[0];
					} else {
						identifierOfMeter = meter[7];
					}
				} else if (meters.length !== 1) {
					// This error could be thrown a number of times, one per meter in CSV, but should only see one of them.
					throw new CSVPipelineError(`Meter identifier provided (\"${identifierOfMeter}\") in request with update for meters but more than one meter in CSV so not processing`, undefined, 500);
				}
				let currentMeter;
				currentMeter = await Meter.getByIdentifier(identifierOfMeter, conn)
					.catch(error => {
						// Did not find the meter.
						let msg = `Meter identifier of \"${identifierOfMeter}\" does not seem to exist with update for meters and got DB error of: ${error.message}`;
						throw new CSVPipelineError(msg, undefined, 500);
					});
				currentMeter.merge(...meter);
				await currentMeter.update(conn);
			} else {
				// Inserting the new meter
				await new Meter(undefined, ...meter).insert(conn)
					.catch(error => {
						// Probably duplicate meter.
						throw new CSVPipelineError(
							`Meter name of \"${meter[0]}\" got database error of: ${error.message}`, undefined, 500);
					}
					);
			}
		}
	} catch (error) {
		throw new CSVPipelineError(`Failed to upload meters due to internal OED Error: ${error.message}`, undefined, 500);
	}
}
```

This is the first section. This section creates a temporary object that gets a map of values from the csv. This is implementing what appears to be multithreading (use of `await` but I'm not nearly experienced enough in that part of js to say for sure).
There is a `todo` in here that we could address if we fix everything else.
```Js
    const temp = (await readCsv(filepath)).map(row => {
		// The Canonical structure of each row in the Meters CSV file is the order of the fields 
		// declared in the Meter constructor. If no headerRow is provided (i.e. headerRow === false),
		// then we assume that the uploaded CSV file follows this Canonical structure.

		// For now, we do not use the header row to remap the ordering of the columns.
		// To Do: Use header row to remap the indices to fit the Meter constructor
		return row.map(val => val === '' ? undefined : val);
	});
```

This is the same meters object but if it contains the header row it removes it because it can't handle it at the current version. The rest of this is just comments from past commits.
```Js
// If there is a header row, we remove and ignore it for now.
	const meters = normalizeBoolean(req.body.headerRow) ? temp.slice(1) : temp;
	// The original code used a Promise.all to run through the meters. The issue is that the promises are run in parallel.
	// If the meters are independent as expected then this works fine. However, in the error case where one CSV file has
	// the same meter name listed twice, the order of the attempts to add to the database was arbitrary. This meant one of them
	// failed due to the duplicate name but you did not know which one. If some of the information on the two meters differed then
	// you did not know which one you would get in the database. The best result would be the first one in the CSV file would be stored
	// as this makes the most logical sense (no update here) and it is consistent. To make this happen a for loop is used as it
	// is sequential. A small negative is the database requests do not run in parallel in the usual case without an error.
	// However, uploading meters is not common so slowing it down slightly seems a reasonable price to get this behavior.
```

Here's the bulk of the code:

We will be further breaking this down into easy-to read and interpret, chunks.
```Js
try {
		for (let i = 0; i < meters.length; i++) {
			let meter = meters[i];
			// First verify GPS is okay
			// This assumes that the sixth column is the GPS as order is assumed for now in a GPS file.
			const gpsInput = meter[6];
			// Skip if undefined.
			if (gpsInput) {
				// Verify GPS is okay values
				const { validGps, message } = isValidGPSInput(gpsInput);
				if (!validGps) {
					let msg = `For meter ${meter[0]} the gps coordinates of ${gpsInput} are invalid with error of "${message}"`;
					throw new CSVPipelineError(msg, undefined, 500);
				}
				// Need to reverse latitude & longitude because standard GPS gives in that order but a GPSPoint for the
				// DB is longitude, latitude.
				meter[6] = switchGPS(gpsInput);
			}

			// Process unit.
			const unitName = meter[23];
			const unitId = await getUnitId(unitName, Unit.unitType.METER, conn);
			if (!unitId) {
				const msg = `For meter ${meter[0]} the unit of ${unitName} is invalid`;
				throw new CSVPipelineError(msg, undefined, 500);
			}
			// Replace the unit's name by its id.
			meter[23] = unitId;

			// Process default graphic unit.
			const defaultGraphicUnitName = meter[24];
			const defaultGraphicUnitId = await getUnitId(defaultGraphicUnitName, Unit.unitType.UNIT, conn);
			if (!defaultGraphicUnitId) {
				const msg = `For meter ${meter[0]} the default graphic unit of ${defaultGraphicUnitName} is invalid`;
				throw new CSVPipelineError(msg, undefined, 500);
			}
			// Replace the default graphic unit's name by its id.
			meter[24] = defaultGraphicUnitId;

			if (normalizeBoolean(req.body.update)) {
				// Updating the new meters.
				// First get its id.
				let identifierOfMeter = req.body.meterIdentifier;
				if (!identifierOfMeter) {
					// Seems no identifier provided so use one in CSV file.
					if (!meter[7]) {
						// There is no identifier given for meter in CSV so use name as identifier since would be automatically set.
						identifierOfMeter = meter[0];
					} else {
						identifierOfMeter = meter[7];
					}
				} else if (meters.length !== 1) {
					// This error could be thrown a number of times, one per meter in CSV, but should only see one of them.
					throw new CSVPipelineError(`Meter identifier provided (\"${identifierOfMeter}\") in request with update for meters but more than one meter in CSV so not processing`, undefined, 500);
				}
				let currentMeter;
				currentMeter = await Meter.getByIdentifier(identifierOfMeter, conn)
					.catch(error => {
						// Did not find the meter.
						let msg = `Meter identifier of \"${identifierOfMeter}\" does not seem to exist with update for meters and got DB error of: ${error.message}`;
						throw new CSVPipelineError(msg, undefined, 500);
					});
				currentMeter.merge(...meter);
				await currentMeter.update(conn);
			} else {
				// Inserting the new meter
				await new Meter(undefined, ...meter).insert(conn)
					.catch(error => {
						// Probably duplicate meter.
						throw new CSVPipelineError(
							`Meter name of \"${meter[0]}\" got database error of: ${error.message}`, undefined, 500);
					}
					);
			}
		}
	} catch (error) {
		throw new CSVPipelineError(`Failed to upload meters due to internal OED Error: ${error.message}`, undefined, 500);
	}
```

That for loop stretches the entirety of the try statement.

Each piece of code from now until the catch statement will be inside of that beefy for loop. For now though, let's read the for loop and figure out what it's looping through.

Alright, this is pretty simple. Remember the meters object? Well we're just iterating through it (how it's doing that I'm not too sure, I'd like to see what meters contains before I make a statement on that). 
`for (let i = 0; i < meters.length; i++)`


As for this beefy chunk of code:

I know more about the meters object. Each row (let's call them rows for lack of a better word), is a meter. The for loop allows us to access a single meter each iteration and the programmer defined (rightly so) each meter as `meter`.

Since meters is stored as a map, it makes sense that we can index it twice to access the individual meter and then the data points for each of the meters. So that's what `gpsInput` is doing. 

The if statement will be skipped if gpsInput is undefined, however for all other values it will run (maybe not null though). In this check we're verifying if the gps value is valid, using the `isValidGPSInput()` method. If it's not a valid gps, it will output an error message giving the meter name `meter[n]` and the message generated from `isValidGPSInput()`, it will also throw an error from the CSVPipelineError method.

The database is ordering the gps information weird (might be something to refactor in another pr), so latitude & longitude need to be reversed. That's accomplished with `switchGPS(gpsInput)`.

```Js
let meter = meters[i];
// First verify GPS is okay
// This assumes that the sixth column is the GPS as order is assumed for now in a GPS file.
const gpsInput = meter[6];
// Skip if undefined.
if (gpsInput) {
    // Verify GPS is okay values
    const { validGps, message } = isValidGPSInput(gpsInput);
    if (!validGps) {
        let msg = `For meter ${meter[0]} the gps coordinates of ${gpsInput} are invalid with error of "${message}"`;
        throw new CSVPipelineError(msg, undefined, 500);
    }
    // Need to reverse latitude & longitude because standard GPS gives in that order but a GPSPoint for the
    // DB is longitude, latitude.
    meter[6] = switchGPS(gpsInput);
}
```

Here's the next section of the code. It appears we're accessing the unitName (this is where the custom energy/unit values are set). We have to get the unitId though because all the unit types are named/id'd in OED. That's where the `getUnitId()` method comes from. We can tell that it's waiting for the unit id though, due to the `await` keyword. It will pause the surrounding `async` method until it gets a response. 

If the unitId is a badValue or doesn't exist then an error message is printed (maybe to the console, I don't know right now, would have to investigate `CSVPipelineError()`) and thrown.

The unit's name is then replaced by its id in the map.

The `graphicUnitName` is probably referring to what's displayed on the front-end, however we're also using `getUnitId()` for it. So I'm going to go with- I don't know what `defaultGraphicUnitName` is and leave it at that (for now). However if that has a bad value then it throws an error. However if it has a good value, then it will replace its respective part of the map with the unit id.
```Js
// Process unit.
const unitName = meter[23];
const unitId = await getUnitId(unitName, Unit.unitType.METER, conn);
if (!unitId) {
    const msg = `For meter ${meter[0]} the unit of ${unitName} is invalid`;
    throw new CSVPipelineError(msg, undefined, 500);
}
// Replace the unit's name by its id.
meter[23] = unitId;

// Process default graphic unit.
const defaultGraphicUnitName = meter[24];
const defaultGraphicUnitId = await getUnitId(defaultGraphicUnitName, Unit.unitType.UNIT, conn);
if (!defaultGraphicUnitId) {
    const msg = `For meter ${meter[0]} the default graphic unit of ${defaultGraphicUnitName} is invalid`;
    throw new CSVPipelineError(msg, undefined, 500);
}
// Replace the default graphic unit's name by its id.
meter[24] = defaultGraphicUnitId;
```

Let's move on to the next section. This is the next day and after a conversation with a good friend who is a more experienced developer than I. He has shed some light on some of the helper functions that are being called, which I will get to after I finish analyzing this `for` loop.

I decided to take most of the rest of the for loop leaving out only the catch statement. Let's start: This if statement is large, with several nested statements inside. Given that, I'm wondering if it would be *possible* to refactor this logic so there's only one `if` statement. That would make the code significantly more readable.

In this section of the code it appears we're calling a function `normalizeBoolean(<arg>)` which returns a boolean value. That value is checked by the `if statment`. The argument that's passed is `req.body.update`, I'm going to go out on a limb here and say that it requires the body to update. Now, what that means... I don't know. No clue. Hope to figure it out someday, but what I can say is that it appears to tie into the whole async/await part of the code (therefore it's tying into the multithreaded section). Maybe we can treat this like await, because it requires another section of the code to update for it to evaluate as true. One thing I can say for certain `normalizeBoolean()` is not a member function of this file.

`identifierOfMeter` is a member variable who's value appears to use the same package notation that `normalizeBoolean()` is requesting. This part of the code appears to be updating new meters. I'm not sure if touching this part of the code would be detrimental to the codebase right now, so I think I'll try to rewrite this later. If that random information is false (I don't know what that means either), then it checks if `meter[7]` has a value/exists. If there is no identifier given for that specific meter, use `identifierOfMeter` as the name. The one thing that I don't understand about this section of code is that if `meter[7]` doesn't exist, then `identifierOfMeter` will be set to `meter[0]`. If it does exist then `identifierOfMeter` will be set to `meter[7]`. I guess I would need to understand the type of data that `identifierOfMeter` is storing, and the type of data that `meter[0]` & `meter[7]` are storing.

Wait, I just realized something:
### Note to Kyle for Later ###
**Come back and double check that this logic works to replace the current logic in the code**
```Js
// We can rewrite this check like this:
if (normalizeBoolean(req.body.update)) {
	if (!identifierOfMeter && !meter[7]) {
		identifierOfMeter = meter[0];
	} else {
		identifierOfMeter = meter[7];
	}
	// I couldn't completely eliminate the second if statement but I believe that I simplified the logic
}
```

Moving on, if `identifierOfMeter` does exist:
Then we check if `meters.length !== 1`. I've always hated this sort of check, especially if it's not necessary. Let me explain why:
1. It's ambiguous. I don't know what this code is saying. Is it saying that `meters.length` has to `== 1` or the check fails? That may be exactly what it's saying but if it's anything else I believe that it should be providing a range of numbers to exclude.
2. There's not another reason but I'm going to leave this in here to prove my point. It's annoying, hard to read, especially if the comments aren't necessarily clear, such as in this case.
If `meters.length !== 1` then an error is thrown. I'm not sure why, or what type of error but if I end up refactoring this I'll get to the bottom of it.

Past that, we initialize a variable `currentMeter` with (and I can't stress this enough):
```Js
let currentMeter; // one line
currentMeter = // this is an anonymous inner class (I believe) but still, you could do this in one line. What's the point of 2??
```
`currentMeter`'s anonymous inner class awaits the return of `Meter.getByIdentifier()` which is not a function of the file. If the meter is not found, it returns an error. This code actually makes sense from what I'm reading. It also seems to be well written. No complaints from me. 
After that we can see that `currentMeter` is `.merge`d with any number of `meter` arguments passed. Finally, `await currentMeter.update(conn)` is called which I understand except for the `conn` argument. I'm assuming it's some sort of connector, and we'll leave it at that.

Actually finally, we reach the else part of our if statement. This part of the code inserts the new meter by creating a `new` `Meter` object, passing in `undefined, ...meter`. Since we're passing an undefined value I sure hope it's being handled somewhere else in the function, namely where `Meter` is created. If any error occurs we `catch` the error and throw it.

```Js
if (normalizeBoolean(req.body.update)) {
	// Updating the new meters.
	// First get its id.
	let identifierOfMeter = req.body.meterIdentifier;
	if (!identifierOfMeter) {
		// Seems no identifier provided so use one in CSV file.
		if (!meter[7]) {
			// There is no identifier given for meter in CSV so use name as identifier since would be automatically set.
			identifierOfMeter = meter[0];
		} else {
			identifierOfMeter = meter[7];
		}
	} else if (meters.length !== 1) {
		// This error could be thrown a number of times, one per meter in CSV, but should only see one of them.
		throw new CSVPipelineError(`Meter identifier provided (\"${identifierOfMeter}\") in request with update for meters but more than one meter in CSV so not processing`, undefined, 500);
	}
	let currentMeter;
	currentMeter = await Meter.getByIdentifier(identifierOfMeter, conn)
		.catch(error => {
			// Did not find the meter.
			let msg = `Meter identifier of \"${identifierOfMeter}\" does not seem to exist with update for meters and got DB error of: ${error.message}`;
			throw new CSVPipelineError(msg, undefined, 500);
		});
	currentMeter.merge(...meter);
	await currentMeter.update(conn);
} else {
// Inserting the new meter
await new Meter(undefined, ...meter).insert(conn)
	.catch(error => {
		// Probably duplicate meter.
		throw new CSVPipelineError(
			`Meter name of \"${meter[0]}\" got database error of: ${error.message}`, undefined, 500);
	}
	);
}
```

Wait, I lied, now we're on to the catch part. All this does is catch any error that hasn't already been thrown and refer to `internal server error` which is a fancy way of saying "We don't know what went wrong, please try again later!"

```Js
catch (error) {
	throw new CSVPipelineError(`Failed to upload meters due to internal OED Error: ${error.message}`, undefined, 500);
}
```

Now, finally on to the internal methods. This is something that my friend Jake pointed out to me that I'm not sure I would've seen on my own:
First off, I'm finally seeing some javadoc explaining what the **** this **** is doing. I realize I'm being dramatic here but it's been hard reading the code to this point, I could use some javadoc.

This function, like it says, checks for `ValidGPSInput`:
It seems to be written decently well too.
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

This code switches the longitude and latitude coordinates like what's required for inputting the data to the database. (Again, might be something nice to fix).
```Js
function switchGPS(gpsString) {
	const array = gpsString.split(',');
	return (array[1] + ',' + array[0]);
}
```
You could also write it like this as I do believe there are no spaces between the commas and the coordinates:
```Js
function switchGPS(gpsString) {
	const array = gpsString.split(',');
	return (`${array[1]},${array[0]}`);
}
```

The final internal function is this:
It takes the `unitName`, the `expectedUnitType`, and a connector (`conn`). Now I feel like this function could be written better but I'm not sure how to do so at the moment. Maybe this is yet another thing to get back to.

Anyway, this function takes those parameters, and returns a constant negative integer if `unitName` doesn't exist. I'm sure that could be improved in some dynamic manner.
A `const` variable `unit` is set to `await Unit.getByName(unitName, conn)`. If the unit doesn't exist after that then `null` is returned. If it does, return the unit's id.

```Js
async function getUnitId(unitName, expectedUnitType, conn) {
	// Case no unit.
	if (!unitName) return -99;
	// Get the unit associated with the name.
	const unit = await Unit.getByName(unitName, conn);
	// Return null if the unit doesn't exist or its type is different from expectation.
	if (!unit || unit.typeOfUnit !== expectedUnitType) return null;
	return unit.id;
}
```

Finally we have this crucial line of code which I assume just ties this file to the rest of the codebase:
```Js
module.exports = uploadMeters;
```

And that's it. That's the end of the file. Hope this helped anyone who decided to read this.