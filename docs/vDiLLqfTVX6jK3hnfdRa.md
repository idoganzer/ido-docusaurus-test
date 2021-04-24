---
title: Debugging AutoSync
---

<div align="center">    <br/>    <div>DOC</div>    <h1>Debugging AutoSync</h1>    <br/>  </div>

### Files Used:
📄 src/swimmagic/applicability.spec.ts

📄 src/swimmagic/autofix.spec.ts


<br/>

So let's say you get one of these tickets to tackle - where `verify` says the Unit is `verified`, but the UI says all snippets are `autosyncable`. Or vice versa. Or any other weird issue coming up with `AutoSync`. What do you do?

This document includes some tips and best practices.

# Prerequisites.
* A deep understanding of AutoSync's main flow (**TBD - link**).

# General things
* Go through `swimm log` and see if you get helpful hints.

# How to investigate?
A good start is to ask yourself what you would expect.

Open the `.swm` file, look at the actual snippets. 

Then, look at their current state in the code.

Also, understand AutoSync's starting point (**TBD - link**). Does it start from a specific commit deduced from the Blob Sha? The `swimm log` should help with that.



<br/>

# Applicability


<br/>

`validateApplicabilityStatuses` checks the applicability of the entire patch as well as hunk by hunk. It may give hints to see the different results.
Running an applicability test that calls this function can be helpful.

<div>    📄 src/swimmagic/applicability.spec.ts  </div>

```ts
⬜ 40      * Validates units applicability statuses
⬜ 41      * @param ignoredSwmsList - ids of invalid/inapplicable swms that should be ignored when validating the applicabilities
⬜ 42      */
🟩 43     async function validateApplicabilityStatuses({ ignoredSwmsList }: { ignoredSwmsList: string[] }) {
🟩 44       const swmsList = await getListOfSwmFilesInRepo();
🟩 45       for (const swm of swmsList) {
🟩 46         if (ignoredSwmsList.includes(swm)) {
🟩 47           continue;
🟩 48         }
🟩 49         const autoFixResult = await autofix.autoFixUnit(swm);
🟩 50         const applicabilityResults = await checkApplicabilityForSwmFile(autoFixResult.autoFixedSwmFile);
🟩 51         try {
🟩 52           expect(applicabilityResults.hunkByHunkApplicability).toEqual(applicabilityResults.gitDiffApplicability);
🟩 53         } catch (e) {
🟩 54           throw `${swm} isAutofixed: ${autoFixResult.autoFixSuccess}; hunkByHunkApplicability:${applicabilityResults.hunkByHunkApplicability},gitDiffApplicability:${applicabilityResults.gitDiffApplicability} `;
🟩 55         }
🟩 56       }
🟩 57     }
⬜ 58     
⬜ 59     async function checkApplicabilityForSwmFile(swmFile: SwmFile) {
⬜ 60       const gitDiffApplicability = isSwmApplicable(swmFile);
```
<br/>

# How to debug?

So it's time to dig in, let's run autosync's main flow. For that, you can take one of the existing tests and tweak it.	

<br/>

For example here - we run the main flow for a specific Unit.


<div>    📄 src/swimmagic/autofix.spec.ts  </div>

```ts
⬜ 186        });
⬜ 187    
⬜ 188        describe('Update context', () => {
🟩 189          test('Update upper context and move down a few lines -> updates accordingly', async () => {
🟩 190            state.set('cwd', currentRepoCwd);
🟩 191            const UPPER_CONTEXT_UNIT_ID = 'ml5rbhkct0pUSBMLTAhc';
🟩 192            const result = await autofix.autoFixUnit(UPPER_CONTEXT_UNIT_ID);
🟩 193            expect(result.autoFixSuccess).toBeTruthy();
🟩 194            const autoFixedUnit = result.autoFixedSwmFile;
⬜ 195            const correctCell: SwmCellSnippet = autoFixedUnit.content.filter((cell): boolean => cell.type === 'snippet')[0] as SwmCellSnippet;
⬜ 196            expect(correctCell.firstLineNumber).toBe(3);
⬜ 197            expect(correctCell.path).toBe('autoFixTests/updateContext/update_upper_context_only.py');
```
<br/>

Notice that we set the `cwd` to the relevant repo. So there's no need to copy the Unit to the staging database.

<div>    📄 src/swimmagic/autofix.spec.ts  </div>

```ts
⬜ 187    
⬜ 188        describe('Update context', () => {
⬜ 189          test('Update upper context and move down a few lines -> updates accordingly', async () => {
🟩 190            state.set('cwd', currentRepoCwd);
⬜ 191            const UPPER_CONTEXT_UNIT_ID = 'ml5rbhkct0pUSBMLTAhc';
⬜ 192            const result = await autofix.autoFixUnit(UPPER_CONTEXT_UNIT_ID);
⬜ 193            expect(result.autoFixSuccess).toBeTruthy();
```
<br/>

From here - you can debug the actual flow - use breakpoints and so on 😎

<br/>

Debugging AutoSync is challenging, but hopefully this document will help you.

Have tips or best practices to share? Please add them here!

<br/>

<br/><br/>

This file was generated by Swimm. [Click here to view it in the app](https://swimm.io/link?l=c3dpbW0lM0ElMkYlMkZyZXBvcyUyRnZlZXp2eEN1enBQclJMTFhXRDJFJTJGZG9jcyUyRnZEaUxMcWZUVlg2akszaG5mZFJh). Timestamp: 2021-04-24T18:52:02.379Z