# Crypton
A social network built on public-key encryption.  Described in more detail here: [http://jjg.2soc.net/post/crypton](http://jjg.2soc.net/post/crypton).

## Notes (mostly to myself)

The first thing we need to figure out is account setup.  What we know is that the user will be uploading their public key to the server, but the rest of the details is somewhat undefined.

The current thought is that once the user's public key has been sent to the server, the server will respond with a message encrypted using the key.  The new user will then decrypt the message out-of-band (the same way they'd decrypt any other OpenPGP message) and if sucessful, will get a link with a login token.  This establishes that the public key belongs to the user and not some other random person.  When the login link is accessed in a browser, some sort of long-running authentication state (a cookie, etc.) could be established to negate the need to go through the authentication process every time the system is accessed (this would of course be optional).

At this point the user is authenticated and has a browser open to the website.  Since there's not much to do all by yourself, it makes sense to walk them through adding some "trusted friends".  Initially this will be done by providing the public keys for each friend; eventually this will allow a keyring to be uploaded to speed up the process, assuming that can be done securely.  Another option might be to search a directory of known public keys (inclusion in the directory would of course be opt-in only).

Once some friends have been added a post to the feed is in order.  The post is entered into a textbox, and a "post" button is clicked to post the message.  Client-side script loops over the public keys in the trusted friends list and encrypts a copy of the post for each friend.  Each encrypted message is `POST`ed to the server API, and it is stored in the server's database.

These `POST` requests target each specific user, and the API endpoints are based on each user's public key fingerprint, something like `/users/D09A&nbsp;9058&nbsp;30EA&nbsp;B6D2&nbsp;E34C&nbsp;30D7&nbsp;F6AC&nbsp;697E&nbsp;0CC4&nbsp;A0DA/feed`.

So far so good, where it gets less clear to me at the moment is the read/decrypt side.

So, a user accesses the site and is logged-in due to the authentication persistence mentioned above.  The server returns a list of posts from the feed, in encrypted form. At this point the client decrypts the posts, and for this it needs the user's private key.  This is where things get tricky, because we have to protect the private key at all costs.  Looking at the `openpgp.js` docs we can read-in the private key as a string (armored) and then use it to decrypt, but we need to be absolutely sure there's no way the key can be lifted from the browser by a nefarious party (XSS attack, etc.).  At the moment I'm not sure how much of a risk this is, or how it might be carried out, but I'm making a note here  because of the criticality of it.

Additionally, we don't want to force the user to provide the private key every time we load a new encrypted message, so this needs to be managed by a) keeping the variable the key is stored in within scope or b) persisting the key somewhere the client code can get at.  B is definately more convinient, and means that we shouldn't have to ask for the private key ever again (on this computer), but again we need to be extremely careful that this doesn't open up the possibility of something else in the browser stealing the key.  One potential safety measure might be to require the key's passphrase between sessions, or on a periodic basis, so that we can persist the key in a more protected format.  This is the area which requires the most thought and consideration if the application is going to meet the privacy requirements that are expected of it.

Assuming the client-side code has access to the private key, the rest of the user activity amounts to pulling encrypted data (messages, posts, etc.) from the server, decripting it for display, then encrypting & transmitting new data back to the server.  One think I haven't explicitly stated is whether or not data such as the trusted friends list are encrypted.  I think that it's a good idea, since we have the means, to encrypt everyhting that's persisted or traverses the server, just to err on the safe side.  However there may be some unanticipated reason this isn't possible (some chicken-and-egg problem or race condition) so I don't want to rule-out the possibility of the need to store non-message data in unencrypted forms.

## Reference

Some random things that have beeen useful so far:

*  https://github.com/openpgpjs/openpgpjs/blob/master/README.md#getting-started
*  https://github.com/openpgpjs/openpgpjs/blob/master/test/general/openpgp.js
*  https://openpgpjs.org/openpgpjs/doc/module-openpgp.html
*  https://openpgpjs.org/openpgpjs/doc/module-key.html
*  https://github.com/openpgpjs/openpgpjs/issues/431

