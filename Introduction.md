# Firestore Data Modelling

Firestore data modelling is based on multiple decisions:
relationships between entity fields, optimizing reads and writes, security of the data and reducing the cost of the firestore fee.
Firestore is a NoSQL like database, in which the entire db is stored as a one large JSON object.

## Some of the Questions to ask for data modelling:

1- What is the relationship between fields / entities
2- Data fields required for every UI screen in the application and their frequency of read
3- How often which data fields will be updated?
4- Security of data fields?
5- Divide sensitive data between collections to lock them down from client side access using firestore security rules, and only access them with Firebase functions and rules (get, exists methods server side)
6- When one document is read, it is read in its entirety. We can not filter fields server end, and send back only the selected fields.
7- Security rules, are applicable on the entire collection/path. Therefore if one method is allowed by a rule it will be allowed 100% of the time for that method (eg: read, write etc)
8- All the fields of a document are indexed individually by default, by the firestore, and also they keys in a map are indexed as well.

## Best practice:

1- the data model should match the UI as close as possible
2- Make data models for every UI screen needs, and use that to inform your firestore data modelling decisions.
2- The complexity of queries and their associated cost of R/W should be reduced
3- The main purpose of the data model, is that we can access it for any method we desire, using its document Id or reference available to us in request, resource or auth object.

## Popular Techniques

## 1-1 Relationship - Embed it

In cases, of 1-1 relationship, and with fields less than 20k per document and size less than 1.0 MB its best to embed all the relevant fields on the document. This way with just one document read we will have all the relevant data fields that we require to populate a UI.

## 1-many relationship : Using Root Collections

Root collections for example: users, notes, posts, comments, likes followers at the root of the firestore can be very flexible and allow future changes to be much easier to extend the app data model as well. But it might have higher Read cost than an embedded or sub collection respectively.
Eg: one user can have many notes, and then on every note we have the userId as a string field stored on it. And we can query all the notes for a specific user, using where ('userId' '==' 'value').

## 1-many relationship: using sub collection

Sub collection are very useful in two ways, they allow nesting of a single users related data inside a nested sub collection thus naturally building a 1-many relationship and scoping data to that user only. Secondly, we only read the documents from the sub-collection when we need them, and can perform queries on them too. And filter the documents we need against a criteria.
Group queries now also allow us to perform search across multiple users sub-collections as long as all of them share the same name.

## Bucketing

We model related data into two separate root collections and they have the same document id.

### Users Secret Information

We should setup a user secrets collection that has important secret stuff that should not be exposed to client side rather available server side to check for access and keeping user information safe. Such as phone numbers, email addresses and their access roles. The documents for every user in the secrets collection will have the same UID as their documents in the profiles collection and their UID from firebase authentication module as well, to create a 1-1 unique relationship

### Banned users collection

we can have a restricted or banned users collection setup in such a way, that every banned user is added to the banned users root collection with an empty document and the document Id is the users UID. And then for every Read/ write requests to the other collection docs we check the request.auth.uid against the banned collection. If a document exists, then the transaction is stopped. Its setup in the security rules as a safeguard and we would also have to check using a background function on new user creation as well, if they are already banned and not allow them to make a new account using same credentials.
Thus firestore security rules, will play the most important part in keeping banned users from accessing resources.

## Many- many relationship

### Reviews for books by users

On a books marketplace, users who buy books, can leave reviews for them. Every user can only leave one review per book. A book can have millions (many) of reviews, and a user can give many reviews as well. Therefore, one way to do can be to put reviews in a separate root collection with every review having a composite document Id created using the user uid and the book uid. such as ${user_uid}_${book_uid}.
This would help us get reviews from the reviews collection for a book and also every review given by a certain user too.
And the review document will hold reference / string values for the book uid and the user uid as well. Along with the text for the review and any other required fields.

UI data needs:
User UI: All of his reviews
Book showing its all reviews by all users

PROS: Flexible, scalable, extendable for future needs
CONS: Multiple document reads, to show all book reviews and user reviews

Alternative, if we have to show book reviews together in a UI frequently. Then we can embed the reviews on the books document along-with the user UID for a reference to read reviews by a certain user for his user profile page. And if we put some of his username, photoUrl and display name too on the review. But remember, when the user changes that, it will have to be updated on every book document where he has made a review too.

/books root collection
title="Book Title1"
publishedDate=21Jan, 22

Reviews Map
{
userID1 = Map
{text="",
rating=4,
username=""}

        userID2= Map
        {text="",
        rating=4,
        username=""}

}

## Aggregation

We calculate values based on number of documents in a collection. This is especially useful in scenarios such as: likes, hearts, favorites, order totals for customers, upvotes-reddit, followers, following etc.
const getLikedBooks = async() => {

    // Get books through user likes
    const userLikes = await db.collection('likes').orderBy('jeff-delaney').get();
    const bookIds = userLikes.docs.map(snap => snap.id);

    const bookReads = bookIds.map(id => db.collection('books').doc(id).get() );
    const books = Promise.all(bookReads)

}

getLikedBooks()

### Example: Post like scenario

As a firestore document has a limit of 20k fields per document and size of 1.0MB, therefore, every time a new like document is created we use a background function to update a counter on the post document. Using the onCreate() method for likes collection. The like document, will only hold the userId, PostId and value=1. And the background function will add the total likes count to the user and post documents. This way, we can search all the posts liked by a user, and also all the users who liked a certain post, as secondary less frequent operations. and the primary operation of showing like / vote count will be available on the post document. Thus saving the reads from likes collection. Also the UID for the likes/ votes document will have a composite Id such as ${userId}_${post_id}. Enforcing 1-1 relationship between post and user who likes it. One user can only like a post once.

## Role Based Authorization

An alternative to the method below, is custom claims provided by the firebase that we setup server side using admin module. And that sets up props on the auth token with custom properties.

### Custom Roles on a separate document

Roles= ['pro', 'editor', 'admin', 'moderator', 'author']
such roles array is created on the user document inside a secret collection, i.e. locked down from client side access.
Custom roles, are good in scenarios, where the entire site is locked down based on a role. And individual content doesn't need separate lists of whose authorized to view a certain page / view / resource.
But if we have a scenario like rent to view, where every content has its own access list then we should rather embed the users ACL (access control list) on the content server side. And check against that instead of having user roles on the user documents for every user.
ACL will be embedded using an Array on the content / post document with userIDs of allowed users on it.

Role base authorization

match /posts/{document} {

    function getRoles() {
        return get(/databases/$(database)/documents/users/$(request.auth.uid)).data.roles;
    }

    allow read;
    allow update: if getRoles().hasAny(['admin', 'editor']) == true;
    allow write: if getRoles().hasAny(['admin']) == true;

}

#### Remember: Security roles are the ones who ensure roles and ACL are applied server side using security rules to prevent unauthorized access. Client side validations are first step and must always be taken as well.

Put the ACL on user, or the ACL on the content / post doc wherever the size is smaller.

ACL
embed on the document the userIds who are allowed to read a certain post
match /posts/{document} {

    allow read;
    allow write: if resource.data.members.hasAny(request.auth.uid);

}

## Collection group queries for sub collections

Sub collections inside a document, such as notes collection in the user document will hold all the notes created by every user as a document. And with group-collection we can query notes of all users as long as their sub collection of notes is called "notes" the name of the sub collection must be same among all users or must have the same path and name.
