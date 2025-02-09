# React Fetch CRUD Codealong (Update and Delete)

## Learning Goals

- Write `fetch` requests for `PATCH` and `DELETE`
- Initiate `fetch` requests from user events
- Update state and trigger a re-render after receiving a response to the `fetch` request
- Perform Update and Delete (CRUD) actions on arrays in state

## Introduction

If you have not yet completed the Create and Read actions on the Shopping List application, [lesson GitHub repo: Fetch CRUD Codealong Create and Read](https://github.com/learn-co-curriculum/react-hooks-fetch-cr-code-along/tree/main), it is recommended you start by completing that lesson. We'll continue working on the application in this lesson.
The last time we worked on this application, we used `fetch` to display our shopping list and persist new items via a form.

In both of those cases, our workflow followed the pattern below:

- When X event occurs (_our application loads_, _a user clicks a button_)
- Make Y fetch request (_GET_)
- Update Z state (_add all items to state_)

In this codealong lesson, we'll get more practice following this pattern to
build out all our final 2 CRUD actions to work with both our **server-side** data (the
database; in our case, the `db.json` file) as well as our **client-side** data
(our React state). We'll be revisiting the shopping list application from the
previous module, this time implemented Update and Delete functionality.

## Instructions

To get started, let's install our dependencies:

```console
$ npm install
```

Then, to run `json-server`, we'll be using the `server` script in the
`package.json` file:

```console
$ npm run server
```

This will run `json-server` on [http://localhost:4000](http://localhost:4000).
Before moving ahead, open
[http://localhost:4000/items](http://localhost:4000/items) in the browser and
refamiliarize yourself with the data. What are the important keys on each object?

Leave `json-server` running. Open a new terminal, and run React with:

```console
$ npm run dev
```

View the application in the browser at
[http://localhost:3000](http://localhost:3000).

Take some time now to refamiliarize yourself with the app and the components 
in the `src/components` folder. Which components are stateful and why? What does 
our component hierarchy look like?

Once you've familiarized yourself with the starter code, let's start building
out the update and delete functionality!

### Updating Items

For our update action, we'd like to give our users the ability to keep track of which
items from their shopping list they've added to their cart. Once more, we can outline
the basic steps for this action like so:

- When X event occurs (_a user clicks the Add to Cart button_)
- Make Y fetch request (_PATCH /items_)
- Update Z state (_update the `isInCart` status for the item_)

From here, we'll again need to **identify which component triggers the event**.
Can you find where the "Add to Cart" button lives in our code? Yep! It's in the
`Item` component. We'll start by adding an event handler for clicks on the
button:

```jsx
// src/components/Item.js

function Item({ item }) {
  // Add function to handle button click
  function handleAddToCartClick() {
    console.log("clicked item:", item);
  }

  return (
    <li className={item.isInCart ? "in-cart" : ""}>
      <span>{item.name}</span>
      <span className="category">{item.category}</span>
      {/* add the onClick listener */}
      <button
        className={item.isInCart ? "remove" : "add"}
        onClick={handleAddToCartClick}
      >
        {item.isInCart ? "Remove From" : "Add to"} Cart
      </button>
      <button className="remove">Delete</button>
    </li>
  );
}
```

Check your work by clicking this button for different items — you should see
each item logged to the console. We can access the `item` variable in the
`handleAddToCartClick` function thanks to JavaScript's scope rules.

Next, let's write out our `PATCH` request:

```js
// src/components/Item.js

function handleAddToCartClick() {
  // add fetch request
  fetch(`http://localhost:4000/items/${item.id}`, {
    method: "PATCH",
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      isInCart: !item.isInCart,
    }),
  })
    .then(r => {
      if (r.ok) {
        return r.json()
      } else {
        console.log("failed to update item")
      }
    })
    .then(updatedItem => console.log(updatedItem))
    .catch(error => console.log(error))
}
```

Just like with our `POST` request, we need to specify the `method`, `headers`,
and `body` options for our `PATCH` request as well. We also need to include the
item's ID in the URL so that our server knows which item we're trying to update.

Since our goal is to let users add or remove items from their cart, we need to
toggle the `isInCart` property of the item on the server (and eventually
client-side as well). So in the body of the request, we send an object with the
key we are updating, along with the new value.

Check your work by clicking the "Add to Cart" button and see if you receive an
updated item in the response from the server. Then, refresh the page to verify
that the change was persisted server-side.

Two steps done! Once more, our last step is to update state. Let's think this
through. Right now, what state determines whether or not the item is in our
cart? Where does that state live? Do we need to add new state?

A reasonable thought here is that we might need to add new state to the `Item`
component to keep track of whether or not the item is in the cart. While we
could theoretically make this approach work, it would be an anti-pattern: we'd
be **duplicating state**, which makes our components harder to work with and
more prone to bugs.

We already have state in our `ShoppingList` component that tells us which items
are in the cart. So instead of creating new state, our goal is to call
`setItems` in the `ShoppingList` component with a new list of items, where the
`isInCart` state of our updated item matches its state on the server.

Just like with our `ItemForm` deliverable, let's start by creating a callback
function in the `ShoppingList` component and passing it as a prop to the `Item`
component:

```jsx
// src/components/ShoppingList.js

