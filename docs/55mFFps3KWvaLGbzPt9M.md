---
title: Upgrade Files
---

<div align="center">    <br/>    <div>DOC</div>    <h1>Upgrade Files</h1>    <br/>  </div>

### Files Used:
📄 src/cli/commands/upgrade-files.ts

📄 src/swimmagic/autofix.ts

📄 src/shared/common.ts


<br/>

In this Swimm we will talk about the `upgrade-files` command and how we deal with backwards computability when updating the `swm` file structure.

<br/>

The command is a maintenance command that was created while making a major change in the `swm` files structures (and in autosync). In order to overcome performance issue related to the SWM changes, we wanted to have a way to update the files in a dedicated operation that will run once and update the files for the users. The other way to update a file is for to edit and save the unit, for each unit in the repo.

<br/>

The command can be run for a specific unitId or for all `swm` in the repository

<div>    📄 src/cli/commands/upgrade-files.ts  </div>

```ts
⬜ 12     import { gitAddFile } from 'Shared/gitwrapper';
⬜ 13     import logger from 'Shared/logger';
⬜ 14     
🟩 15     export async function upgradeFiles(unitId: string, all: boolean) {
⬜ 16       if (!unitId && !all) {
⬜ 17         pprint.err(pprint.styles.error('Please enter the Swimm file Id you would like to upgrade or use -a to upgrade all Swimm files in this repo.'));
⬜ 18         return config.ERROR_RETURN_CODE;
```
<br/>

before starting the upgrade we verify that the units are applicable. this is important because we don't want to save an outdated version of the files.

<div>    📄 src/cli/commands/upgrade-files.ts  </div>

```ts
⬜ 33       if (all) {
⬜ 34         // verify all units are applicable
⬜ 35         pprint.out(pprint.styles.info(`Verifying units before attempting to upgrade...\n`));
🟩 36         const verifyResult = await cliUtils.verifyUnitsInRepo({ unitId: unitId, isAutofixable: true });
⬜ 37         if (!verifyResult) {
⬜ 38           pprint.err(pprint.styles.error('Some of the units are not up to date, please update or sync them before running upgrade-files.'));
⬜ 39           return config.ERROR_RETURN_CODE;
```
<br/>

## The upgrade

After passing all the validation steps, we preform the update for the  a specific unit (if the command was provided with a unit ID) or to all the units in the repo if provided with the `-a` flag. 

<br/>



<div>    📄 src/cli/commands/upgrade-files.ts  </div>

```ts
⬜ 58       for (const unitId of unitsToUpgrade) {
⬜ 59         try {
⬜ 60           spinner.start(` * ${unitId}`);
🟩 61           await upgradeUnitFile(unitId);
⬜ 62           spinner.succeed();
⬜ 63         } catch (err) {
⬜ 64           logger.error(`Upgrading Swimm file ${unitId} failed. Details: ${err.toString()} `, { service: 'cli-upgrade-files' });
```
<br/>

We `autosync` each unit - the autosync call returns us the unit at it's latest format. We needed to call it to make sure that the snippets are aligned with the latest state of the repo.
At this point we know the snippets are applicable, and autosync just fix the lines numbers and contexts.

<div>    📄 src/cli/commands/upgrade-files.ts  </div>

```ts
⬜ 73     
⬜ 74     async function upgradeUnitFile(unitId) {
⬜ 75       const swmFilePath = utils.pathInSwmFolder(`${unitId}${config.SWM_FILE_EXTENSION}`);
🟩 76       // autoFixUnit load and convert unit to the new file structure
🟩 77       // we autosyncing the applicable unit (patch applicable) to make sure the lines and context are correct on save
🟩 78       const autosyncResult = await autoFixUnit(unitId);
⬜ 79       if (!autosyncResult.autoFixSuccess) {
⬜ 80         throw new Error(`There was an issue while trying to autosync Swimm file: ${unitId}`);
⬜ 81       }
```
<br/>

After a successful autosync we can update the files blob_sha (meta) with the current (HEAD) SHAs.

<div>    📄 src/cli/commands/upgrade-files.ts  </div>

