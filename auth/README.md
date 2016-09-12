# Firebase Auth

Before using the `Auth` service you need to configure the providers you want to use.

Each provider requires a different setup process. Individual guides are provided explaining how to achieve this.

## Google

This provider doesn't require special configuration since it is automatically configured when you create your Firebase project.

Follow these steps to enable it:

1. Open the [Firebase console](https://firebase.google.com) and select your project.
2. Click the `Auth` option in the left side menu.
3. Click the `SIGN-IN METHOD` button in the top menu and then select `Google` from the providers list.
4. Click the `Enable` toggle button and set it to `on` and then press the `Save` button.

The Google provider has been successfully enabled.

## Facebook and Twitter

* Click [here](./facebook) to read the Facebook setup process.
* Click [here](./twitter) to read the Twitter setup process.

## Email with Password and Anonymous
*Main article: [Email & Password Auth](./email)*

Firebase Auth can also work without using a Federated provider. Email and Anonymous auth have been separated into their own guide.

## Implementation (Federated Login)

Once you have configured one or more Federated providers you will be able to use them in your project.

Open or create a new project.

Open the file where you want to implement the Sign-In feature.

Add the following constants and variables:

```actionscript
private static const FIREBASE_API_KEY:String = "YOUR-FIREBASE-APIKEY";
private static const FIREBASE_CREATE_AUTH_URL:String = "https://www.googleapis.com/identitytoolkit/v3/relyingparty/createAuthUri?key="+FIREBASE_API_KEY;
private static const FIREBASE_VERIFY_ASSERTION_URL:String = "https://www.googleapis.com/identitytoolkit/v3/relyingparty/verifyAssertion?key="+FIREBASE_API_KEY;
private static const FIREBASE_REDIRECT_URL:String = "https://<YOUR-PROJECT-ID>.firebaseapp.com/__/auth/handler";
private var webView:StageWebView;		
private var sessionId:String;
private var requestUri:String;
```

Firebase uses Google Identity Toolkit for its Auth backend. Add one or more buttons and assign them an `EventListener` to them when they get pressed.

I recommend to use the following function if you are using several providers and call it with the provider of your choice:

```actionscript

private function facebookButtonHandler():void
{
    //The startAuth function only requires one parameter, a String with the domain corresponding to the provider you want to authenticate
    startAuth("facebook.com"); //Use this for Facebook
    startAuth("google.com"); //Use this for Google
    startAuth("twitter.com"); //Use this for Twitter
}

private function startAuth(provider:String):void
{			
    var header:URLRequestHeader = new URLRequestHeader("Content-Type", "application/json");

    var myObject:Object = new Object();
    myObject.continueUri = FIREBASE_REDIRECT_URL;
    myObject.providerId = provider;
			
    var request:URLRequest = new URLRequest(FIREBASE_CREATE_AUTH_URL);
    request.method = URLRequestMethod.POST;
    request.data = JSON.stringify(myObject);
    request.requestHeaders.push(header);
			
    var loader:URLLoader = new URLLoader();
    loader.addEventListener(flash.events.Event.COMPLETE, authURLCreated);
    loader.addEventListener(flash.events.IOErrorEvent.IO_ERROR, errorHandler);
    loader.load(request);			
}

//We also create an errorHandler since Firebase actually gives useful error codes and messages
private function errorHandler(event:flash.events.IOErrorEvent):void
{
	trace(event.currentTarget.data);
}
```

This function connects to the Google Identity Toolkit, it requires two parameters:

* `providerId` which is the domain of the prefered auth provider.
* `continueUri` which is the same URI used for configuring the providers.

Now we create the `authURLCreated` function where we will read the response:

```actionscript
private function authURLCreated(event:flash.events.Event):void
{
	var rawData:Object = JSON.parse(event.currentTarget.data);
			
	//We store the sessionId value from the response for later use
	sessionId = rawData.sessionId;

	webView = new StageWebView();
	webView.addEventListener(LocationChangeEvent.LOCATION_CHANGE, changeLocation);
	webView.stage = this.stage;
	webView.viewPort = new Rectangle(0, 0, stage.stageWidth, stage.stageHeight);
			
	//We load the URL from the response, it will automatically contain the client id, scopes and the redirect URL
	webView.loadURL(rawData.authUri);
}
```

We created and instantiated our `StageWebView` which will load an URL that has been dynamically created by Google Identity Toolkit.

We also saved the `sessionId` value since it will be used for a later step.

Now we add the `changeLocation` handler.

```actionscript
private function changeLocation(event:LocationChangeEvent):void
{			
	var location:String = webView.location;
						
	if(location.indexOf("/__/auth/handler?code=") != -1 || location.indexOf("/__/auth/handler?state=") != -1 || location.indexOf("/__/auth/handler#state=") != -1){
				
		//We are looking for a code parameter in the URL, once we have it we dispose the webview and prepare the last URLRequest	
		webView.removeEventListener(LocationChangeEvent.LOCATION_CHANGE, changeLocation);
		webView.dispose();
				
		requestUri = location;
		getAccountInfo();
	}           
}
```

Here is where things start getting hard since there's no official documentation. This is what is supossed to happen:

1. Once a successful authentication has been made, the StageWebView will change its URL to a 'success' page.
2. This success page contains a `code`, but thankfully we won't need to parse the code, only check if it exists.
3. After further testing I noticed that the success URL varies its form depending on the provider that was used.

* Facebook success URL code part looks like `?code=`
* Twitter success URL code part looks like `?state=`
* Google success URL code part looks like `#state=`

In the previous snippet I added the conditional to detect if the code exists in any of the 3 providers previously mentioned.

4. Once we have an URL that contains the `code` we save it to a String and then call our next function `getAccountInfo()`

```actionscript
private function getAccountInfo():void
{     
	var header:URLRequestHeader = new URLRequestHeader("Content-Type", "application/json");
			
	var myObject:Object = new Object();
	myObject.requestUri = requestUri;
	myObject.sessionId = sessionId;
			
	var request:URLRequest = new URLRequest(FIREBASE_VERIFY_ASSERTION_URL);
	request.method = URLRequestMethod.POST;
	request.data = JSON.stringify(myObject);
	request.requestHeaders.push(header);
			
	var loader:URLLoader = new URLLoader();
	loader.addEventListener(flash.events.Event.COMPLETE, registerComplete);
	loader.addEventListener(flash.events.IOErrorEvent.IO_ERROR, errorHandler);
	loader.load(request);		
}
```

We created another `URLRequest` with 2 parameters:

* `requestUri` is the URI that contains the `code`, this code will be parsed by the Google Identity Toolkit servers and then used to connect to their respective OAuth servers to retrieve the logged in user profile information.
* `sessionId` is from the very start when we requested the `authUri`.

Now we add the `registerComplete` function that will contain the logged in user information.

```actionscript
private function registerComplete(event:flash.events.Event):void
{
	trace(event.currentTarget.data);
    var rawData:Object = JSON.parse(event.currentTarget.data);
}
```

If everything was successful you will receive a JSON file with detailed information about the logged in user. You will be able to see the user on your [Firebase console](https://firebase.gogole.com) in the Auth section.

This information is formatted the same for all providers, the most important values are:

Name | Description
---|---
`localId`| A unique id assigned for the logged in user for your specific Firebase project. This is very useful when working with Firebase Database and Firebase Storage.
`idToken`| An Auth token used to access protected data and files from Firebase Database and Firebase Storage. For more information go to the Firebase Database and Firebase Storage guides.
`displayName`| The logged in user full name (Google and Facebook) or they handler in Twitter.
`photoUrl`| The logged in user avatar.
`email`| The logged in user email address.

Note that not all providers return the same information, for example Twitter doesn't return an Email Address.

Once you have the profile information you might want to save it on an Object that can be globally accessed, you will need it when performing Auth requests.