## Timeline feed example

### A data model for the timeline that shows recent posts from the celebrities followed by a user.

From fireship.io

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

## Push Notifications using user mentions in a post

We add a mentions array to every post created. It will hold the userIds for the users mentioned in the post, and then a background function will run onCreate to send a push notification or email to every userId in that array
