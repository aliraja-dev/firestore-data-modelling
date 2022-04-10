# Firestore Data Modelling

Firestore data modelling is based on multiple decisions:
relationships between entity fields, optimizing reads and writes, security of the data and reducing the cost of the firestore fee.
Firestore is a NOSQL like database, in which the entire db is stored as a one large JSON object.

Embed. Model data directly on a document.
Root collection. Normalize data into separate collections, then reference document IDs.
Sub-collection. Nest data in a collection under a document.
Bucket. Separate data into multiple documents, but embed as much as possible.

Questions to ask for data modelling in firestore:
1- relationship between fields / entities
2- Data fields required for every UI screen in the application and their frequency of read
3- How often which data fields will be updated?
4- Security of data fields?
5- dividing data between collections to lock them down and only access them with Firebase functions and security rules.
6- When one document is read, it is read in its entirety. We can not filter fields server end, and send back those fields only.
7- security rules, are applicable on the entire collection. Therefore if one method is allowed by a rule it will be allowed 100% of the time for that method (eg: read, write etc)
8- All the fields of a document are indexed individually by default, by the firestore, and also they keys in a map are also indexed.

Best practice:
1- the data model should match the UI as close as possible
2- the efficiency of queries and their cost of R/W should be reduced

## 1-1 Relationship

In cases, of 1-1 relationship, and with fields less than 20k per document and size less than 1.0 MB its best to embed all the relevant fields on the document. This way with just one document read we will have all the relevant data fields that we require to populate a UI.

## 1-many relationship : Using Root Collections

Root collections for example: users, notes, posts, comments, likes followers at the root of the firestore can be very flexible and allow future changes to be much easier to extend the app data model as well. But it might have higher Read cost than an embedded or sub collection respectively.
Eg: one user can have many notes, and then on every note we have the userId as a string field stored on it. And we can query all the notes for a specific user, using where ('userId' '==' 'value').

## 1-many relationship: using sub collection

Sub collection are very useful in two ways, they allow nesting of a single users related data inside a nested sub collection thus naturally building a 1-many relationship and scoping data to that user only. Secondly, we only read the documents from the subcollection when we need them, and can perform queries on them too. And filter the documents we need against a criteria.
Group queries now also allow us to perform search across multiple users subcollections as long as all of them share the same name.

To get likes, hearts or favorite books for a user.
/ books UID is also the UID for likes document in root likes collection. And the authors doc iD is placed as a field on the likes document with boolean value. to show if a user has heart it or not.

// 10. Bucket
// Get a collection of documents with an array of IDs

const getLikedBooks = async() => {

    // Get books through user likes
    const userLikes = await db.collection('likes').orderBy('jeff-delaney').get();
    const bookIds = userLikes.docs.map(snap => snap.id);

    const bookReads = bookIds.map(id => db.collection('books').doc(id).get() );
    const books = Promise.all(bookReads)

}

getLikedBooks()

Role base authoriztion

match /posts/{document} {

    function getRoles() {
        return get(/databases/$(database)/documents/users/$(request.auth.uid)).data.roles;
    }

    allow read;
    allow update: if getRoles().hasAny(['admin', 'editor']) == true;
    allow write: if getRoles().hasAny(['admin']) == true;

}

ACL
embed on the document the userIds who are allowed to read a certain post
match /posts/{document} {

    allow read;
    allow write: if resource.data.members.hasAny(request.auth.uid);

}

Model a tree or Hierarchy for threaded comments
composite key that represents the parents keys
A
AB
ABD

const topLevel = db.collection('comments').where('parent', '==', false);

const level = db.collection('comments').where('parent', '==', id)

const traverseAll = (id) => {
const tree = db.collection('comments')
.where('parent', '>=', id)
.where('parent', '<=', `${id}~`)
}

Follow a celebrity and how to show a feed for user with his all followings recent posts in his timeline / feed

import { db } from './config';
import firebase from 'firebase/app;
const remove = firebase.firestore.FieldValue.arrayRemove;
const union = firebase.firestore.FieldValue.arrayUnion;

export const follow = (followed, follower) => {
const followersRef = db.collection('followers').doc(followed);

followersRef.update({ users: union(follower) });
}

// 2. Unfollow User

export const unfollow = (followed, follower) => {
const followersRef = db.collection('followers').doc(followed);

    followersRef.update({ users: remove(follower) });

}

// 3. Get posts of followers

export const getFeed = async() => {

    const followedUsers = await db.collection('followers')
        .where('users', 'array-contains', 'jeffd23')
        .orderBy('lastPost', 'desc')
        .limit(10)
        .get();


    const data = followedUsers.docs.map(doc => doc.data());

    const posts = data.reduce((acc, cur) => acc.concat(cur.recentPosts), []);


    const sortedPosts = posts.sort((a, b) => b.published - a.published)


    // render sortedPosts in DOM

}
