# apple-token-revoke-in-realm

**This tutorial has been adapted from @jooyoungho 's one on Firebase - thank you for your work, it has saved me several headaches and hours...**

This document describes how to revoke the token of Sign in with Apple in the MongoDB Realm environment.<br>
In accordance with Apple's review guidelines, apps that do not take action by June 30, 2022 may be removed.<br>

The whole process is as follows.
1. Get ```authorization_code```  when user logs in.
2. Get a ```refresh_token``` with no expiry time using ```authorization_code``` with expiry time.
3. After saving the ```refresh_token```, revoke it when the user decides to delete the account.

You can get a refresh token at https://appleid.apple.com/auth/token and revoke at https://appleid.apple.com/auth/revoke.

Follow the tutorial from Realm on how to set-up the Apple authentication: https://www.mongodb.com/docs/atlas/app-services/authentication/apple/#std-label-apple-id-authentication
**Important note:** I had some issues related to the client_id during the set-up and the revoke flow. Instead of the service_id as mentionned in this tutorial, I had to use the bundle_id to make it work.

## Getting started

If you have implemented Apple Login using Realm, you should have ASAuthorizationAppleIDCredential somewhere in your project.<br>
You will need the ```authorization_code``` first to get a ```refresh_token``` from Apple REST API.

```swift
SignInWithAppleButton(signButtonLabel, onRequest: { request in
			request.requestedScopes = [.email]
		}, onCompletion: { result in
			switch result {
			case .success(let authResults):
				guard let credentials = authResults.credential as? ASAuthorizationAppleIDCredential,
						let identityToken = credentials.identityToken,
						let identityTokenString = String(data: identityToken, encoding: .utf8)
				else { return }

				appleIdToken = identityTokenString
				print("Successfully signed in/up with Apple.")

				if let email = credentials.email {
					login(email, authCredentials: credentials)
				} else {
					login("", authCredentials: credentials)
				}
			case .failure(let error):
				print("Sign with Apple failed: \(error.localizedDescription)")
			}
		})
      
    private func login(_ email: String, authCredentials: ASAuthorizationAppleIDCredential) -> Void {
	
			var credentials: Credentials

			if !appleIdToken.isEmpty {
				credentials = Credentials.apple(idToken: appleIdToken)
			} else {
				credentials = Credentials.anonymous
			}

			app.login(credentials: credentials) { result in
				switch result {
				case .failure(let error):
					print("Login to Realm failed: \(error.localizedDescription)")
				case .success(let user):
					print("Successfully logged into Realm as \(user.id).")

					if let authorizationCode = authCredentials.authorizationCode,
						 let codeString = String(data: authorizationCode, encoding: .utf8) {
							print("The authorization code is: \(codeString)")
							// other steps will follow...
					}
				}
			}
	}
      
      
  ```
  
Once you get this far, you can get the ```authorization_code``` when the user log in.<br>
However, we need to get a refresh token through ```authorization_code```, and this operation requires a ```JWT token```, so let's do this with a Realm function.
Turn off Xcode for a while and go to your code in Realm App Services - functions.<br>

First, let's declare a function that creates a JWT globally. Install the required packages through the depedencies.<br>
If you do not know your ID information, please refer to the relevant issue: https://github.com/jooyoungho/apple-token-revoke-in-firebase/issues/1 <br>
If you don't have a key, you need to create a new one, and turn on the Sign in with Apple option to link it with the app.

Function ```makeJWT```
  ```javascript
 
 // makeJWT
exports = function(){
  const jwt = require('jsonwebtoken')

  let privateKey = "YOUR_PRIVATE_KEY" // it needs to be on one line with each return lines replaced by \n
  const originalPrivateKey = privateKey.replace(/\\n/g, '\n')

 
  //Sign with your team ID and key ID information.
  let token = jwt.sign({ 
  iss: 'YOUR_KEY_ID',
  iat: Math.floor(Date.now() / 1000),
  exp: Math.floor(Date.now() / 1000) + 120,
  aud: 'https://appleid.apple.com',
  sub: 'YOUR_CLIENT_ID'
  
  }, originalPrivateKey, { 
  algorithm: 'ES256',
  header: {
  alg: 'ES256',
  kid: 'YOUR_KEY_ID',
  } });
  
  return token;

};
  ```
  
