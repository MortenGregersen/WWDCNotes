# Meet Apple Maps Server APIs

Simplify your app’s mapping architecture by implementing the Apple Maps stack across MapKit, MapKit JS, and Apple Maps Server APIs. Learn how these APIs can reduce network calls and increase power efficiency, which can help improve the overall performance of your app. We'll show you how to use geocoding and estimated time of arrival APIs to build functionality for a simple store locator, and explore the API authentication flow.

@Metadata {
   @TitleHeading("WWDC22")
   @PageKind(sampleCode)
   @CallToAction(url: "https://developer.apple.com/wwdc22/10006", purpose: link, label: "Watch Video (13 min)")

   @Contributors {
      @GitHubUser(multitudes)
   }
}



Apple Maps app offers various end-user experiences to Apple customers around the globe.

![MapKit][maps]

[maps]: WWDC22-10006-maps  

![MapKit JS][mapkitajs]

[mapkitajs]: WWDC22-10006-mapkitajs

New! introducing the Apple Maps Server APIs.

![MapKit server APIs][server]
  
[server]: WWDC22-10006-server

![MapKit server APIs][three]
  
[three]: WWDC22-10006-three

These APIs will help integrating Maps into apps. With Geocoding APIs, an address an be converted to geographic coordinates latitude and longitude. Similarly, with Reverse Geocoding, we can do the opposite -- go from geographic coordinates to an address. With Search API, we enter a search string to discover places like businesses, points of interest...
The ETA API can help find the closest store.

![the stack][fullStack]

[fullStack]: WWDC22-10006-fullStack

A benefit is the reduction in network calls.  
Many times, users are making repetitive and redundant requests, maybe looking up the same address over and over again from an app running on different user devices. This causes a lot of network calls and wasted bandwidth.  
Delegating this common operation to a server and doing it only once in the back end using server APIs will help consume less bandwidth and be power efficient too.

# Let's take some of these APIs for a spin.

Example: Building contact cards for a store locator application. There are three stores with their addresses and distance from the customer location.

![the comic Book][comicBook]

[comicBook]: WWDC22-10006-comicBook

Let's assume that these addresses are on a server which stores and serves the locations of comic bookstores.  
There are many ways to build this, but for a second, let's assume we don't have these new server APIs.
What would a basic architecture look like? How would a client application get this data? In this diagram, our application is making a call to the server to get the list of store addresses.

![Basic Architecture][basicArchitecture]

[basicArchitecture]: WWDC22-10006-basicArchitecture

The back-end server returns a list of store addresses to your client device.  
Since we don't have the server APIs in this example, now our client application has to perform various actions on the address to build the contact card.  
To perform a single task, a client may have to make multiple calls to various back-end services.

Here we can see that the client app is making a call directly to the Apple Maps Server, either by using MapKit or MapKit JS. This chattiness between a client and a back end can adversely impact the performance and scale of the application.  

Over a cellular network with typically high latency, using individual requests in this manner is inefficient and could result in broken connectivity or incomplete requests. While each request may be done in parallel, the application must send, wait, and process data for each request all on separate connections increasing the chance of failure.  
Finally, all the responses on the client will have to be merged. And while all these calls happen, the user is seeing a spinner. Plus, the client device is using more bandwidth and power for these extra calls.

That is not a good user experience.

Now, let's look at a model architecture with access to Apple Maps Server APIs. Start using our back-end server as a gateway to reduce chattiness between the client and the services.

![Server Aggregation][serverAggregation]
  
[serverAggregation]: WWDC22-10006-serverAggregation

Just like before, we request a list of stores to be displayed from our client. Next, we make a request from the server to do geocoding. We then receive responses for each API from the Apple Maps Server.  

The comic book server combines the responses from each service and sends the response to the application.  
This pattern can reduce the number of requests that the application makes to back-end services, and improve application performance over high-latency networks.

In summary, our client makes one call to our server to get the list of stores. Our server then does the heavy lifting to make appropriate API calls to compose a response most suited for your user.

# Use Geocoding and ETA API to get the distance to the store.

We can use the Geocode API to find the latitude and longitude for the store addresses, which we'll later use for ETA calculations.

In this example, first, we are going to take the address for the comic book store and URL encode it.

![Geocoding Example][fetch]
  
[fetch]: WWDC22-10006-fetch

Next, we'll use the Geocode API and pass this URL-encoded address as a query parameter. (skipping over the authentication details for now). In the response, you can see the latitude and longitude for the address returned.

![Fetch Coordinates][fetchCoordinates]
  
[fetchCoordinates]: WWDC22-10006-fetchCoordinates

We'll repeat the same process to find the latitude and longitude for the customer's address. This will be later used for ETA calculations.

There are more fields in the response. The detailed documentation is in the Resources section below.  

Let's set the origin and destination on the ETA API with the data we got from the Geocode API.

