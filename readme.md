# hyperdom-loader

A magic asynchronous loader thingy for hyperdom.

# magic?

Ok, fair enough. This is better described with an example. Let's say you have a page with a search box, and you want to load the results as you type. You want to make an AJAX GET request to fetch the results and display them in your DOM. Each time the user types a new character, you want to get some new results. Conversely, each time the user _doesn't_ type a new character, you _don't_ want to load search results, which sounds pretty obvious. What `hyperdom-loader` can do is allow you to get the search results on each new render, but only if the search query has changed. This is much easier than attaching event handlers to the search box.

```js
var hyperdom = require('hyperdom');
var h = hyperdom.html;
var loader = require('hyperdom-loader');
var $ = require('jquery');

var searchResults = loader(function (query) {
  if (query) {
    return $.get('/search?q=' + encodeURIComponent(query));
  }
});

function render(model) {
  var results = searchResults(model.query);

  return h('div',
    h('label', 'Search', h('input', {type: 'text', binding: [model, 'query']})),
    results && model.query
      ? h('ol',
          results.map(function (result) {
            return h('li', result);
          })
        )
      : h('loading...')
  );
}

hyperdom.append(document.body, render, {});
```

The first time `searchResults` is called, it will start loading the results and return `undefined`. As soon as results come in hyperdom will refresh and `searchResults` will be called again, this time returning the results. As the search query changes, the new results will arrive and the page will be refreshed.

# routes

Another example is when you have client-side routes. Let's say, for example, that you have a blog engine and each blog entry is on its own route, naturally. Each time you get to a new blog URL, you want to load the corresponding blog entry. With `hyperdom-loader` you can load the blog entry in the render function, and it will do the magic of only loading a new blog entry when the URL changes. In this case you may want to use `{timeout: 0}` to ensure that when the URL changes, you get a "loading" page instead of the previous page.

```js
var hyperdom = require('hyperdom');
var h = hyperdom.html;
var loader = require('hyperdom-loader');
var $ = require('jquery');
var router = require('hyperdom-router');

router.start();

var postRoute = router.route('/post/:id');
var rootRoute = router.route('/');

var loadPost = loader(function (id) {
  return $.get('/api/post/' + encodeURIComponent(id));
}, {timeout: 0});

function render(model) {
  return h('div',
    rootRoute(function () {
      return h('h1', 'welcome to my blog!');
    }),
    postRoute(function (params) {
      var post = loadPost(params.id);
      
      if (post) {
        return h('.post',
          h('h1', post.title),
          h('article', post.body)
        );
      } else {
        return h('h1', 'loading...');
      }
    })
  );
}

hyperdom.append(document.body, render, {});
```

# api

```js
var loader = require('hyperdom-loader');

var loaderFn = loader(fn, [options]);
```

* `fn` - a function that returns a resource, or a promise of a resource. This is only called if the arguments passed to `loaderFn` are different to the last arguments passed.
* `loaderFn` - a function that returns either `undefined` if the promise returned from `fn` is still pending, or the fulfilled value of the promise when it has been fulfilled. If the promise is rejected with an error, then `loaderFn` will throw that error.
* `options.timeout` - `loaderFn` will continue to return the previous value even when new arguments are passed to it, however if the promise returned from `fn` takes more than `timeout` then `loaderFn` will start returning `undefined` again. Default is 140ms. Set this to 0 to return `undefined` as soon as the arguments have changed.
* `options.onfulfilled` - a function to be called when the promise returned from `fn` is fulfilled. By default refreshes the hyperdom view.
* `options.key` - a function that takes the arguments passed to `fn` and returns an array of values that are relevant in comparing one call with another. Set this to the string `'json'` if you want to compare arguments as rendered in JSON.
* `options.equality` - [**deprecated**, use `key` above] Default argument equality is by reference, set this to `json` if you want to compare arguments by comparing the JSON representation of the arguments.
* `options.loading` - a function that is called on the previous value while waiting for a new value to be fulfilled. This can be used to return (or throw) the previous value, or some variant to indicate that a new value will soon be available. The default behaviour is to always throw the previous error, or return the previous value if the new value has only been waiting for 140 milliseconds (`options.timeout`):

    ```js
      function loading(lastValue, loadingSince, isException) {
        if (isException) {
          throw lastValue;
        } else if (loadingSince < timeout) {
          return lastValue;
        }
      }
    ```

## reset

```js
loader.reset();
```

Resets the argument cache, so the next call will call `fn`. Do this after you know the result will be different, i.e. something has changed on the server.