The above function is returned by creating JWT based on your key information.<br>
Now, let's get the ```refresh_token``` with the ```authorization_code```.<br>
We will create a function called ```getRefreshToken```.

  ```javascript
  // getRefreshToken
 exports = async function(arg){
  const fetch = require("node-fetch");
  
  const client_secret = context.functions.execute("makeJWT");
    
  let body = {
    "code" : arg.code,
    "client_id" : "YOUR_CLIENT_ID",
    "client_secret" :  client_secret,
    "grant_type" : "authorization_code"
  }

var formBody = [];
for (var property in body) {
  var encodedKey = encodeURIComponent(property);
  var encodedValue = encodeURIComponent(body[property]);
  formBody.push(encodedKey + "=" + encodedValue);
}
formBody = formBody.join("&");
  
 const response = await fetch('https://appleid.apple.com/auth/token', 
 {
	method: 'post',
	body: formBody,
	headers: {'Content-Type': 'application/x-www-form-urlencoded'}
 })
 const data = await response.json();
  return data.refresh_token
}
  ```

When you call the above function, you get the code from the query and get a ```refresh_token```.
For code, this is the ```authorization_code``` we got from the app in the first place.
Before connecting to the app, let's add a revoke function as well.
Let's create another function called ```revokeToken```

  ```javascript
  // revokeToken
exports = async function(arg){
  const fetch = require("node-fetch");
  const refresh_token = arg.refreshToken
  const client_secret = context.functions.execute("makeJWT");
  
  let data = {
      'token': refresh_token,
      'client_id': "YOUR_CLIENT_ID",
      'client_secret': client_secret,
      'token_type_hint': 'refresh_token'
  };
  
  var formBody = [];
  for (var property in data) {
    var encodedKey = encodeURIComponent(property);
    var encodedValue = encodeURIComponent(data[property]);
    formBody.push(encodedKey + "=" + encodedValue);
  }
  formBody = formBody.join("&");
  
  await fetch("https://appleid.apple.com/auth/revoke", 
 {
	method: 'post',
	body: formBody,
	headers: {'Content-Type': 'application/x-www-form-urlencoded'}
 })
};
  ```

The above function revokes the login information based on the ```refresh_token``` we got.<br>

Now back to Xcode.<br>
Call the Functions address in the code you wrote earlier to save the ```refresh_token```.<br>
In the example, it is saved as UserDefaults, but for security reasons, iCloud Keychain is recommended.


  ```swift
  ...

  // Add new code below
  
 if let authorizationCode = authCredentials.authorizationCode, 
 		let codeString = String(data: authorizationCode, encoding: .utf8) {
	print("code string: \(codeString)")
					
	Task {
		if let user = app.currentUser {
			do {
				var response : AnyBSON
				let request = ("code", AnyBSON(stringLiteral: codeString))
				response = try await user.functions.getRefreshToken([AnyBSON(dictionaryLiteral: request)])
				if let refreshToken = response.stringValue {
					UserDefaults.standard.set(refreshToken, forKey: "refreshToken")
					UserDefaults.standard.synchronize()
				}
			} catch(let err) {
				print(err)
			}
		}
	}
  
  ...
  
  ```


At this point, the user's device will save the ```refresh_token``` as UserDefaults when logging in.
Now all that's left is to revoke when the user leaves the service.


  ```swift
    if let token = UserDefaults.standard.string(forKey: "refreshToken") {
			let request = ("refreshToken", AnyBSON(stringLiteral: token))
			if let user = app.currentUser {
				try await user.functions.revokeToken([AnyBSON(dictionaryLiteral: request)])
			}
		}
          
  ```

If we've followed everything up to this point, our app should have been removed from your Settings - Password & Security > Apps Using Apple ID.
