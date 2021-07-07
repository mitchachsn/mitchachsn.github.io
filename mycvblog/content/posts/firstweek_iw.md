---
title: "Implementing Firebase to create an effective multiplayer lobby"
date: 2021-06-29T11:39:05+01:00
draft: false
---

# Main layout of game lobby

In my little over a month intershp with Infinite Whys, I was tasked with creating a web application multiplayer lobby that functions in a similar way how lobbies in the popular game "[Among us][1]" functioned. With this project given to me, the main goal is to create all the necessary functions needed for the lobby to run soundly, thus having a heavier focus on the back-end code. This blog article will run through how I go about constructing the code using various firebase functions as well as Javascript, and will follow the user's path of joining/creating a lobby.

![Front-end lobby image incomplete](images/front-end_lobbyincomplete.png)

# Database model (Realtime database)

Before jumping into the process of how I created the functions for the lobby, the database model must have been discussed and established in order for these functions to work properly. In this case, the usage of [Firebase Realtime Database][2] was used. it is a cloud-hosted NoSQL database that allows for storing and syncing of data between active users in realtime. There are many benefits that firebase provides that prove to be important to a lobby structure such as:

1. Real time updates
2. Offline support
3. Accessibility
4. Scalable across multiple databases
   
With these benefits, constructing the database is possible with more efficiency. The basic structure of the lobby/room is shown in the image below:

![Database structure](images/database-structure.png)

The parts of the lobby/room will be discuss further in the other parts of the article, but from the image, the lobbies shall be recognised by unique lobby codes, and each lobby separates the users into a single owner and the remaining are players. the data is also stored as one large JSON tree, making it very easy to store simple data.

With the database structure defined, the main lobby functions can be made based around it. Below shows the workflow of the lobby.

![Lobby workflow](mycvblog/static/images/workflow_lobby.png)

# Creating a room

Creating a room/lobby is one of the essential functionality needed for a multiplayer lobby, so it will be the first to be implmented. The basic process of how a user can create a room is as follows, which can also be seen in the flowchart:

1. User gets Aunthenticated (when entering the site)
2. Room is created with unique code
3. User is appointed owner (based on their unique ID)
4. User can start the game or disban the lobby

The process of implementing the above uses a couple of resources from firebase which are stated below and will be explained further in this section:

1. Firebase Aunthentication
2. Cloud Functions and triggers

## Authentication

[Anonymous Authentication][3] is used to allow for the users to have access to the lobbies to join or to create a new lobby to input into the database. This also allows for each user to have a unique ID, which is used to recognise the user and as a reference to the user when they create or join a lobby. This also allows for multiple users to share the same display name without any clashes in referencing. The anonymous authentication can be implemented in the client-side using the code below:

```javascript
    firebase.auth().signInAnonymously()
      .then((user) => {
        console.log('authenticated', user);
        user_UID = user.uid;
        console.log(user.uid);

      }).catch((error) => {
        var errorCode = error.code;
        var errorMessage = error.message;
      });
```

And once the user joins/creates a room, the unique ID will then be pushed to the database. Having this authentication also allow for future plans of conversion into a permanent account if the game heads in that direction.

## Cloud Functions and triggers

When creating the lobby, there are several [cloud functions][5] created that could have been done in the client-side directly instead, which will reduces the workload on firebase and in turn lower the total cost of the [blaze][4] plan. However, I decided to do all these functions in the form of cloud functions to further my understanding on the syntax and workings of firebase cloud functions. 

There are many benefits that cloud functions provide, especially with it being a serverless framework. A partial code is shown below how a cloud function is used to as a listener for other users in the lobby if a new user joins the room:

```javascript

    admin.initializeApp();

    var room_database = admin.database().ref('rooms/ABCD/players');

    // authentication trigger (new user enters/create room)

    exports.newUserAdded = functions.auth.user().onCreate((user) => {
        console.log('user created', user.uid, user.displayName);
        const userUID = user.uid;
        return room_database.push({
        PlayerUID: userUID
        });
    });
```

The above can also be categorised as a trigger, as it uses the onCreate() handler, which tiggers when data is created, updated, or deleted in the database. This is the basis for keeping users in a lobby updated with players entering/leaving the lobby, and allows for the data to update for all the users involved. Many other [event handlers][6] can be used to accomodate different triggers required.

# Joining a room

With joining a room, similar functionalities are used when comparing with creating a room. The changes is that instead of creating a new lobby, the user will have the ability to enter a lobby code, and there a function will cross-check with the database to see if this lobby exist. If it does then it will update the lobby list and place the user's unique ID in the player list. It will also redirect the user's front-end to the lobby mentioned. Here below is a snippet of the code explained:

```javascript

// http callable function (join OR create room)
exports.addRequest = functions.https.onCall((data, context) => {
  if(!context.auth){
    console.log(context.auth.uid);
    throw new functions.https.HttpsError(
      'unauthenticated',
      'only authenticated users can join/create room'
    );
  }
  if(data.text.length > 4){
    throw new functions.https.HttpsError(
      'invalid-argument',
      'Invalid room code' // to avoid any wrong room code (do this in front-end instead?)
    );   
  }

  console.log(context.auth.uid);

  var room_code = context.text
  var full_dir = "rooms/" + room_code + "/players"

  return full_dir.push({
    PlayerUID: context.auth.uid,
    displayName: context.name
  });


});

```

extra parts to code is the error detection if an invalid code is entered or if the user is not authenticated by firebase. 

# Functionalities in lobby

Once the users are in the lobby, the players are able to:

1. Change display name
2. changing status (ready or not ready)

while the owner of the lobby are able to:

1. Change display name
2. Start game (when all users are ready)
3. Disban entire lobby (delete lobby)

Some of these functions are not implemented yet but are for future improvements to the lobby.


[1]: <https://innersloth.com/gameAmongUs.php> "About among us"
[2]: <https://firebase.google.com/docs/database> "About Firebase RTDB"
[3]: <https://firebase.google.com/docs/auth/web/anonymous-auth> "About Anony Auth"
[4]: <https://firebase.google.com/pricing> "Blaze plan pricing"
[5]: <https://firebase.google.com/docs/functions> "Cloud functions"
[6]: <https://firebase.google.com/docs/functions/database-events> "Event Handlers"