function ShoppingList() {
  const [selectedCategory, setSelectedCategory] = useState("All");
  const [items, setItems] = useState([]);

  useEffect(() => {
    fetch("http://localhost:4000/items")
      .then(r => {
        if (r.ok) {
          return r.json()
        } else {
          console.log("fetch request failed")
        }
      })
      .then(items => setItems(items))
      .catch(error => console.log(error))
  }, []);

  // add this callback function
  function handleUpdateItem(updatedItem) {
    console.log("In ShoppingCart:", updatedItem);
  }

  // ...rest of component

  return (
    <div className="ShoppingList">
      <ItemForm onAddItem={handleAddItem} />
      <Filter
        category={selectedCategory}
        onCategoryChange={handleCategoryChange}
      />
      <ul className="Items">
        {/* pass it as a prop to Item */}
        {itemsToDisplay.map(item => (
          <Item key={item.id} item={item} onUpdateItem={handleUpdateItem} />
        ))}
      </ul>
    </div>
  );
}
```

In the `Item` component, we can destructure the `onUpdateItem` prop and call it
when we have the updated item response from the server:

```jsx
// src/components/Item.js

// Destructure the onUpdateItem prop
function Item({ item, onUpdateItem }) {
  function handleAddToCartClick() {
    // Call onUpdateItem, passing the data returned from the fetch request
    fetch(`http://localhost:4000/items/${item.id}`, {
      method: "PATCH",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        isInCart: !item.isInCart,
      }),
    })
      .then(r => {
        if (r.ok) {
          return r.json()
        } else {
          console.log("failed to update item")
        }
      })
      .then(updatedItem => onUpdateItem(updatedItem))
      .catch(error => console.log(error))
  }
  // ... rest of component
}
```

Check your work by clicking the button once more. You should now see the updated
item logged to the console, this time from the `ShoppingList` component.

As a last step, we need to call `setState` with a new array that replaces one
item with the new updated item from the server. Recall from our lessons on
working with arrays in state that we can use `.map` to help create this new
array:

```js
// src/components/ShoppingList.js

function handleUpdateItem(updatedItem) {
  const updatedItems = items.map(item => {
    if (item.id === updatedItem.id) {
      return updatedItem;
    } else {
      return item;
    }
  });
  setItems(updatedItems);
}
```

Clicking the button should now toggle the `isInCart` property of any item in the
list on the server as well as in our React state! To recap:

- When X event occurs
  - When a user clicks the Add to Cart button, handle the button click
- Make Y fetch request
  - Make a `PATCH` request to `/items/:id`, using the clicked item's data for
    the ID and body of the request, and access the updated item in the response
- Update Z state
  - Send the item from the fetch response to the `ShoppingList` component, and
    set state by creating a new array which contains the updated item in place
    of the old item

### Deleting Items

Last one! For our delete action, we'd like to give our users the ability to
remove items from their shopping list:

- When X event occurs (_a user clicks the DELETE button_)
- Make Y fetch request (_DELETE /items_)
- Update Z state (_remove the item from the list_)

From here, we'll again need to **identify which component triggers the event**.
Our delete button is in the `Item` component, so we'll start by adding an event
handler for clicks on the button:

```jsx
// src/components/Item.js