```ts
⬜ 86     
⬜ 87       const convertedAndAutosyncedUnit = autosyncResult.autoFixedSwmFile;
⬜ 88       try {
🟩 89         // update file blobs - current HEAD
🟩 90         updateBlobShas({ swmFile: convertedAndAutosyncedUnit, meta: meta });
⬜ 91     
⬜ 92         convertedAndAutosyncedUnit.meta = { ...meta };
⬜ 93     
```
<br/>

## Load And Convert
In addition to updating the snippets, line numbers and blobs sha, one of the major thing that is happening is converting old version of `swm` files to the latest structure.
The `loadAndConvertSwmFile` method is called as part of the autosync unit call but it being called when ever we loading a swm file (in the cli and in the app).

<br/>



<div>    📄 src/swimmagic/autofix.ts  </div>

```ts
⬜ 28      * @return {Promise<boolean>} - Was the unit fixing process successful
⬜ 29      */
⬜ 30     export async function autoFixUnit(unitId: string): Promise<{ autoFixSuccess: boolean; autoFixedSwmFile?: SwmFile }> {
🟩 31       const swmFile = await loadAndConvertSwmFile(pathInSwmFolder(`${unitId}${config.SWM_FILE_EXTENSION}`));
⬜ 32     
⬜ 33       try {
⬜ 34         const autoFixResult = autoFixUnitMainFlow({ destCommit: 'HEAD', unit: swmFile });
```
<br/>

We first loading the swm JSON and then do the conversion to the latest structure 

<div>    📄 src/shared/common.ts  </div>

```ts
⬜ 109      return decodedSwmFileContent;
⬜ 110    }
⬜ 111    
🟩 112    export async function loadAndConvertSwmFile(swmPath: string): Promise<SwmFile> {
🟩 113      const loadedSwm = await utils.loadSwmFile(swmPath);
🟩 114      return convertSWMStructure(loadedSwm);
🟩 115    }
⬜ 116    
⬜ 117    /**
⬜ 118     * Convert an old dumb-doc into the new SwmFile structure.
```
<br/>

The first part of the convert is update very old file to version 1.0.4.

<div>    📄 src/shared/common.ts  </div>

```ts
⬜ 156    }
⬜ 157    
⬜ 158    // For backward compatibility, if needed - convert diff from an old SWM file without swimmPatch to dynamicSwimmPatch structure.
🟩 159    export function convertSWMStructure(unitFile) {
🟩 160      let convertedSWMFile = { ...unitFile };
🟩 161      if (utils.assertFileVersion({ fileVersion: unitFile.file_version, operator: '<', versionToCompare: '2.0.0' })) {
🟩 162        // Up to 1.0.4 inclusive - the conversion process relied on this to happen
🟩 163        // There is no 1.0.5 - after 1.0.4 we had 2.0.0
🟩 164        convertedSWMFile = preV2prepareLoadedSwmFile(unitFile);
🟩 165    
🟩 166        if (utils.assertFileVersion({ fileVersion: unitFile.file_version, operator: '<', versionToCompare: '1.0.3' })) {
🟩 167          convertedSWMFile = upgradeSWMFileStructureFrom102OrBelowTo104(convertedSWMFile);
🟩 168        } else {
🟩 169          if (utils.assertFileVersion({ fileVersion: unitFile.file_version, operator: '<', versionToCompare: '1.0.4' })) {
🟩 170            convertedSWMFile.hunksOrder = generateHunksOrderStaticSwimmPatchSwmFile104(convertedSWMFile.swimmPatch);
🟩 171          }
🟩 172        }
⬜ 173        // Now convert from 1.0.4 to 2.0.0
⬜ 174        try {
⬜ 175          convertedSWMFile = upgradeSWMFileFrom104to200(convertedSWMFile, unitFile.file_version);
```
<br/>

And then converting to the latest version.

<div>    📄 src/shared/common.ts  </div>

```ts
⬜ 172        }
⬜ 173        // Now convert from 1.0.4 to 2.0.0
⬜ 174        try {
🟩 175          convertedSWMFile = upgradeSWMFileFrom104to200(convertedSWMFile, unitFile.file_version);
⬜ 176        } catch (err) {
⬜ 177          throw new Error(`could not convert unit to file vesion 2.0.0, Details: ${err.toString()}`);
⬜ 178        }
```
<br/>

