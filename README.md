## Kyle Petan Code Review

I am going to show you how I figured on a way to deal with paged results from the facebook api.

An api call looked like this. Boy that's a long query string.
```Javascript
function apiFacebookPosts(item, accessToken, since = '15-12-1') {
  return fetch(`https://graph.facebook.com/v2.5/${item}?				fields=name,posts{story,created_time,id,message,comments,picture,likes},likes.since(${since}).limit(33){likes.limit(50),id,message,picture,created_time,comments}&access_token=${accessToken}`);
}
```

The React component called a function getFacebookPosts in its componentWillMount method.

In getFacebookPosts function I defined two helper functions: getMoreLikes and getMorePosts. These two functions deal with the case when the facebook api offers more posts than the built-in query limit and the total posts are under 99 (an arbitary limit). The getMoreLikes worked similarly.

```Javascript
getFacebookPosts() {
  const facebookUserId = window.facebook_id; // Django set this.
  const facebookAccessToken = window.fb_token; // And this.
  const currentDate = new Date();

  currentDate.setMonth(currentDate.getMonth() - 1);
  const date = currentDate.toISOString().slice(2, 10); // Date to use for since field of facebook query

  const getMoreLikes = (address, id, count) => {
    fetch(address)
    .then(
      response => response.json(),
        err => console.error(err))
      .then(json => {
        const likeCount = count + json.data.length;
        if (json.data.length >= 50 && json.paging.next) { // 50 like limit per request
          // Keep getting more Likes!
          getMoreLikes(json.paging.next, id, likeCount);
        } else {
          // Update Post with the current id's like count.
          const post = this.state.facebookPosts.find(item => {
            return item.id === id;
          });
          const oldState = this.state.facebookPosts.filter(item => {
            return item.id !== id;
          });
          const newPost = Object.assign({}, post, { 'totalLikes': likeCount });
          this.setState({ facebookPosts: [
            ...oldState,
            newPost
          ] });
        }
      });
  };

  const getMorePosts = (address) => {
    fetch(address)
    .then(
      response => response.json(),
        err => console.error(err))
      .then(json => {
        json.data.map(
          item => {
            const newItem = {};
            newItem.totalLikes = item.likes.data.length;
            newItem.message = item.message;
            newItem.created = item.created_time;
            newItem.picture = item.picture;
            newItem.id = item.id;
            this.setState({ facebookPosts: [...this.state.facebookPosts, newItem] });
            if (item.likes.data.length >= 50) {
              getMoreLikes(item.likes.paging.next, item.id, item.likes.data.length);
            }
          });
        if (this.state.facebookPosts.length < 98 && json.posts.data.length >= 33) { // Our users do not want more than 98 posts to be loaded (enough is enough). Our api request limits to 33 posts per request
          getMorePosts(json.paging.next);
        } else {
          this.setState({ facebookReady: true });
        }
      });
  };

  if (window.facebook_id) { // this is false if django has no facebook_id associated with user
    apiFacebookPosts(facebookUserId, facebookAccessToken, date)
    .then(
      response => response.json(),
        err => console.error(err))
      .then(
        json => {
          this.setState({ facebookName: json.name, facebookLikes: json.likes });
          json.posts.data.map(
            item => {
              const newItem = {};
              newItem.totalLikes = item.likes ? item.likes.data.length : 0;
              newItem.message = item.message;
              newItem.created = item.created_time;
              newItem.picture = item.picture;
              newItem.id = item.id;
              newItem.story = item.story;
              newItem.comments = item.comments ? item.comments.data.length : 0;
              this.setState({ facebookPosts: [...this.state.facebookPosts, newItem] }); // Object Spread Syntax
              if (item.likes && item.likes.data.length >= 50) {
                getMoreLikes(item.likes.paging.next, item.id, item.likes.data.length);
              }
            }
          );
          if (json.posts.data.length < 33) {
            this.setState({ facebookReady: true });
          } else {
            getMorePosts(json.posts.paging.next);
          }
        }
      );
  } else {
    this.setState({ facebookPosts: {} });
    this.setState({ facebookReady: true });
  }
}
```
That's that.

I am going to show you our request form, in abbreviated form.

One function showFilesUploaded allowed a user to select files (maybe images) from their file system and see them previewed graphically before submitting the form.

It was called by the onChange method of the input element concerned with uploading files.
```Javascript
showFilesUploaded(evt) {
  // https://developer.mozilla.org/en-US/docs/Using_files_from_web_applications
  const preview = window.document.querySelector('.Preview');
  while (preview.firstChild) {
    preview.removeChild(preview.firstChild);
  }
  const files = evt.target.files;
  for (let i = 0; i < files.length; i++) {
    const file = files[i];
    const imageType = /^image\//;
    if (!imageType.test(file.type)) {
      continue;
    }
    const img = document.createElement('img');
    img.classList.add('preview_img');
    img.file = file;
    img.width = 50;
    img.height = 50;
    preview.appendChild(img);
    const reader = new FileReader();
    reader.onload = (function(aImg) { return function(e) { aImg.src = e.target.result; }; })(img);
    reader.readAsDataURL(file);
  }
}
// ...
render() {
  return (
    //...
    <Input
      onChange={this.showFilesUploaded}
      className="file-upload-button"
      multiple="true"
      type="file"
      name="media"
    />
    <div className="Preview">
      {previewDisplay}
    </div>
    //...
  )
}

```

Finally, when using google analytics data. The process looked like this:

```Javascript
getGoogleAnalyticsData() {
  const that = this;
  const gapi = window.gapi;
  gapi.analytics.ready(() => {
    gapi.analytics.auth.authorize({
      'serverAuth': {
        'access_token': window.access_token
        // Access Token was created on the server by
        // supplying a client id and passcode stored in json file    
      }
    });
    this.visitsToday.bind(that)();
    this.visitsThisMonth.bind(that)();
    this.visitsThisYear.bind(that)();
    this.trafficByDevice.bind(that)();
    this.averageTimeOnSite.bind(that)();
    this.topLandingPages.bind(that)();
    this.sessionsByMedium.bind(that)();
    this.sessionsBySocialNetwork.bind(that)();
    this.newReturningSessionUsers.bind(that)();
    this.trafficByReferral.bind(that)();
    this.sessionsUsersTenDays.bind(that)();
    this.sessionsByCountry.bind(that)();
  });
  this.setState({ loading: false });
}
// ...
visitsToday() {
  const that = this;
  const gapi = window.gapi;
  gapi.client.analytics.data.ga.get({
    'ids': 'ga:XXX',
    'start-date': 'today',
    'end-date': 'today',
    'metrics': 'ga:sessions',
    'dimensions': 'ga:date'
  })
  .then(
    response => {
      const result = response.result.totalsForAllResults['ga:sessions'];
      // In this case instead of creating a graph, I just recieved the integer for display.
      that.setState({ visitsToday: parseInt(result, 10) });
    },
    () => window.setTimeout(this.visitsToday.bind(this), (Math.random() * 1000) + 2000)
    // Simulate a live stream of data by calling every 2-20 seconds.
  );
}
```