function Item({ item, onUpdateItem }) {
  // ...rest of component

  function handleDeleteClick() {
    console.log(item);
  }

  return (
    <li className={item.isInCart ? "in-cart" : ""}>
      {/* ... rest of JSX */}

      {/* ... add onClick */}
      <button className="remove" onClick={handleDeleteClick}>
        Delete
      </button>
    </li>
  );
}
```

This step should feel similar to our approach for the update action. Next, let's
write out our `DELETE` request:

```js
// src/components/Item.js

function handleDeleteClick() {
  fetch(`http://localhost:4000/items/${item.id}`, {
    method: "DELETE",
  })
    .then(r => {
      if (r.ok) {
        return r.json()
      } else {
        console.log("failed to delete item")
      }
    })
    .then(() => console.log("deleted!"))
    .catch(error => console.log(error))
}
```

Note that for a `DELETE` request, we must include the ID of the item we're
deleting in the URL. We only need the `method` option — no `body` or `headers`
are needed since we don't have any additional data to send besides the ID.

You can verify that the item was successfully deleted by clicking the button,
checking that the console message of `"deleted!"` appears, and refreshing the
page to check that the item is no longer on the list.

Our last step is to update state. Once again, the state that determines which
items are being displayed is the `items` state in the `ShoppingList` component,
so we need to call `setItems` in that component with a new list of items that
**does not contain our deleted item**.

We'll pass a callback down from `ShoppingList` to `Item`, just like we did for
the update action:

```jsx
// src/components/ShoppingList.js

function ShoppingList() {
  const [selectedCategory, setSelectedCategory] = useState("All");
  const [items, setItems] = useState([]);

  useEffect(() => {
    fetch("http://localhost:4000/items")
      .then(r => {
        if (r.ok) {
          return r.json()
        } else {
          console.log("fetch request failed")
        }
      })
      .then(items => setItems(items))
      .catch(error => console.log(error))
  }, []);

  // add this callback function
  function handleDeleteItem(deletedItem) {
    console.log("In ShoppingCart:", deletedItem);
  }

  // ...rest of component

  return (
    <div className="ShoppingList">
      <ItemForm onAddItem={handleAddItem} />
      <Filter
        category={selectedCategory}
        onCategoryChange={handleCategoryChange}
      />
      <ul className="Items">
        {/* pass it as a prop to Item */}
        {itemsToDisplay.map(item => (
          <Item
            key={item.id}
            item={item}
            onUpdateItem={handleUpdateItem}
            onDeleteItem={handleDeleteItem}
          />
        ))}
      </ul>
    </div>
  );
}
```

Call the `onDeleteItem` prop in the `Item` component once the item has been
deleted from the server, and pass up the item that was clicked:

```jsx
// src/components/Item.js

// Deconstruct the onDeleteItem prop
function Item({ item, onUpdateItem, onDeleteItem }) {
  function handleDeleteClick() {
    // Call onDeleteItem, passing the deleted item
    fetch(`http://localhost:4000/items/${item.id}`, {
      method: "DELETE",
    })
      .then(r => {
      if (r.ok) {
        return r.json()
      } else {
        console.log("failed to delete item")
      }
    })
    .then(() => onDeleteItem(item))
    .catch(error => console.log(error)) 
  }
  // ... rest of component
}
```

As a last step, we need to call `setState` with a new array that removes the
deleted item from the list. Recall from our lessons on working with arrays in
state that we can use `.filter` to help create this new array:

```js
// src/components/ShoppingList.js
function handleDeleteItem(deletedItem) {
  const updatedItems = items.filter(item => item.id !== deletedItem.id);
  setItems(updatedItems);
}
```

Clicking the delete button should now delete the item in the list on the server
as well as in our React state! To recap:

- When X event occurs
  - When a user clicks the Delete button, handle the button click
- Make Y fetch request
  - Make a `DELETE` request to `/items/:id`, using the clicked item's data for
    the ID
- Update Z state
  - Send the clicked item to the `ShoppingList` component, and set state by
    creating a new array in which the deleted item has been filtered out

## Conclusion

Synchronizing state between a client-side application and a server-side
application is a challenging problem! Thankfully, the general steps to
accomplish this in React are the same regardless of what kind of action we are
performing:

- When X event occurs
- Make Y fetch request
- Update Z state

Keep these steps in mind any time you're working on a feature involving
synchronizing client and server state. Once you have this general framework, you
can apply the other things you've learned about React to each step, like
handling events, updating state, and passing data between components.

## Resources

- [MDN: Using Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch#uploading_json_data)