Here are few of the changes to the `swm`

<br/>

set up a `task` entry for unit with hands-on task

<div>    📄 src/shared/common.ts  </div>

```ts
⬜ 196        task: {},
⬜ 197      };
⬜ 198    
🟩 199      // populate task fields
🟩 200      const task: SwmTask = {
🟩 201        dod: swmFileInVersion104.dod,
🟩 202        hints: swmFileInVersion104.hints,
🟩 203        tests: swmFileInVersion104.tests,
🟩 204      };
🟩 205      if (swmFileInVersion104.cr) {
🟩 206        task.crActions = swmFileInVersion104.cr;
🟩 207      }
🟩 208      newStructure.task = task;
⬜ 209    
⬜ 210      // populate file_blobs
⬜ 211      for (const file of Object.keys(swmFileInVersion104.swimmPatch)) {
```
<br/>

move from swimmPatch structure - where hunks where under the files name and the order was saved separately. 
to an array of content with more readable structure where the order is clear and allow a mix of snippets and text blocks (and more).

<div>    📄 src/shared/common.ts  </div>

```ts
⬜ 211      for (const file of Object.keys(swmFileInVersion104.swimmPatch)) {
⬜ 212        newStructure.meta.file_blobs[file] = '';
⬜ 213      }
🟩 214      // populate content
🟩 215      // first cell is a text that include the intro
🟩 216      newStructure.content.push({ type: 'text', text: swmFileInVersion104.description });
🟩 217      // addind snippets based on the order
🟩 218    
🟩 219      for (const hunkOrderId of swmFileInVersion104.hunksOrder) {
🟩 220        const hunkOrderIdSplitIndex = hunkOrderId.lastIndexOf('_');
🟩 221        const hunkFile = hunkOrderId.substring(0, hunkOrderIdSplitIndex);
🟩 222        const hunkIndexInFile = hunkOrderId.substring(hunkOrderIdSplitIndex + 1, hunkOrderId.length);
🟩 223        const swimmFilePatch = swmFileInVersion104.swimmPatch[hunkFile];
🟩 224        const patchType = dynamicFileDiffTypeToSwmFilePatchType(swimmFilePatch.diffType);
🟩 225        let snippetCell = {};
🟩 226        if (utils.assertFileVersion({ fileVersion: orignalFileVersion, operator: '<', versionToCompare: '1.0.3' })) {
🟩 227          const hunkToAdd = swimmFilePatch.hunkContainers[hunkIndexInFile];
🟩 228          snippetCell = dynamicHunkContainerToSwmCellSnippet(hunkToAdd, hunkFile, patchType);
🟩 229          if (hunkToAdd.swimmHunkMetadata && hunkToAdd.swimmHunkMetadata.hunkComments) {
🟩 230            snippetCell['comments'] = hunkToAdd.swimmHunkMetadata.hunkComments;
🟩 231          }
🟩 232        } else {
🟩 233          // version 1.0.4
🟩 234          snippetCell = ConvertOldHunkToNewHunkFormat({ hunk: swimmFilePatch.hunks[hunkIndexInFile], fileName: hunkFile, patchType: patchType });
🟩 235        }
🟩 236    
🟩 237        newStructure.content.push(snippetCell);
⬜ 238      }
⬜ 239    
⬜ 240      // add summary to content
```
<br/>

To summarize this unit, it is important to know how we support older versions of files and understand the flow that happens when we load a `swm` file, and how we can leverage the `upgrade-files` command to update the files and save a bit on performance - especially autosyncing the snippets and updating the blobs SHA.

<br/>

<br/><br/>

This file was generated by Swimm. [Click here to view it in the app](https://swimm.io/link?l=c3dpbW0lM0ElMkYlMkZyZXBvcyUyRnZlZXp2eEN1enBQclJMTFhXRDJFJTJGZG9jcyUyRjU1bUZGcHMzS1d2YUxHYnpQdDlN). Timestamp: 2021-04-24T18:52:02.046Z