![fetching ETA][ETA2]
  
[ETA2]: WWDC22-10006-ETA

We have the origin latitude, longitude and the destination latitude, longitude.
Specify up to 10 destinations here if needed, and feeding that in the ETA API as origin and as destination query parameters which are URL encoded. The response to the API is a list of ETAs, one for each destination provided.

![fetching ETA][fetchingETA]
  
[fetchingETA]: WWDC22-10006-fetchingETA

In this case, we only have one since we provided one destination.  
Here for our example, we are interested in distanceMeters to calculate the distance to the store and we have all the pieces we need: the store address and the distance for the user to reach the store. we can also choose to augment or overlay this data with our store information, like store hours.

# Authentication

One critical piece is authentication.

All the Apple Maps Server APIs are authenticated.  

Using MapKit JS, we are already halfway there. Apple Maps Server APIs use the same mechanism as MapKit JS to authenticate.

![Authentication flow][authflow]
  
[authflow]: WWDC22-10006-authflow

- First, download our private key from our developer account.  
- Then use this private key to generate a Maps auth token in JWT format. (There is a detailed doc about how to generate one linked below.)  
- Exchange this Maps auth token using the token API to get Maps access token.  
- Authenticate the Maps auth token on the back end and send back Maps access token.  
- This is in JWT format and will be used for all API interactions.  
- This access token needs to be refreshed every 30 minutes by repeating the highlighted process here.

Here is a simple example of how to use the token API to fetch the access token.

![accessToken][accessToken]
  
[accessToken]: WWDC22-10006-accessToken

We are using the token API here, passing the Maps auth token as a header. Getting back a Maps access token that can be used to access the API. This will be in JWT format and will have standard fields like expiry, issuedAt, etc.

As a convenience, the expiresInSeconds field shows for how long the token is valid for. In this case, it's 30 minutes.

Maps auth token is not the same as Maps access token, we exchange the Maps auth token to get a 30-minute long Maps access token to access the server APIs.

# The API interaction with Maps access token 

![API interaction with Maps access token][authflow2]
  
[authflow2]: WWDC22-10006-authflow2

We'll pass the Maps access token along with server API call.  
It is added as a header to the API call, just like we saw a few slides ago.  
The Apple Maps Server will validate the Maps access token.  
Once the validation is successful, the Apple Maps Server will respond with an API response.

# Usage limits

There is a daily cap on how many API calls the app can make, and it's big! 

![daily cap][25000]
  
[25000]: WWDC22-10006-25000

Devs will get a quota of 25,000 service calls per day in total.

Calling services via MapKit JS and server APIs use the same quota.
Developers can view usage stats at the Maps developer dashboard.

![dashboard][devDash]
  
[devDash]: WWDC22-10006-devDash

If using MapKit JS, the server API usage is categorized as Services, which we can see highlighted here.


# Exceeding Quota

When the daily quota is exceeded, which means more than 25,000 server API calls, Apple will start rejecting new service calls and respond with HTTP status 429, which means too many requests.

![Exceeding Quota][exceedingq]

[exceedingq]: WWDC22-10006-exceedingq

ddevelopers should make sure that the app experience degrades gracefully in such scenarios.  
In rare scenarios, when our services makes an unusual amount of requests -- maybe it's due to some bug in code or infrastructure -- it's possible to get HTTP status 429 as well.  
When we receive HTTP 429, it is important not to simply loop repeatedly in making requests.

A better approach is to retry with increasing delays in between attempts.  
This approach is known as exponential backoff.

# Wrapping up

Four new server APIs: Geocoding, Reverse Geocoding, Search, and ETA.  

Full stack implementation, using these APIs in conjunction with MapKit and MapKit JS.

Redundant and repetitive calls can be optimised by delegating those tasks to the back-end server using Apple Maps Server APIs.

Daily quota for these APIs is 25,000 and is shared with the developer's MapKit JS service usage.

# Check out also 

[Meet MapKit for SwiftUI -  WWDC23](https://developer.apple.com/wwdc23/10043)  
[What's new in MapKit - WWDC22](https://developer.apple.com/wwdc22/10035)   
[What's new for enterprise developers](https://developer.apple.com/videos/play/tech-talks/110356)  
[Apple Developer: MapKit JS](https://developer.apple.com/maps/mapkitjs/)  
[Apple Maps Server API](https://developer.apple.com/documentation/applemapsserverapi)  
[Creating a Maps identifier and a private key](https://developer.apple.com/documentation/mapkitjs/creating_a_maps_identifier_and_a_private_key)  
[Creating and using tokens with MapKit JS](https://developer.apple.com/documentation/mapkitjs/creating_and_using_tokens_with_mapkit_js)  
[Maps for Developers](https://developer.apple.com/maps/)  
[Maps Server API test environment](https://maps.developer.apple.com/maps-server-api-playground)  

