# Get started with Microsoft Graph in a Python app 

This article describes the tasks required to get an access token from Azure AD and call Microsoft Graph. It walks you through the [Microsoft Graph Connect Sample for Python](https://github.com/microsoftgraph/python3-connect-rest-sample) and explains the main concepts that you implement to use the Microsoft Graph API. The article describes how to access Microsoft Graph by using direct REST calls.

![Office 365 Python Connect sample screenshot](./images/web-screenshot.png)

##  Prerequisites

* [Python 3.5.2](https://www.python.org/downloads/)
* [Flask-OAuthlib](https://github.com/lepture/flask-oauthlib)
* [Flask-Script 0.4](http://flask-script.readthedocs.io/en/latest/)
* A [Microsoft account](https://www.outlook.com/) or an [Office 365 for business account](https://msdn.microsoft.com/en-us/office/office365/howto/setup-development-environment#bk_Office365Account)
* The [Microsoft Graph Connect Sample for Python](https://github.com/microsoftgraph/python3-connect-rest-sample)

## Register the application in Azure Active Directory

First, you need to register your application and set permissions to use Microsoft Graph. This lets users sign into the application with work or school accounts.

## Register the application

Register an app on the Microsoft App Registration Portal. This generates the app ID and password that you'll use to configure the app for authentication.

1. Sign into the [Microsoft App Registration Portal](https://apps.dev.microsoft.com/) using either your personal or work or school account.

2. Choose **Add an app**.

3. Enter a name for the app, and choose **Create application**.

	The registration page displays, listing the properties of your app.

4. Copy the application ID. This is the unique identifier for your app.

5. Under **Application Secrets**, choose **Generate New Password**. Copy the app secret from the **New password generated** dialog.

	You'll use the application ID and app secret to configure the app.

6. Under **Platforms**, choose **Add platform** > **Web**.

7. Make sure the **Allow Implicit Flow** check box is selected, and enter *http://localhost:5000/login/authorized* as the Redirect URI.

	The **Allow Implicit Flow** option enables the OpenID Connect hybrid flow. During authentication, this enables the app to receive both sign-in info (the **id_token**) and artifacts (in this case, an authorization code) that the app uses to obtain an access token.

	The redirect URI *http://localhost:5000/login/authorized* is the value that the OmniAuth middleware is configured to use once it has processed the authentication request.

8. Choose **Save**.

## Create OAuth client

Your app needs to register an instance of the Flask-OAuth client that you'll use to start the OAuth flow and get an access token. 

In the Connect sample, the following code (located in [*connect/__init__.py*](https://github.com/microsoftgraph/python3-connect-rest-sample/blob/master/connect/__init__.py)) registers the client with all of the required values, including the application id (client_id), application secret (client_secret) and the authorization URL used to authenticate the user.

```python
	# Put your consumer key and consumer secret into a config file
	# and don't check it into github!!
	microsoft = oauth.remote_app(
		'microsoft',
		consumer_key=client_id,
		consumer_secret=client_secret,
		request_token_params={'scope': 'User.Read Mail.Send'},
		base_url='https://graph.microsoft.com/v1.0/',
		request_token_url=None,
		access_token_method='POST',
		access_token_url='https://login.microsoftonline.com/common/oauth2/v2.0/token',
		authorize_url='https://login.microsoftonline.com/common/oauth2/v2.0/authorize'
	)
```

<!--<a name="authCode"></a>-->
## Receive an authorization code in your reply URL page

After the user signs in, the browser is redirected to your reply URL, the ```login/authorized``` route in [*connect/__init__.py*](https://github.com/microsoftgraph/python3-connect-rest-sample/blob/master/connect/__init__.py), with an access token in the response. The sample stores the token as a session variable.

```python
	@app.route('/login/authorized')
	def authorized():
		response = microsoft.authorized_response()
	
		if response is None:
			return "Access Denied: Reason=%s\nError=%s" % (
				request.args['error'], 
				request.args['error_description']
			)
	
		# Check response for state
		if str(session['state']) != str(request.args['state']):
			raise Exception('State has been messed with, end authentication')
		# Remove state session variable to prevent reuse.
		session['state'] = ""
			
		# Okay to store this in a local variable, encrypt if it's going to client
		# machine or database. Treat as a password. 
		session['microsoft_token'] = (response['access_token'], '')
		# Store the token in another session variable for easy access
		session['access_token'] = response['access_token']
		meResponse = microsoft.get('me')
		meData = json.dumps(meResponse.data)
		me = json.loads(meData)
		userName = me['displayName']
		userEmailAddress = me['userPrincipalName']
		session['alias'] = userName
		session['userEmailAddress'] = userEmailAddress
		return redirect('main')
```

<!--<a name="request"></a>-->
## Use the access token in a request to the Microsoft Graph API

With an access token, your app can make authenticated requests to the Microsoft Graph API. Your app must append the access token to the **Authorization** header of each request.

The Connect sample sends an email using the ```me/microsoft.graph.sendMail``` endpoint in the Microsoft Graph API. The code is in the ```call_sendMail_endpoint``` function in the [*connect/graph_service.py*](https://github.com/microsoftgraph/python3-connect-rest-sample/blob/master/connect/graph_service.py) file. This is the code that shows how to append the access code to the Authorization header.

```python
	# Set request headers.
	headers = { 
	  'User-Agent' : 'python_tutorial/1.0',
	  'Authorization' : 'Bearer {0}'.format(access_token),
	  'Accept' : 'application/json',
	  'Content-Type' : 'application/json'
	}
```

> **Note** The request must also send a **Content-Type** header with a value accepted by the Graph API, for example, `application/json`.

The Microsoft Graph API is a very powerful, unifiying API that can be used to interact with all kinds of Microsoft data. Check out the API reference to explore what else you can accomplish with Microsoft Graph.