libHN
=====

The definitive Cocoa framework for adding HackerNews to your iOS/Mac app. This mini library includes features such as grabbing Posts (including filtering by Top, Ask, New, Jobs, Best), Comments, Logging in, and Submitting new posts/comments!

---------------------

## Getting Started

Installing libHN is a breeze. First things first, add all of the classes in the top-level **libHN Classes** folder inside of this repository into your app. Done? Good. Now, just <code>#import "libHN.h"</code> in any of your controllers, classes, or views you plan on using libHN in. That's it. We're done here.

---------------------

## HNManager

**HNManager** is going to be your go-to class for using libHN. Every action flows through there - all web calls, session generation, etc. It's your conduit to HackerNews functionality. HNManager is a Singleton class, and has a <code>defaultManager</code> initialization that you should use to make sure everything gets routed correctly through the Manager.

---------------------

## Fetching Posts

Because of the way HackerNews is set up, there are two methods for getting posts. The first one <code>loadPostsWithFilter:completion:</code>, is your beginning method to retrieving posts based on a filter. So if you go to the [HN homepage](https://news.ycombinator.com/) this is what you'd get if you call this method and use <code>PostFilterTypeTop</code> as the PostFilterType parameter.

If you notice on the homepage, at the very bottom, there's a "More" button. Click that then look at the URL Bar. Notice the funky string that looks like this: "fnid=kS3LAcKvtXPC85KnoQszPW" at the end of the URL? HackerNews works on assigning an fnid, or basically a SessionKey, to determine what page you are going to and the authenticity of its request/response. This is used for every action on the site except for getting the first 25 links of any post type. This is where the second method comes in, <code>loadPostsWithFNID:completion:</code>, which takes in an FNID string to determine what posts should come next.

**loadPostsWithFilter**

This method takes in a PostFilterType parameter and returns an NSArray of <code>HNPost</code> objects. The various PostFilterTypes, and the types of posts you receive are listed below:

* PostFilterTypeTop - [HomePage](https://news.ycombinator.com/)
* PostFilterTypeAsk - [Ask HN](https://news.ycombinator.com/ask)
* PostFilterTypeNew - [Newest Posts](https://news.ycombinator.com/newest)
* PostFilterTypeJobs - [HN Jobs](https://news.ycombinator.com/jobs)
* PostFilterTypeBest - [Highest Rated Posts Recently](https://news.ycombinator.com/best)

And here's how to use this:

```objc
[[HNManager sharedManager] loadPostsWithFilter:(PostFilterType)filter completion:(NSArray *posts){
  if (posts) {
    // Posts were successfuly retrieved
  }
  else {
    // No posts retrieved, handle the error
  }
}];
```

**loadPostsWithFNID**

Now that you've gotten the first set of posts, use this method to keep retrieving posts in that Filter. The FNID parameter is mostly taken care of with the <code>postFNID</code> property of the HNManager. If you wanted to do something custom, you could pass in a string of your choosing here, but I recommend sticking with the default postFNID property. Every time you load posts with any of these two methods, the postFNID parameter is updated on the sharedManager.

```objc
[[HNManager sharedManager] loadPostsWithFNID:[[HNManager sharedManager] postFNID] completion:(NSArray *posts){
  if (posts) {
    // Posts were successfuly retrieved
  }
  else {
    // No posts retrieved, handle the error
  }
}];
```

**HNpost.{h,m}**

The actual HNPost object is fairly simple. It just contains the metadata about the post like Title, and the URL. There is a class method here that scans through the HTML passed in to return the array of posts that the two web methods above return. This is the low-level stuff that you should never have to mess with, but might be beneficial to pore over if you'd like to learn more or implement changes yourself.

```objc
// HNPost.h

// Enums
typedef NS_ENUM(NSInteger, PostType) {
    PostTypeDefault,
    PostTypeAskHN,
    PostTypeJobs
};

// Properties
@property (nonatomic, assign) PostType *Type;
@property (nonatomic,retain) NSString *Username;
@property (nonatomic, retain) NSURL *Url;
@property (nonatomic, retain) NSString *UrlDomain;
@property (nonatomic, retain) NSString *Title;
@property (nonatomic, assign) int Points;
@property (nonatomic, assign) int CommentCount;
@property (nonatomic, retain) NSString *PostId;
@property (nonatomic, retain) NSString *TimeCreatedString;

// Methods
+ (NSArray *)parsedPostsFromHTML:(NSString *)html FNID:(NSString **)fnid;
```

---------------------

## Fetching Comments

There's only one method to load comments, and naturally, it follows from loading the Posts. After you load your Posts, you can pass one in to the following method to return an array of <code>HNComment</code> objects. If you go to an AskHN post, you'll notice that the text is inline with the rest of the comments (separated by a text area for a reply), so I decided to include that self-post as the first comment in the returned array. You can tell what this is by using the <code>Type</code> property of the HNComment. The same goes for an HNJobs post. Sometimes, a Jobs post will be a self-post to HN, instead of an external link, so you can capture this data in the exact same way as a regular comment. If the Type == CommentTypeJobs, then you know you have a self jobs post.

The main reason I did this for AskHN and Jobs was to get any Link data out of the post, and to present things nicely to the user inline with any other comments inside my own personal app.

```objc
[[HNManager sharedManager] loadCommentsFromPost:(HNPost *)post completion:(NSArray *comments){
  if (comments) {
    // Comments retrieved.
  }
  else {
    // No comments retrieved, handle the error
  }
}];
```

**HNComment.{h,m}**

Similar to the HNPost object, HNComment features a handy class method that generates an NSArray of HNComments by parsing the HTML itself. Again, I'd look this over just to get a feel for how it works.

```objc
// HNComment.h

// Enums
typedef NS_ENUM(NSInteger, CommentType) {
    CommentTypeDefault,
    CommentTypeAskHN,
    CommentTypeJobs
};

// Properties
@property (nonatomic, assign) CommentType *Type;
@property (nonatomic, retain) NSString *Text;
@property (nonatomic, retain) NSString *Username;
@property (nonatomic, retain) NSString *CommentId;
@property (nonatomic, retain) NSString *ParentID;
@property (nonatomic, retain) NSString *TimeCreatedString;
@property (nonatomic, retain) NSString *ReplyURLString;
@property (nonatomic, assign) int Level;
@property (nonatomic, retain) NSArray *Links;

// Methods
+ (NSArray *)parsedCommentsFromHTML:(NSString *)html;
```

---------------------

## Logging In/Out

User related actions are a vital aspect of being part of the HackerNews community. I mean, if you can't be active in discussion or submit interesting links, then you might as well be a bystander. Unfortunately most HN Reader iOS/Mac apps neglect this part of the community and focus more on the interesting links themselves. There's a good reason for this - it's not trivial to implement; you have to think about Cookies and going through two web calls just to get a submission or comment to go through. It's annoying, and I've decided to make developers' lives easier by doing the annoying work myself and abstracting it away so you don't have to think about it again. It all starts with logging in.

The way HN operates in the browser is off of an HTTP Cookie. This Cookie is generated at login, and kept around for a pretty long time. Logging in on a different computer invalidates all Cookies for a user. Therefore, it's necessary to check if there's a cookie, and validate it before attempting to login. This can be accomplished with the <code>setCookie</code> method of the HNManager. It will find the Cookie on the device and attempt to validate it. If it does check out, the <code>SessionCookie</code> property of the HNManager will not be nil. If this property is nil, you're going to need to login and generate a new Cookie for your device (which will then subsequently set the SessionCookie property on success, as well as the SessionUser with the returned HNUser object). This can be done with the following method:

```objc
[[HNManager sharedManager] loginWithUsername:@"user" password:@"pass" completion:(HNUser *user){
  if (user) {
    // Login was successful!
  }
  else {
    // Login failed, handle the error
  }
}];
```

Logging out just deletes the SessionCookie property and the SessionUser property from memory, so you can't use them any more to make user-specific requests like submitting and commenting. Logging out is dead simple to implement.

```objc
[[HNManager sharedManager] logout];
```

---------------------

## Submitting a New Post!

Coming soon!

---------------------

## Replying to a Post/Comment!

Coming soon!

---------------------

## License

Coming soon!

---------------------
