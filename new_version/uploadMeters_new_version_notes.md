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

