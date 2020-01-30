# Let Users Know When You Have Updated Your Service Workers in Create React App

*Show an alert component when you have pushed a new service worker, allowing the user to update their page right away*

![](https://cdn-images-1.medium.com/max/2000/1*o-uGfk9X59ZU1mloaqxnfA.png)

Create React App (CRA) is great for developing progressive web apps (PWAs). It has offline/cache-first behaviour built in. It’s not enabled by default, but you can opt in. It uses service workers and has a lot of pitfalls you can read about from [official docs](https://create-react-app.dev/docs/making-a-progressive-web-app/).

This piece is going to show you how to trigger an alert (or toast or actually whatever component you want) when you have updated your service worker. Usually, this will be when your app has some new updates and you want the user to see them right away.

This piece assumes you have a new project made with CRA. If you don’t, you can do it easily with:

```javascript
npx create-react-app my-app
```

## Registering a Service Worker

If you navigate to `src/index.js` you find on the last line:

```javascript
serviceWorker.unregister();
```

Switch it to:

```javascript
serviceWorker.register();
```

And registering a service worker is pretty much done. If you deploy your app to an HTTPS-enabled site, it’ll be cached.

Remember that service-worker implementation in CRA works only in production. You can make sure it works by checking the offline checkbox from the Chrome DevTools Network tab and reloading your page.

It still shows your app!

## Is your updated Service Worker not visible?

Now comes the harder part. You add or change code in your app and deploy — but users aren’t seeing your updates. As the docs state:
> The default behavior is to conservatively keep the updated service worker in the “[waiting](https://developers.google.com/web/fundamentals/primers/service-workers/lifecycle#waiting)” state. This means that users will end up seeing older content until they close (reloading is not enough) their existing, open tabs.

What if you want users to see your new updates without having to close all tabs? CRA is providing that option, too.

In the `src/serviceWorker.js` is a function named `registerValidSW` that provides access to service-worker updates and success events via callbacks and also prints information about these events to the console. This is how you know when to show that the app is cached for offline use or there’s a newer version available.

{% gist https://gist.github.com/gglukmann/876826ae923056e569f1514e9d3b5f2b.js %}

The `registerValidSW` function takes in two arguments — the second one is the one we’re interested in. `config` can be an object that has `onSuccess` and `onUpdate` callbacks in it. You should wonder now how and where could we make such an object?

If you look where `registerValidSW` is called, you see that it comes from `export function register(config)`. This is the very same function that we saw on the last line in `src/index.js`. Now, we’re back in our own code and we could do something like:

```javascript
serviceWorker.register({
  onSuccess: () => store.dispatch({ type: SW_INIT }),
  onUpdate: reg => store.dispatch({ type: SW_UPDATE, payload: reg }),
});
```

When those functions are called, they dispatch a function, and you can do whatever you want with them, like show a message.

With `onSuccess` it’s easier — you can just show the alert somewhere on your page. Perhaps it says, “Page has been saved for offline use.”. With onUpdate, you want to let the user know there’s a newer version available, and you can add a button with “Click to get the latest version.”.

## Showing a user alert when page is first time saved for offline use

In the above example, I used Redux store to dispatch an action, and I have store set up with:

```javascript
const initalState = {
  serviceWorkerInitialized: false,
  serviceWorkerUpdated: false,
  serviceWorkerRegistration: null,
}
```

Now when dispatching `SW_INIT` type action, we change `serviceWorkerInitialized` state to `true` and can use this selector inside any React component.

In my `src/App.js` (or any other component), we get it from store with Redux Hooks:

```javascript
const isServiceWorkerInitialized = useSelector(
  state => state.serviceWorkerInitialized
);
```

And we can show an Alert when it is `true`:

```javascript
{isServiceWorkerInitialized && (
  <Alert text="Page has been saved for offline use" />
)}
```

![Alert when Service Worker is installed](https://cdn-images-1.medium.com/max/2000/1*dQ-oTIRtgnbd8PQ84cXJVA.png)<em><sup>Alert when Service Worker is installed</em></sup>

## Showing the user an alert and a button when a new version of Service Worker is available

Using the same pattern, we show alert components when service workers have been updated.

```javascript
{isServiceWorkerUpdated && (
  <Alert
    text="There is a new version available."
    buttonText="Update"
    onClick={updateServiceWorker}
  />
)}
```

This time we add an onClick function that’ll be triggered when clicking on the “Update” button inside the alert component. Because we want user to click on a button and get a new version of the app.

All the magic is inside `updateServiceWorker` function that we are going to create.

This example is using CRA v3, which has a little addition generated inside the `public/service-worker.js` file. (If you’re using an older version of CRA, I’ve created a solution for that, too — just write to me.)

{% gist https://gist.github.com/gglukmann/4d7c61bb55bbba28b8058d685a692bd7.js %}

`skipWaiting` is a function that forces your new service worker to become the active one, and next time the user opens a browser and comes to your page, they can see the new version without having to do anything.

You can read more about `skipWaiting` from [MDN](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerGlobalScope/skipWaiting). But this just forces your service worker to be the active one, and you see changes only next time. We want to ensure the user has a new version right now. That’s why we have to call it and then refresh the page ourselves — but only after the new service worker is active.

To call it, we need an instance of our new service worker. If you scroll back up to where we registered the service worker, you can see the `onUpdate` function had an argument called `reg`. That’s the registration object, and that’s our instance. This will be sent to the `serviceWorkerRegistration` property in the Redux store, and we can get our waiting SW from `serviceWorkerRegistration.waiting`.

This will be our function that’s called when the user presses the “Update” button inside the alert:

```javascript
const updateServiceWorker = () => {
  const registrationWaiting = serviceWorkerRegistration.waiting;

  if (registrationWaiting) {
    registrationWaiting.postMessage({ type: 'SKIP_WAITING' });

    registrationWaiting.addEventListener('statechange', e => {
      if (e.target.state === 'activated') {
        window.location.reload();
      }
    });
  }
};
```

Because the service worker is a worker and thus in another thread, to send any messages to another thread we have to use `Worker.postMessage` ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Worker/postMessage)). Message type is `'SKIP_WAITING'` as we saw from generated `public/service-worker.js` file.

And we create an eventListener that waits for our new service-worker state change, and when it’s activated, we reload the page ourselves. And that’s pretty much it.

Now the user can see there’s a newer version available, and if they want, they can update it right away.

![Alert when new Service Worker is available](https://cdn-images-1.medium.com/max/2000/1*4k3sIy_o9fICXdJug4wFtA.png)<em><sup>Alert when new Service Worker is available</sup></em>

## Conclusion

I think it’s good to let the user decide if they want a new version right away or not. They have the option to click on the “Update” button and get the new version or just ignore it. Then, the new version of the app will be available when they close their tabs and open your app again.

Thanks.

Here’s a link to the [example repository](https://github.com/gglukmann/cra-sw).
