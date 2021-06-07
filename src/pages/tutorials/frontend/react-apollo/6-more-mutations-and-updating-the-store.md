---
title: More Mutations and Updating the Store
---

The next piece of functionality that you'll implement is the voting feature! Authenticated users are allowed to submit a vote for a link. The most upvoted links will later be displayed on a separate route that you'll implement soon!

### Preparing the React Components

Once more, the first step to implement this new feature is to make your React components ready for the expected functionality.

Open `Link.js` and update `render` to look as follows:

```js
render() {
  const userId = localStorage.getItem(GC_USER_ID)
  return (
    <div>
      {userId && <div onClick={() => this._voteForLink()}>▲</div>}
      <div>{this.props.link.description} ({this.props.link.url})</div>
        <div>{this.props.link.votes.length} votes | by {this.props.link.postedBy ? this.props.link.postedBy.name : 'Unknown'} {this.props.link.createdAt}</div>
    </div>
  )
}
```

You're already preparing the `Link` component to render the number of votes for each link and the name of the user that posted it. Plus you'll render the upvote button if a user is currently logged in.

Also make sure to import `GC_USER_ID` on top the file:

```js
import { GC_USER_ID } from '../constants'
```

### Updating the Schema

For this new feature, you also need to update the schema again since votes on links will be represented with a custom type.

Open `project.graphcool` and add the following type:

```js
type Vote {
  user: User! @relation(name: "UsersVotes")
  link: Link! @relation(name: "VotesOnLink")
}
```

Each `Vote` will be associated with the `User` who created it as well as the `Link` that it belongs to. You also have to add the other end of the relation. 

Still in `project.graphcool`, add the following field to the `User` type:

```graphql
votes: [Vote!]! @relation(name: "UsersVotes")
```  

Now add another field to the `Link` type:

```graphql
votes: [Vote!]! @relation(name: "VotesOnLink")
```

Next open up a Terminal window and navigate to the directory where `project.graphcool` is located. Then apply your schema changes by typing the following command:

```sh
graphcool push
```

Here is what the Terminal output looks like:

```sh
$ gc push
 ✔ Your schema was successfully updated. Here are the changes: 

  | (+)  A new type with the name `Vote` is created.
  |
  | (+)  The relation `UsersVotes` is created. It connects the type `User` with the type `Vote`.
  |
  | (+)  The relation `VotesOnLink` is created. It connects the type `Link` with the type `Vote`.

Your project file project.graphcool was updated. Reload it in your editor if needed.
```

Awesome! Let's now move on and implement the upvote mutation!


### Calling the Mutation

Open `Link.js` and add the following mutation definition to the bottom of the file:

```js
const CREATE_VOTE_MUTATION = gql`
  mutation CreateVoteMutation($userId: ID!, $linkId: ID!) {
    createVote(userId: $userId, linkId: $linkId) {
      id
      link {
        votes {
          id
          user {
            id
          }
        }
      }
      user {
        id
      }
    }
  }
`

export default graphql(CREATE_VOTE_MUTATION, {
  name: 'createVoteMutation'
})(Link)
```

This step should feel pretty familiar by now. You're adding the ability to call the `createVoteMutation` to the `Link` component by wrapping it with the `CREATE_VOTE_MUTATIIN`.

Finally, you need to implement `_voteForLink` as follows:

```js
_voteForLink = async () => {
  const userId = localStorage.getItem(GC_USER_ID)
  const voterIds = this.props.link.votes.map(vote => vote.user.id)
  if (voterIds.includes(userId)) {
    console.log(`User (${userId}) already voted for this link.`)
    return
  }

  const linkId = this.props.link.id
  await this.props.createVoteMutation({
    variables: {
      userId,
      linkId
    }
  })
}
```

Notice that in the first part of the method, you're checking whether the current user already voted for that link. If that's the case, you return early from the method and not actually execute the mutation.

You now also need to include the information about the links' votes in the `allLinks` query that's defined in `LinkList`.

Open `LinkList.js` and update the definition of `ALL_LINKS_QUERY` to look as follows:

