# Bulk Upload Issue

<details>
<summary>tl;dr</summary>

### Problem

Uploading flash card decks in production caused page freezing due to missing `getOne` and `getList` implementations, leading to repeated, failed data fetches.

### Investigation

Logs pointed to an issue in the React Admin front end, where `getList` wasn’t implemented for the resource, causing HTTP errors. A difference in query settings between staging and production exacerbated the issue by continuously fetching unavailable data in production.

### Solution

1. Implement `getOne` for the resource.
2. Align query client settings in production and staging.
3. Redirect users to the list page if data is missing to prevent error pages.

This resolves the freezing and data-fetching errors.

</details>

---

## Problem

When uploading any number of flash card deck flash cards in production, after making the request, the page would freeze, and wouldn't give the user any indication as to what has gone wrong.

## Error flow

On production

- Upload the csv of any number of rows.
- Click `Proceed`
- After bulk assign has completed, we redirect to show page
- By default show page calls `dataProvider.getOne(resource)`, which in this case, does not exist.
- `dataProvider.getOne(resource)` if the resource doesn't exist, the fallback is call `dataProvider.getList(resource)`.
- if `dataProvider.getList(resource)` doesn't exist, then the response is to throw an HttpError.
- Each time the error was received, it seems like that show page would automatically try refetch, and these requests would get stuck.

## Investigation

### React Admin or Backend?

On investigating this issue, we found that in DataDog we were getting an error that said

`console error: t: getList not implemented for resource: bulkAssignFlashCardDeckFlashCard`

Which to me implied that the issue was within the admin frontend itself, and not so much the backend flash card assigning part. To validate these thoughts, we went through DataDog to see what errors we were getting in the backend. Alas, the backend had no issues, and the flash card decks - flash card relationships were being correctly assigned.

Therefore, this meant we could focus on React Admin. First place to look was with how the show page interacts with the data provider.

Looking through the `dataProvider.getOne`, we can see that it falls back to using `dataProvider.getList`

```ts
default: {
  // Fallback to getList
  return dataProvider
	.getList(resource, {
	  filter: {
		withIds: [id],
	  },
	  pagination: null,
	  sort: null,
	} as unknown as GetListParams)
	.then((res) => ({
	  data: res.data[0],
	}));
}
```

After making this request to `getList`, if it doesn't find that the resource has been implemented, then we will see an HTTP Error be thrown.

```ts
default:
  throw new HttpError(`getList not implemented for resource: ${resource}`, 404);
```

This is the error we were seeing in the DataDog logs, hooray!

This meant that we needed to be able to allow the show page have an endpoint to call and that the ID's match up so that it doesn't throw another error.

After making this change, it is all working as expected locally, but why does this fix it locally?

### Difference between staging and production

For some time, it didn't make sense as to why production was throwing errors and dealing with the same situation differently than it was in staging. Firstly, we went through and checked the s3 bucket to make sure that we were deploying the same version for both staging and production. This was not an issue, as they were the same.

With some more investigating, the only difference in environment variable was the react query client.

`USE_NEW_QUERY_CLIENT`

This is used to control whether we pass our own React Query Client into the admin panel, or we use React Query with all the default values. The settings that we have adjusted for staging are that we have a stale time which is non-zero, this means that we won't immediately make a new request when we go to a new page. So in the case of the request on production (as described above), it seems with a staletime of 0 AND refetch on window focus being true, would ensure that we would keep fetching the data until we receive something valid. Which in this case where we are fetching for the `dataProvider.getOne` of a non-existent resource implementation and it was always returning an HTTP Error.

## Solution

Firstly, we want to implement a `getOne` for this bulk assign resource. We only need to return the id, and don't care about anything else.

Secondly, we want to make sure that production and staging behave the same, so we set the query client to true. (Later we can remove this feature flag)

Thirdly, if there is no data returned when they visit the page outside of an auto-redirect, then we just redirect them back to the list page. This prevents any random error pages showing for data that we can't fetch at different times.
