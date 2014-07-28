myLock Easy Encryption
==========

**WARNING: THIS IS A PROOF OF CONCEPT AND SHOULD NOT BE USED FOR REAL SECRETS.**

Demo: https://diafygi.github.io/myLock/

myLock is a public key encryption website and library that lets you encrypt data
to be sent to others using only usernames and passphrases. No private keyfiles
are required to use myLock. Usernames can be published widely and any user can
encrypt a file that can only be decrypted by the usernames they have selected.

This project was heavily inspired by [miniLock](https://github.com/kaepora/miniLock),
who use the same base encryption libraries as myLock. Thanks guys!

##Why use myLock?

1. The only software requirements for encrypting and decrypting a myLock file is
a modern web browser. No additional software or plugins are needed. This allows
users to just visit the demo website above to encrypt and decrypt files. This
can hopefully lower the barrier to entry for people to start encrypting their
communications.

2. The demonstration website above is made to be [unhosted](https://www.unhosted.org/).
It can be saved to your computer (just right click and "Save As") and opened
directly with no need for an internet connection. No external files are required
and no server calls are made beyond the initial website page load (or no calls
at all if you are hosting the file locally).

3. This entire project is only ~1000 lines of code, and the core library is less
than 500 lines of code. It is meant to be self contained, easy to learn, and
easy to audit (I would love to have a security audit donated to the project).

4. Since myLock is open source it can be included with other software to
allow easy asymmetric encryption. I would love to see people create myLock
wrappers for existing email and social network APIs.

5. myLock is designed to be used anonymously. No personally identifiable
information is ever requested, and you can generate as many usernames as you
want. You can use myLock as disposable encryption by typing gibberish into the
passphrase for account creation, encrypt the file to the username you intend,
save the encrypted file, then log out.

##How it works

Public key encryption works by using a public key to encrypt a file, that then
can only be decrypted by the person who has the complementary secret key. This
means you can widely publish your public key (on your twitter profile, email
signature, personal website, etc.) and others can then encrypt files that only
you can decrypt.

Normally, public key encryption requires that the user keep a secret key saved
somewhere on your computer. However, myLock uses an algorithm that generates a
secret key from your username and passphrase, so that you only need the username
and passphrase to recreate the secret key (see [Drawbacks](#drawbacks)).

Another really cool feature of myLock is that your username contains your public
key. That means you can just publish your username wherever you want, and it is
all people need to encrypt files for you.

##Drawbacks

1. Unfortunately, since only the username and passphrase are used to derive your
secret key, it means your passphrase has to be strong, so myLock requires a
passphrase that is at least 40 characters to encourage users to use stronger
passphrases. We make it harder to brute force passphrases by salting them with
random bytes, but common phrases such as song lyrics and movie quotes should not
be used.

2. Since the username contains the public key for a user, it cannot be chosen by
the user. Instead, usernames are generated at the time of account creation. The
good thing about usernames is that they can be public. You don't have to worry
about keeping your username saved somewhere hidden, so you can post it somewhere
you can easily copy it when you need to log back in.

3. You need both your username and your passphrase to log back into your account
because your username contains the random bytes that was originally used as a
salt when you created your account. Without those bytes, the original secret key
cannot be recreated. Luckily, you don't have worry about keeping your username
secret so you can keep it somewhere you can easily copy/paste from.

##Technical details

###External libraries
myLock includes three external libraries:

* [TweetNaCl.js](https://github.com/dchest/tweetnacl-js/) - Encryption library
* [bs58.js](https://github.com/cryptocoinjs/bs58) (using the miniLock [variant](https://github.com/kaepora/miniLock/blob/master/src/js/lib/base58.js)) - Encoding library
* [scrypt-async.js](https://github.com/dchest/scrypt-async-js/) - Hashing libary

These libraries are included inline and minified in the myLock `index.html`. The
exact commit, and steps (if any) to recreate the minified code are commented
above each library in myLock's code.

###User creation steps

1. A user enters a 40+ character secret phrase.
2. This secret phrase is hashed using [`nacl.hash()`](https://github.com/dchest/tweetnacl-js#naclhashmessage) (uses SHA-512).
3. A random 8-byte salt is generated using [`nacl.randomBytes()`](https://github.com/dchest/tweetnacl-js#naclrandombyteslength) (uses window.crypto.getRandomValues).
4. The secret key is generated from the secret phrase hash and random salt using [`scrypt(salt, hash, 17, 8, 32, 1000, callback)`](https://github.com/dchest/scrypt-async-js/blob/master/README#L21) (uses scrypt).
5. A public key is derived form the secret key using [`nacl.box.keyPair.fromSecretKey()`](https://github.com/dchest/tweetnacl-js#naclboxkeypairfromsecretkeysecretkey) (uses curve25519).
6. A username is created by concating the public key and the salt encoded in base 58 (i.e. `myLock_<publicKey><salt>`).

###User login steps

1. A user enters their username and secret phrase.
2. The secret phrase is hashed using [`nacl.hash()`](https://github.com/dchest/tweetnacl-js#naclhashmessage) (uses SHA-512).
3. The username is broken into its `<publicKey>` and `<salt>` components.
4. A secret key is generated from the secret phrase hash and the given salt using [`scrypt(salt, hash, 17, 8, 32, 1000, callback)`](https://github.com/dchest/scrypt-async-js/blob/master/README#L21).
5. A public key is derived form the secret key using [`nacl.box.keyPair.fromSecretKey()`](https://github.com/dchest/tweetnacl-js#naclboxkeypairfromsecretkeysecretkey) (uses curve25519).
6. A username is created by concating the public key and the salt encoded in base 58 (i.e. `myLock_<publicKey><salt>`).
7. The recreated username is compared to the user's input to validate a successful login.

###File encryption steps

1. This assumes you have a secret key, your username, and the usernames of recipients.
2. A 32-byte random file key is generated using [`nacl.randomBytes()`](https://github.com/dchest/tweetnacl-js#naclrandombyteslength) (uses window.crypto.getRandomValues).
3. The file is symmetrically encrypted in chunks using [`nacl.secretbox()`](https://github.com/dchest/tweetnacl-js#naclsecretboxmessage-nonce-key) (uses xsalsa20-poly1305).
4. Nonces are generated by hashing the previous chunk's nonce (chunk size defaults to 200kB or less for the last chunk).
5. A json object that contains the file's filename, filetype, chunk size, and file key is encrypted with the public key of the each recipient and signed with your secret key using [`nacl.box()`](https://github.com/dchest/tweetnacl-js#naclboxmessage-nonce-theirpublickey-mysecretkey) (uses curve25519-xsalsa20-poly1305).
6. An ephemeral key pair is generated using [`nacl.box.keyPair()`](https://github.com/dchest/tweetnacl-js#naclboxkeypair).
7. A json object that contains the encrypted file information, and your username is encrypted with the public key of the each recipient and signed with the ephemeral secret key using [`nacl.box()`](https://github.com/dchest/tweetnacl-js#naclboxmessage-nonce-theirpublickey-mysecretkey) (uses curve25519-xsalsa20-poly1305).
8. A json object that contains the encrypted ephemeral information and ephemeral public key is encoded to base 58.
9. These encoded json objects (one per recipient) are prepended to the encrypted file using a period (".") to denote separation between headers (two periods ".." denote the end of all headers and the start of the encrypted file).
10. The resulting file contains asymmetrically encrypted headers and a symmetrically encrypted file.

###File decryption steps

1. This assumes you have a secret key and an encrypted file.
2. The file is scanned for headers (split by periods ".").
3. When a header is found, the ephemeral json object is decoded from base 58.
4. The ephemeral information is decrypted with the included public ephemeral key and your secret key using [`nacl.box.open()`](https://github.com/dchest/tweetnacl-js#naclboxopenbox-nonce-theirpublickey-mysecretkey).
5. If the ephemeral information fails to decrypt, that header is skipped and the next header is tried.
6. If all headers are skipped, the file cannot be decrypted.
7. If a header's ephemeral information is successfully decrypted, it is decoded to the json object that contains the sender's username and the encrypted file information.
8. The sender's public key is decoded from the sender's username.
9. The file information is decrypted with the sender's public key and your secret key using [`nacl.box.open()`](https://github.com/dchest/tweetnacl-js#naclboxopenbox-nonce-theirpublickey-mysecretkey).
10. The file information contains the chunk size, file size, and the file key of the encrypted file itself (as well as the file's filename and filetype).
11. The file chunks are decrypted with their calculated nonces and the file key using [`nacl.secretbox.open()`](https://github.com/dchest/tweetnacl-js#naclsecretboxopenbox-nonce-key).
12. The resulting decrypted file is combined with the filename and filetype into a Blob object.

###Encrypted file structure

The following encrypted myLock file has three recipients (i.e. three headers) and two chunks:

`<header>.<header>.<header>..<chunk><chunk>`

Headers are separated by a period (the headers are encoded in base 58, so periods don't appear), and the file starts after a double period ("..").

###Header structure

```
{
    "ephemeralPublicKey": <Base58>,
    "userInfoNonce": <Base58>,
    "userInfoEncrypted": {
        "sender": <String>,
        "fileInfo": {
            "fileInfoNonce": <Base58>,
            "fileInfoEncrypted": {
                "chunkSize": <Integer>,
                "fileName": <String>,
                "fileSize": <String>,
                "fileType": <MIME-type>,
                "fileKey": <Base58>,
                "fileNonce": <Base58>,
            } (encrypted with recipient.publicKey, fileInfoNonce, and sender.secretKey),
        },
    } (encrypted with recipient.publicKey, userInfoNonce, and ephemeral.secretKey),
}
```

##myLock core library API

####`myLock.setUsername(username, passphrase)`

Sets a username and secret key in the myLock object. If username is `undefined`,
a username is generated (can be retrieved via the `myLock.getUsername()`
function). Passphrases must be at least 40 characters.

####`myLock.onUsernameDone(error)`

Gets called when a username has been set. A successful completion will leave
error undefined. An unsuccessful completion will have an error string.

####`myLock.getUsername()`

Will return the username set in the myLock object. NOTE: there is not a way
to retrieve the secret key in the myLock object.

####`myLock.encrypt(filename, file, recipients)`

Encrypt a file for a list of recipients (filename is a string, file is a File
or Blob object, recipients is an array of usernames).

####`myLock.onEncryptProgress(progress)`

Gets called when progress has been made on encryption. The progress value is an
integer between 0 and 100.

####`myLock.onEncryptDone(file, error)`

Gets called when a file has been encrypted. A successful completion will leave
error undefined. An unsuccessful completion will have an error string. The file
is a Blob object.

####`myLock.decrypt(file)`

Decrypt a file (file is a File or Blob object).

####`myLock.onDecryptProgress(progress)`

Gets called when progress has been made on decryption. The progress value is an
integer between 0 and 100.

####`myLock.onDecryptDone(sender, filname, file, error)`

Gets called when a file has been decrypted. A successful completion will leave
error undefined. An unsuccessful completion will have an error string. The
sender is a string. The filename is a string. The file is a Blob object.

##Example

```javascript
var myLock = new myLockCore();

//fires on username creation or login
myLock.onUsernameDone = function(error){

    //errors are strings or undefined
    if(error !== undefined){
        console.log("Username error: " + error);
        return;
    }

    //get the created username
    var my_username = myLock.getUsername();
    console.log("Username: " + my_username);

    //fires on encrypt progress
    myLock.onEncryptProgress = function(progress){
        console.log("Encrypt progress = %s", progress);
    }

    //fires on encrypt completion
    myLock.onEncryptDone = function(file, error){

        //errors are strings or undefined
        if(error !== undefined){
            console.log("Encryption error: " + error);
            return;
        }

        //fires on encrypt progress
        myLock.onDecryptProgress = function(progress){
            console.log("Decrypt progress = %s", progress);
        }

        //fires on decrypt completion
        myLock.onDecryptDone = function(sender, filename, file, error){

            //errors are strings or undefined
            if(error !== undefined){
                console.log("Decryption error: " + error);
                return;
            }

            //print the contents of the file
            var reader = new FileReader();
            reader.onload = function(){
                console.log("Decrypted " + filename + " (" + file.size + " bytes) from " + sender);
                console.log(reader.result);
            };
            reader.readAsText(file);
        }

        //decrypt the file
        myLock.decrypt(file);
    }

    //encrypt "Hello World!" to myself
    myLock.encrypt(
        "hi.txt",
        new Blob(["Hello World!"], {type: "text/plain"}),
        [my_username]
    );
}

//create an account
myLock.setUsername(undefined, "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa");
```

##Demo

https://diafygi.github.io/myLock/

##License and Feedback

This project is released under the MIT license, but external libraries may be
licensed differently. This project is hosted on [Github](https://www.github.com/diafygi/myLock),
so please file bug reports and pull requests there.