```js
export const ALL_LINKS_QUERY = gql`
  query AllLinksQuery($first: Int, $skip: Int, $orderBy: LinkOrderBy) {
    allLinks(first: $first, skip: $skip, orderBy: $orderBy) {
      id
      createdAt
      url
      description
      postedBy {
        id
        name
      }
      votes {
        id
        user {
          id
        }
      }
    }
  }
`
```

All you do here is to also include information about the user who posted a link as well as information about the links' votes in the query's payload.

You can now go and test the implementation! Run `yarn start` and click the upvote button on a link. You're not getting any UI feedback yet, however, you'll notice that after you clicked the button for the first time, all subsequent clicks will simply print the loggin statement: _User (cj42qfzwnugfo01955uasit8l) already voted for this link._

Great, so you at least know that the mutation is working! You can also refresh the page and you'll see the vote is counted and the link is now shown to have one upvote.


### Updating the Cache

One cool thing about Apollo is that you can manually control the contents of the cache. This is really handy especially after a mutation was performed, since this allows to determine precisely how you want the cache to be updated. Here, you'll use it to make sure the UI gets updated after the mutation was performed.

You can implement this functionality by using Apollo's [imperative store API](https://dev-blog.apollodata.com/apollo-clients-new-imperative-store-api-6cb69318a1e3).

Open `Link` and update the call to `createVoteMutation` inside the `_voteForLink` method as follows:

```js
const linkId = this.props.link.id
await this.props.createVoteMutation({
  variables: {
    userId,
    linkId
  },
  update: (store, { data: { createVote } }) => {
    this.props.updateStoreAfterVote(store, createVote, linkId)
  }
})
```

The `update` function that we're adding as an argument to the mutation call will be called when the server returns the response. It receives the payload of the mutation (`data`) and the current cache (`store`) as arguments. You can then use this input to determine a new state of the cache. 

Notice that we're already _desctructuring_ the server response and retrieving the `createVote` field from it. 

All right, so now you know what this `update` function is, but the actual implementation will be done in the parent component of `Link`, which is `LinkList`. 

Open `LinkList.js` and add the following method to inside the scope of the `LinkList` component:

```js
_updateCacheAfterVote = (store, createVote, linkId) => {
  // 1
  const data = store.readQuery({ query: ALL_LINKS_QUERY })
  
  // 2
  const votedLink = data.allLinks.find(link => link.id === linkId)
  votedLink.votes = createVote.link.votes
  
  // 3
  store.writeQuery({ query: ALL_LINKS_QUERY, data })
}
```

What's going on here?

1. You start by reading the current state of the cached data for the `ALL_LINKS_QUERY` from the `store`.
2. Now you're retrieving the link that the user just voted for from that list. You're also manipulating that link by resetting its `votes` to the `votes` that were just returned by the server.
3. Finally, you take the modified data and write it back into the store.

Next you need to pass this function down to the `Link` component so it can be called from there. 

Still in `LinkList.js`, update the way how links are rendered in `render`:

```js
<Link key={link.id} updateStoreAfterVote={this._updateCacheAfterVote} link={link}/>
```

That's it! The `updater` function will now be executed and make sure that the store gets updated properly after a mutation was performed. The store update will trigger a rerender of the component and thus update the UI with the correct information!

While we're at it, let's also implement `update` for adding new links!

Open `CreateLink.js` and update the call to `createLinkMutation` inside `_createLink` like so:

```js
await this.props.createLinkMutation({
  variables: {
    description,
    url,
    postedById
  },
  update: (store, { data: { createLink } }) => {
    const data = store.readQuery({ query: ALL_LINKS_QUERY })
    data.allLinks.splice(0,0,createLink)
    store.writeQuery({
      query: ALL_LINKS_QUERY,
      data
    })
  }
})
```

The `update` function works in a very similar way as before. You first read the current state of the results of the `ALL_LINKS_QUERY`. Then you insert the newest link to the top and write the query results back to the store.

The last think you need to do for this to work is add import the `ALL_LINKS_QUERY` into that file:

```js
import { ALL_LINKS_QUERY } from './LinkList'
```

Conversely, it also needs to be exported from where it is defined. 

Open `LinkList.js` and adjust the definition of the `ALL_LINKS_QUERY` by adding the `export` keyword to it:

```js
export const ALL_LINKS_QUERY = ...
```

Awesome, now the store also updates with the right information after new links are added. The app is getting in shape. 🤓