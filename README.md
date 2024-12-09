![passport-magic-login](https://user-images.githubusercontent.com/7525670/104158644-0c61f400-53ee-11eb-960f-167c6ebd3ec9.png)

Passwordless authentication with magic links for Passport.js 🔑

- User signup and login without passwords
- Supports magic links sent via email, SMS or any other method you prefer
- Create magic links in code for inclusion in your own methods and pipelines
- User interface agnostic: all you need is an input and a confirmation screen
- Handles secure token generation, expiration and confirmation

Originally implemented by [Tobias Lins](https://twitter.com/linstobias) for [Splitbee](https://splitbee.io) and extracted
for [Feedback Fish](https://feedback.fish), with further development by [Neil Rackett](https://github.com/neilrackett)
for [Mesmotronic](https://mesmotronic.com)

## Usage

To use magic link authentication, you can:

1. Setup the Passport strategy and Express routes on your server
2. POST a request with the user's email or phone number from the client once they have entered it into the login input; _or_
3. Generate a magic link in code and include it in your own method

### Installation

```
npm install @mesmotronic/passport-magic-login
```

### Frontend usage

This is what the usage from the frontend might look like once you've set it all up. It only requires a single request:

```JS
// POST a request with the users email or phone number to the server
fetch(`/auth/magiclogin`, {
  method: `POST`,
  body: JSON.stringify({
    // `destination` is required.
    destination: email,
    // However, you can POST anything in your payload and it will show up in your verify() method
    name: name,
  }),
  headers: { 'Content-Type': 'application/json' }
})
  .then(res => res.json())
  .then(json => {
    if (json.success) {
      // The request successfully completed and the email to the user with the
      // magic login link was sent!
      // You can now prompt the user to click on the link in their email
      // We recommend you display json.code in the UI (!) so the user can verify
      // that they're clicking on the link for their _current_ login attempt
      document.body.innerText = json.code
    }
  })
```

### Backend setup

To make this work so easily, you first need to setup passport-magic-login:

```JS
import MagicLoginStrategy from "passport-magic-login"

// IMPORTANT: ALL OPTIONS ARE REQUIRED!
const magicLogin = new MagicLoginStrategy({
  // Used to encrypt the authentication token. Needs to be long, unique and (duh) secret.
  secret: process.env.MAGIC_LINK_SECRET,

  // The authentication callback URL
  callbackUrl: "/auth/magiclogin/callback",

  // Called with the generated magic link so you can send it to the user
  // "destination" is what you POST-ed from the client
  // "href" is your confirmUrl with the confirmation token,
  // for example "/auth/magiclogin/confirm?token=<longtoken>"
  sendMagicLink: async (destination, href) => {
    await sendEmail({
      to: destination,
      body: `Click this link to finish logging in: https://yourcompany.com${href}`
    })
  },

  // Once the user clicks on the magic link and verifies their login attempt,
  // you have to match their email to a user record in the database.
  // If it doesn't exist yet they are trying to sign up so you have to create a new one.
  // "payload" contains { "destination": "email" }
  // In standard passport fashion, call callback with the error as the first argument (if there was one)
  // and the user data as the second argument!
  verify: (payload, callback) => {
    // Get or create a user with the provided email from the database
    getOrCreateUserWithEmail(payload.destination)
      .then(user => {
        callback(null, user)
      })
      .catch(err => {
        callback(err)
      })
  }
  
  
  // Optional: options passed to the jwt.sign call (https://github.com/auth0/node-jsonwebtoken#jwtsignpayload-secretorprivatekey-options-callback)
  jwtOptions: {
    expiresIn: "2 days",
  }
})

// Add the passport-magic-login strategy to Passport
passport.use(magicLogin)
```

Once you've got that, you'll then need to add a couple of routes to your Express server:

```JS
// This is where we POST to from the frontend
app.post("/auth/magiclogin", magicLogin.send);

// The standard passport callback setup
app.get(magicLogin.callbackUrl, passport.authenticate("magiclogin"));
```

That's it, you're ready to authenticate! 🎉

#### Creating a magic link without POST

If you need to create a magic link without POSTing data to the API you can use the `create` method, for example:

```JS
const sendCustomerNotification = async (destination: string) => {
  // The `create` method generates a magic link for the specified destination (email) 
  // and returns a URL and verification code 
  const { href, code } = magicLogin.create(destination);

  await sendEmail({
    to: destination,
    body: `You have a new document! Click this link to login: https://yourcompany.com${href} (verification code ${code})`
  });
};

sendCustomerNotification('customer@email.address');
```

## License

Licensed under the MIT license. See [LICENSE](./LICENSE) for more information!
