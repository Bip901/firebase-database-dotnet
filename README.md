# Purpose of this fork

This fork is meant to work around Godot 3's C# Web Export bugs (see [issue 101122](https://github.com/godotengine/godot/issues/101122) and [issue 100740](https://github.com/godotengine/godot/issues/100740)).

It does this by doing the following:

1. Replacing all `System.Net.Http.HttpClient` references with a new, similar interface, `IHttpClient`.
   This allows dependency injection, which allows a custom HttpClient implementation, and is also a better coding practice.
2. Allowing injection into all `Task.Delay` calls via `Firebase.TaskDelayProvider.Constructor`
3. Massively simplifying the package to further remove dependencies which don't play well with web exports. This does come at the cost of some convenience functions, see the commits for more details.

To use the new package, you must first set `Firebase.Database.Http.HttpClientProvider.Constructor` to a delegate that returns the concrete `HttpClient` implementation, otherwise you will get a `NullReferenceException`. For example:

```csharp
Firebase.Database.Http.HttpClientProvider.Constructor = GetClient;

public IHttpClient GetClient(bool allowAutoRedirect = false, TimeSpan? timeout = null)
{
	//return new MyHttpClientImpl(...);
}
```

# FirebaseDatabase.net
[![AppVeyor Build status](https://ci.appveyor.com/api/projects/status/ep8xw22cexktghba?svg=true)](https://ci.appveyor.com/project/bezysoftware/firebase-database-dotnet)

Simple wrapper on top of [Firebase Realtime Database REST API](https://firebase.google.com/docs/database/). Among others it supports streaming API which you can use for realtime notifications.

For Authenticating with Firebase checkout the [Firebase Authentication library](https://github.com/step-up-labs/firebase-authentication-dotnet) and related [blog post](https://medium.com/step-up-labs/firebase-authentication-c-library-8e5e1c30acc2)

To upload files to Firebase Storage checkout the [Firebase Storage library](https://github.com/step-up-labs/firebase-storage-dotnet) and related [blog post](https://medium.com/step-up-labs/firebase-storage-c-library-d1656cc8b3c3)

## Installation
```csharp
// Install release version
Install-Package FirebaseDatabase.net

// Install pre-release version
Install-Package FirebaseDatabase.net -pre
```

## Supported frameworks
.NET Standard 2.0 - see https://github.com/dotnet/standard/blob/master/docs/versions.md for compatibility matrix

## Usage

### Authentication

The simplest solution where you only use your app secret is as follows:

```csharp
var auth = "ABCDE"; // your app secret
var firebaseClient = new FirebaseClient(
  "<URL>",
  new FirebaseOptions
  {
    AuthTokenAsyncFactory = () => Task.FromResult(auth) 
  });
```

Note that using app secret can only be done for server-side scenarios. Otherwise you should use some sort of third-party login. 

```csharp
var firebaseClient = new FirebaseClient(
  "<URL>",
  new FirebaseOptions
  {
    AuthTokenAsyncFactory = () => LoginAsync()
  });

...

public static async Task<string> LoginAsync()
{
  // manage oauth login to Google / Facebook etc.
  // call FirebaseAuthentication.net library to get the Firebase Token
  // return the token
}
  
```

As you can se, the AuthTokenAsyncFactory is of type `Func<Task<string>>`. This is to allow refreshing the expired token in streaming scenarios, in which case the func is called to get a fresh token.

#### Using service account
See [this issue](https://github.com/step-up-labs/firebase-database-dotnet/issues/221#issuecomment-704583145)

### Querying

#### C#
```csharp
using Firebase.Database;
using Firebase.Database.Query;

' Since the dinosaur-facts repo no longer works, populate your own one with sample data in "sample.json"
var firebase = new FirebaseClient("https://dinosaur-facts.firebaseio.com/");
var dinos = await firebase
  .Child("dinosaurs")
  .OrderByKey()
  .StartAt("pterodactyl")
  .LimitToFirst(2)
  .OnceAsync<Dinosaur>();
  
foreach (var dino in dinos)
{
  Console.WriteLine($"{dino.Key} is {dino.Object.Height}m high.");
}
```

#### VB.net
```vbnet
Imports Firebase.Database
Imports Firebase.Database.Query

' Since the dinosaur-facts repo no longer works, populate your own one with sample data in "sample.json"
Dim client = New FirebaseClient("https://dinosaur-facts.firebaseio.com/")

Dim dinos = Await client _
    .Child("dinosaurs") _
    .OrderByKey() _
    .StartAt("pterodactyl") _
    .LimitToFirst(2) _
    .OnceAsync(Of Dinosaur)

For Each dino In dinos
    Console.WriteLine($"{dino.Key} is {dino.Object.Height}m high.")
Next
```

### Saving & deleting data

```csharp
using Firebase.Database;
using Firebase.Database.Query;
...
var firebase = new FirebaseClient("https://dinosaur-facts.firebaseio.com/");

// add new item to list of data and let the client generate new key for you (done offline)
var dino = await firebase
  .Child("dinosaurs")
  .PostAsync(new Dinosaur());
  
// note that there is another overload for the PostAsync method which delegates the new key generation to the firebase server
  
Console.WriteLine($"Key for the new dinosaur: {dino.Key}");  

// add new item directly to the specified location (this will overwrite whatever data already exists at that location)
await firebase
  .Child("dinosaurs")
  .Child("t-rex")
  .PutAsync(new Dinosaur());

// delete given child node
await firebase
  .Child("dinosaurs")
  .Child("t-rex")
  .DeleteAsync();
```

### Realtime streaming

#### C#
```csharp
using Firebase.Database;
using Firebase.Database.Query;
using System.Reactive.Linq;
...
var firebase = new FirebaseClient("https://dinosaur-facts.firebaseio.com/");
var observable = firebase
  .Child("dinosaurs")
  .AsObservable<Dinosaur>()
  .Subscribe(d => Console.WriteLine(d.Key));
  
```

#### VB.net
```vbnet
Imports System.Reactive.Linq
Imports Firebase.Database
Imports Firebase.Database.Query
' ...
Dim client = New FirebaseClient("https://dinosaur-facts.firebaseio.com/")

child _
  .Child("dinosaurs") _
  .AsObservable(Of InboundMessage) _
  .Subscribe(Sub(f) Console.WriteLine(f.Key))
```

```AsObservable<T>``` methods returns an ```IObservable<T>``` which you can take advantage of using [Reactive Extensions](https://github.com/Reactive-Extensions/Rx.NET)
