# Memory bank

Users are stored as a list of weakRefs:
```js
class UserRegistry {
  constructor() {
    this.users = [];
  }
  addUser(user) {
    this.users.push(new WeakRef(user));
  }
  ```
  An admin user we need to log in as is also created at the start, and we can't make a new user with a duplicate name. Triggering garbage collection will delete the admin user, so we can make one with its username and get the flag.
<br>
The withdraw option creates a bunch of `Bill` objects, with our signature. By adding a long signature to our user, and then withrdawing our entire balance of 101 in a bogus denomination like `0.01`, we can create enough objects to trigger garbage collection, delete the admin user, and then register with their username and get the flag.
  ```js
        const numBills = amount / denomination;
        const bills = [];

        for (let i = 0; i < numBills; i++) {
          bills.push(new Bill(denomination, currentUser.signature || 'VOID'));
        }
        
        currentUser.balance -= amount;
        
  ```
