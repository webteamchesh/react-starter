## 🍃 Contensis Delivery API

When writing your own backing code that makes calls to the Delivery API, we strongly recommend using the exports available in `contensis-react-base/util` package. They come pre-connected to the current Contensis project and uses a delivery client that is hooked up with handlers that allow cache invalidation to work for pages rendered in SSR. They have been updated to provide full TypeScript intellisense which is invaluable for both TypeScript and JavaScript developers.

### Strongly recommended approach

The methods available in these imports allow us to make the simple api calls we need to surface regular content that provides a wrapped instance of `contensis-delivery-api` [npm package](https://www.npmjs.com/package/contensis-delivery-api) JavaScript client that works with your app.

For the most reliable SSR experience we have created a new hook that can be called within any React component and ensures everything is sourced from the same component tree (and scoped to the current request in SSR).
```typescript
import { useDeliveryApi } from '@zengenti/contensis-react-base/util';
```

To use this in redux-saga backing code we need to pass the reference to the delivery api we obtained from the hook call with the action payload when we dispatch the action from within the component.

```tsx
const SomeReduxComponent = () => {
  const deliveryApi = useDeliveryApi();
  const dispatch = useDispatch();
  const myData = useSelector(state => state.myData);

  useEffect(() => {
    if (!myData) {
      dispatch({
        type: 'FETCH_DATA',
        deliveryApi
      });
    }
  }, []);


  return (<MyComponentToRender data={myData} />);
}
```
And in the redux-saga code we can access the api in the payload we provided from the action argument

```typescript
export const sagas = [
  takeEvery('FETCH_DATA', fetchData),
];

export function* fetchData(action: { deliveryApi: SSRContext['api'] }) {
  const { deliveryApi } = action;
  const query = getDataQuery();
  const payload = yield deliveryApi.search(query, 1);
  
  yield put({
    type: 'FETCHED_DATA',
    myData: payload?.items || [];
  });
}

```

A simpler (but less reliable) way to access the API utilities is by importing them directly:

> ⚠️ **We advise against this approach in favor of the context-based method described above.**
<br>The objects returned by the useDeliveryApi and useSSRContext hooks are tied to the current component tree and its rendering context. Under high concurrency, we've observed that using static imports (like the one above) can lead to incorrect SSR cache invalidation headers being generated for some concurrent requests.

``` typescript
import { deliveryApi } from '@zengenti/contensis-react-base/util';
```

The next example provides a similar Delivery API implementation except calls made client-side are cached locally until the next page reload preventing needless duplicate api calls.

We are now advising against this approach due to the same SSR caveat as above
``` typescript
import { cachedSearch } from '@zengenti/contensis-react-base/util';
```

Alternatively, calling `getClient()` will return a 'connected' instance of the normal [delivery api client](https://github.com/contensis/contensis-delivery-api#examples) so you could access any part of the Delivery API that might not be available in the existing methods like this

```tsx
import { useDeliveryApi } from '@zengenti/contensis-react-base/util';

const SomeComponent = () => {
  const deliveryApi = useDeliveryApi();
  const client = deliveryApi.getClient();

  return null;
}
```
You can see an example implementation here: https://gitlab.zengenti.com/starter-projects/react-starter/-/blob/master/src/app/redux/siteConfig/sagas.ts

Use any of these approaches that work best for your implementation.

### Routing Hooks (`onRouteLoad` & `onRouteLoaded`)

When using the `withEvents` wrapper, the routing hooks `onRouteLoad` and `onRouteLoaded` receive an `ssr` argument. This `ssr` object is the same `SSRContext` returned by the `useSSRContext` React hook in your component tree.

You can destructure this `ssr` object to access an `api` field, which provides the same browser-cached, environment-aware API functions available via the `useDeliveryApi` hook. These functions are fully compatible with SSR cache invalidation.

> We recommend passing the `ssr` object to any custom logic invoked within these routing functions - for example, when loading site configuration entries or triggering backend search sagas. 

This ensures that requests made during SSR include the necessary context for cache invalidation, while also providing a consistent developer experience whether code is running server-side or client-side. These runtime nuances are handled internally by the methods exposed through the `api` field.

You can see an example implementation here: https://gitlab.zengenti.com/starter-projects/react-starter/-/blob/master/src/app/routes/withEvents.ts

### Why it's recommended

#### Zero config

The imported method will already be connected to the default `ALIAS` and `PROJECT` provided in the `.env` file with the necessary credentials to make everything work. You don't need to do anything to instantiate and authenticate your Delivery client, these methods should just work with your app.

#### Cache invalidation

These methods contain special response handlers that, in pages that use server-side rendering in production, will ensure these pages are rendered with the necessary headers and keys that will allow the cached server-rendered content to become invalidated when any of the content requested by that page changes.

Any other method used to fetch and render data from the Delivery API will not contain the response handlers to handle server-side cache invalidation and as such cannot guarantee pages that were server-rendered will not become instantly invalidated when that requested content has changed in the CMS. Instead these pages could take up to 1 hour in production to reflect changes made to the content fetched directly from the API using other methods.

Pages that contain content that is rendered purely client-side is unaffected by this issue but using these methods ensure the cache invalidation will perform optimally should any of this content become server-rendered in the future.

#### Existing pattern

These exported methods have existed as long as we have been using React for writing our web apps. They have been used a lot across many implementations and have been developed and enhanced many times and are designed to work best with Contensis Cloud deployments.

We strongly urge you to refactor existing similar implementations to use these imported methods from `contensis-react-base/util` to get the best experience with deployments to Contensis Cloud.

#### Future-proofing

Having these wrapped methods available as imports from `contensis-react-base` package means everybody can get the same experience without updating tons of app code each time we want to introduce new features or need to fix any existing bugs.

#### Secure Delivery API

This feature is currently under development and is dependent on importing wrapped methods as above for fetching sensitive content from the Delivery API in your own backing code.
