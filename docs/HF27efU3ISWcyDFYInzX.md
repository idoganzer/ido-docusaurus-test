---
title: Firestore Whitelisted Fields
---

<div align="center">    <br/>    <div>DOC</div>    <h1>Firestore Whitelisted Fields</h1>    <br/>  </div>

### Files Used:
📄 cloud/firestore.rules

📄 cloud/firestore_rules.spec.js


<br/>

When updating a document in Firestore we want to make sure we allow to update only relevant and/or specific fields

To do so, we're using [Firestore security rules](https://swimm.io/link?l=c3dpbW0lM0ElMkYlMkZ3b3Jrc3BhY2VzJTJGYVJ2TXFjMHlXQVZjSmxMTjk0NEQlMkZyZXBvcyUyRnZlZXp2eEN1enBQclJMTFhXRDJFJTJGdW5pdHMlMkZQRFVlRGtoS1NGQWRRS1VkTUxUag==) in order to make sure DB access is secure

<br/>

Whitelist fields example - repository
-------------------------------------

<br/>

adding `validateAllowedRepositoryUpdateFields()` and validateAllowedRepositoryCreateFields() to our allow create and update conditions

<div>    📄 cloud/firestore.rules  </div>

```rules
⬜ 123    
⬜ 124        match /repositories/{repository} {
🟩 125          allow create: if validateAllowedRepositoryCreateFields();
🟩 126          allow update: if isUserLifeguard(repository) && validateAllowedRepositoryUpdateFields();
⬜ 127          allow read: if true;  // Allow reading the fields of the repo document - these are public
```
<br/>

Checking our requests has only valid whitelisted keys for creation/update

The request will fail if it has non-whitelisted fields in its data

<br/>



<div>    📄 cloud/firestore.rules  </div>

```rules
⬜ 196          function isSwimmCreator(repoId, swimmId) {
⬜ 197            return get(/databases/$(database)/documents/repositories/$(repoId)/swimms/$(swimmId)).data.creator == request.auth.uid;
⬜ 198          }
🟩 199          function validateAllowedRepositoryCreateFields(){
🟩 200            // Make sure the request fields changes only allowed fields and nothing else
🟩 201            return request.resource.data.keys().toSet().hasOnly(["name", "description", "logo", "creator", "owner", "created", "modified", "modifier", "modifier_name", "url", "provider", "integrations", "counter_swimms", "counter_playlists", "counter_swimmers", "counter_lifeguards"]);
🟩 202          }
⬜ 203          function validateAllowedRepositoryUpdateFields(){
⬜ 204            // Make sure the request fields changes only allowed fields and nothing else
⬜ 205            return request.resource.data.diff(resource.data).affectedKeys().hasOnly(["id", "name", "description", "logo", "modified", "modifier", "modifier_name", "integrations"]);
```
<br/>

**Notice** - We check `affectedKeys` (document fields changed by the request) has only whitelisted keys

We're adding to whitelist only the fields that we're able to update via the UI

<div>    📄 cloud/firestore.rules  </div>

```rules
⬜ 200            // Make sure the request fields changes only allowed fields and nothing else
⬜ 201            return request.resource.data.keys().toSet().hasOnly(["name", "description", "logo", "creator", "owner", "created", "modified", "modifier", "modifier_name", "url", "provider", "integrations", "counter_swimms", "counter_playlists", "counter_swimmers", "counter_lifeguards"]);
⬜ 202          }
🟩 203          function validateAllowedRepositoryUpdateFields(){
🟩 204            // Make sure the request fields changes only allowed fields and nothing else
🟩 205            return request.resource.data.diff(resource.data).affectedKeys().hasOnly(["id", "name", "description", "logo", "modified", "modifier", "modifier_name", "integrations"]);
🟩 206          }
⬜ 207          function canDeleteSwimm(repoId, swimmId) {
⬜ 208            return isUserLifeguard(repoId) || (isUserInPrivateRepo(repoId) && isSwimmCreator(repoId, swimmId))
⬜ 209          }
```
<br/>

Whitelist fields tests
----------------------

Now we test our create/update rules:

<br/>

Make sure a user can create a new repo only when all fields are whitelisted fields

<div>    📄 cloud/firestore_rules.spec.js  </div>

```js
⬜ 670          const repoDocument = db.collection('repositories').doc('test');
⬜ 671          await firebase.assertFails(repoDocument.update({ name: 'name-can-be-edited', url: 'https://swimm.io/route/to/url-failure' }));
⬜ 672        });
🟩 673        it('a user can create a new repo with valid whitelisted data', async () => {
🟩 674          const db = await dbSetup({ uid: noLifeguardUid }, mockData);
🟩 675          const repositories = db.collection('repositories');
🟩 676          await firebase.assertSucceeds(
🟩 677            repositories.add({
🟩 678              counter_lifeguards: 0,
🟩 679              counter_playlists: 0,
🟩 680              counter_swimmers: 0,
🟩 681              counter_swimms: 0,
🟩 682              creator: 'creatorId',
🟩 683              name: 'new added test respository',
🟩 684              owner: 'owner',
🟩 685              provider: 'github',
🟩 686              url: 'https://github.com/kpdecker/jsdiff.git',
🟩 687            })
🟩 688          );
🟩 689        });
🟩 690        it('a user cannot create a new repo with non-whitelisted data', async () => {
🟩 691          const db = await dbSetup({ uid: noLifeguardUid }, mockData);
🟩 692          const repositories = db.collection('repositories');
🟩 693          await firebase.assertFails(repositories.add({ name: 'name-can-be-edited', something: 'not-whitelisted-data' }));
🟩 694        });
⬜ 695        it("a swimmer cannot update the repo's document", async () => {
⬜ 696          const db = await dbSetup({ uid: noLifeguardUid }, mockData);
⬜ 697          const repoDocument = db.collection('repositories').doc('test');
```
<br/>

Make sure a lifeguard that is allowed to update a repository cannot edit not whitelisted fields

<div>    📄 cloud/firestore_rules.spec.js  </div>

```js
⬜ 665            })
⬜ 666          );
⬜ 667        });
🟩 668        it("a lifeguard cannot update repo's non-whitelisted fields", async () => {
🟩 669          const db = await dbSetup({ uid: lifeguardUid }, mockData);
🟩 670          const repoDocument = db.collection('repositories').doc('test');
🟩 671          await firebase.assertFails(repoDocument.update({ name: 'name-can-be-edited', url: 'https://swimm.io/route/to/url-failure' }));
🟩 672        });
⬜ 673        it('a user can create a new repo with valid whitelisted data', async () => {
⬜ 674          const db = await dbSetup({ uid: noLifeguardUid }, mockData);
⬜ 675          const repositories = db.collection('repositories');
```
<br/>

<br/><br/>

This file was generated by Swimm. [Click here to view it in the app](https://swimm.io/link?l=c3dpbW0lM0ElMkYlMkZyZXBvcyUyRnZlZXp2eEN1enBQclJMTFhXRDJFJTJGZG9jcyUyRkhGMjdlZlUzSVNXY3lERllJbnpY). Timestamp: 2021-04-24T18:52:02.156Z
