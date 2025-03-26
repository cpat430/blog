# Next Draft Mode

## Problem

Due to our website caching content on build, it means that in order to see content changes replicated in the site involves a full rebuild of the website. This can be quite a pain point for our subject leads who are involved in creating content and want to see how the changes will look. The issue they are facing is that once they make some changes and would like to see the changes reflected, they need to ask the dev team to rebuild the website, and then wait until a developer is able to rebuild it. Sometimes the developer isn't able to release some things to production if they are sitting in staging, so we would have to create a new build using old data, or the subject leads would have to wait.

We don't want the content to be updated live for every user that is on the website as this would result in performance issues (slower navigation and page loads), as well as increasing the database load unnecessarily. Another issue here is that it would affect our SEO strength as content would no longer be available within the html delivered by the server to the client.

## Solution

### Implement Next's Draft Mode

Next has built in functionality for a 'Draft Mode', which essentially allows you to enable this. Enabling this mode sets a cookie on the clients side to indicate that they are in draft mode. When we are in draft mode, the `getStaticProps` and `getServerSideProps` are called at request time instead, meaning we can selectively decide what pages are able to make a new content request and ignore the cached content.

One thing to consider with this option is that we need to ensure that the users that can use this mode are specific users with the allowed roles. For now, anyone with the `SubjectLead` or `SuperAdmin` role are able to access this.

For implementing Draft Mode, we needed to add a couple API routes to our Next.js website.

`/api/draft.page.ts`

This page is responsible for checking the permissions of the user by checking the roles that the current user has, and then if they meet the permissions checks, then we would enable draft mode using `req.setDraftMode({ enabled: true })`. Then redirect the user back to the page they initialised this from.

`/api/disable-draft.page.ts`

This page is responsible for disabling draft mode. It doesn't need to make any permission checks. If they don't have draft mode enabled, and navigate here, nothing will change, and they will be redirected to where they should be.

### Indicating to the user they're in draft mode

A big user experience consideration was when a user does activate draft mode, we want the user to be aware they are in draft mode. To achieve this, we utilise the `isPreview` property from the `next/router`'s `useRouter` hook. This would be true when we are in draft mode on the client side (using the cookie that was set from the server API routes). This is shown at the top of the page as long as they are in draft mode. The banner includes a button that allows the user to disable draft mode when they are done with it. Otherwise, they can also disable in the same place where they enabled it in the account dropdown in the navigation bar.

![draft mode banner](/files/image.png)

### Worse performance in draft mode

Whilst in draft mode, the website performance is quite inhibited as we are having to fetch fresh data, and for each page that we navigate to, there is a new request made to the `getStaticProps` and `getServerSideProps` which will use the new content hierarchy to generate new props. Although it is much slower, the gain from being able to preview content changes in production without having to rebuild the site much outweighs this slight performance drop.

## Learnings

### Cloudfront Caching

After implementing draft mode, it would work in a local build as expected, however, once it was in our staging environment, it would no longer behave the same as it was in the local production build.

The cookie was being set as expected, and the cookie would work on the pages that used `getServerSideProps`. However, on any of the Static Site Generated pages, it would not render the draft mode banner to indicate that the user was currently in draft mode.

In the end, it turns out it was because of cloud front which would cache the page, and return the same page on the next render. To deal with this, we needed to add a cache policy which would consider the draft mode cookie as a part of the CloudFront cache key. After making this change, it worked as expected, and we didn't need to make any hacky type of changes which were being considered initially.

### Logging out keeping the cookie

Initially, when we would enable draft mode and then log out of an account. The cookie would still be set, meaning that if I log into an account that has draft mode access and log out; I would still have access to draft mode on an account that doesn't have access.

To deal with this, we just needed to ensure that we navigate to the disable-draft page when we log out, and then redirect the user to the logout page as a part of the usual flow.

Issues encountered

- Some pages not working with draft mode
  - thought it was because the api token was being missed
  - Was actually because cloudfront was caching the page, even when the draft mode cookie was set.
  - Had to update the cloudfront cache key to include the cookie from draft mode.

## Conclusion

Overall, this addition to the website will improve the subject leads quality of life for creating content and visualising these changes without having to wait for a rebuild from the dev team. This doesn't mean subject leads won't need rebuilds anymore, it will just mean that they can make quick checks to the content while they are in the process of adding it via our admin panel.
