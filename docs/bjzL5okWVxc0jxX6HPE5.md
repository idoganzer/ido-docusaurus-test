---
title: Creating a frontend unit test
---

<div align="center">    <br/>    <div>DOC</div>    <h1>Creating a frontend unit test</h1>    <br/>  </div>

### Files Used:
📄 src/app/common/mixins/unitForm.js

📄 src/app/tests/common/mixins/unitForm.spec.js


<br/>

As you already know, tests are important. In the FE we don't want to test the actual rendering because it changes very frequently, but we do want to test the logic behind it.

In this walkthrough you will learn the steps for creating a unit test for a function in a FE component.
In our case we will add a test for the `deleteHunk` function in the `unitForm` mixin. 
the function should remove a hunk snippet by an index in a specific file.


<br/>

This is the function that we will test. It's here just as a reference

<div>    📄 src/app/common/mixins/unitForm.js  </div>

```js
🟩 216        deleteCell(cellIndex) {
🟩 217          // keep original data for recover
🟩 218          const deletedCell = this.swm.content.splice(cellIndex, 1)[0];
🟩 219          this.addSnippetIndex = cellIndex;
🟩 220          this.logUpdateHunkChanges({ action: 'delete', applicability: deletedCell.applicability });
🟩 221          this.$toasted.show(`${deletedCell.type === 'snippet' ? 'Snippet' : 'Text Block'} deleted`, {
🟩 222            theme: 'toasted-primary',
🟩 223            position: 'top-right',
🟩 224            duration: 4000,
🟩 225            action: {
🟩 226              text: 'Cancel',
🟩 227              onClick: (e, toastObject) => {
🟩 228                toastObject.goAway(0);
🟩 229                this.addCellInSpecificIndex(deletedCell, cellIndex);
🟩 230              },
🟩 231            },
🟩 232          });
🟩 233          // if the last cell was snippet and it was deleted - add an empty text block
🟩 234          // can be true only in docs
🟩 235          if (this.swm.content.length === 0) {
🟩 236            this.swm.content.push({ type: 'text', text: '' });
🟩 237          }
🟩 238        },
```
<br/>

In this step we declare what we are going to test.

<div>    📄 src/app/tests/common/mixins/unitForm.spec.js  </div>

```js
⬜ 13     
⬜ 14     describe('unitForm', () => {
⬜ 15       describe('deleteCell', () => {
🟩 16         it('should remove a snippet from the content', async () => {
⬜ 17           const route = {
⬜ 18             params: {
⬜ 19               unitId: 123,
```
<br/>

First we declare the current app route with relevant params needed for initializing our component and test. 

<div>    📄 src/app/tests/common/mixins/unitForm.spec.js  </div>

```js
⬜ 14     describe('unitForm', () => {
⬜ 15       describe('deleteCell', () => {
⬜ 16         it('should remove a snippet from the content', async () => {
🟩 17           const route = {
🟩 18             params: {
🟩 19               unitId: 123,
🟩 20               repoId: 456,
🟩 21             },
🟩 22             query: {},
🟩 23           };
⬜ 24     
⬜ 25           let wrapper = shallowMount(unitForm, { mocks: { $route: route, $toasted: { show: () => true } } });
⬜ 26     
```
<br/>

In this step we mount the component (`unitForm`) and we set the component data required for the test.
Note that we had to mock the `$toasted` component for the component to successfully mount.

<div>    📄 src/app/tests/common/mixins/unitForm.spec.js  </div>

```js
⬜ 22             query: {},
⬜ 23           };
⬜ 24     
🟩 25           let wrapper = shallowMount(unitForm, { mocks: { $route: route, $toasted: { show: () => true } } });
🟩 26     
🟩 27           await wrapper.setData({
🟩 28             swm: {
🟩 29               content: [
🟩 30                 {
🟩 31                   path: '0.js',
🟩 32                 },
🟩 33                 {
🟩 34                   path: '1.js',
🟩 35                 },
🟩 36                 {
🟩 37                   path: '2.js',
🟩 38                 },
🟩 39                 {
🟩 40                   path: '3.js',
🟩 41                 },
🟩 42               ],
🟩 43             },
🟩 44           });
⬜ 45     
⬜ 46           wrapper.vm.deleteCell(2); // removing 2.js
⬜ 47     
```
<br/>

In this step we call the function that we wish to test. 
Note that we call `wrapper.vm` and not just `wrapper`

<div>    📄 src/app/tests/common/mixins/unitForm.spec.js  </div>

```js
⬜ 43             },
⬜ 44           });
⬜ 45     
🟩 46           wrapper.vm.deleteCell(2); // removing 2.js
⬜ 47     
⬜ 48           expect(wrapper.vm.swm.content.length).toBe(3);
⬜ 49           expect(wrapper.vm.swm.content[2].path).toBe('3.js');
```
<br/>

Finally we can call the  `expect` functions to verify that the data changed as expected.

<div>    📄 src/app/tests/common/mixins/unitForm.spec.js  </div>

```js
⬜ 45     
⬜ 46           wrapper.vm.deleteCell(2); // removing 2.js
⬜ 47     
🟩 48           expect(wrapper.vm.swm.content.length).toBe(3);
🟩 49           expect(wrapper.vm.swm.content[2].path).toBe('3.js');
⬜ 50         });
⬜ 51         it('when deleting the last cell, an empty text cell should be added', async () => {
⬜ 52           const route = {
```
<br/>

In this walkthrough we verified that the function works correctly by verifying the component data but you can also check the component props data or expect mocks calls (e.g calling an adapter) 

<br/>

<br/><br/>

This file was generated by Swimm. [Click here to view it in the app](https://swimm.io/link?l=c3dpbW0lM0ElMkYlMkZyZXBvcyUyRnZlZXp2eEN1enBQclJMTFhXRDJFJTJGZG9jcyUyRmJqekw1b2tXVnhjMGp4WDZIUEU1). Timestamp: 2021-04-24T18:52:02.226Z
