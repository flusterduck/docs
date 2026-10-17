# Site builders and tag managers

If you can add a script tag to every page, Flusterduck works. That covers Shopify, Webflow, Wix, WordPress, Squarespace and Google Tag Manager. None of them needs a build step or a plugin from us.

Every section below uses the same tag. Replace the key with your site's publishable key (onboarding shows it, and Settings > Sites mints a new one if you lost it):

```html
<script src="https://flusterduck.com/d.js" data-key="fd_pub_xxxxxxxxxxxx" data-dnt="false" async></script>
```

Keep `async` and `data-dnt="false"` exactly as shown. Without `data-dnt="false"` the SDK drops every visitor whose browser sends Do Not Track, which is 30-50% of traffic on Firefox and Brave.

## Shopify

1. Online Store > Themes > the three-dot menu on your live theme > Edit code.
2. Open `layout/theme.liquid`.
3. Paste the tag right before `</body>` and save.

This covers your storefront: home, collections, product pages and the cart. Shopify doesn't run theme code on its hosted checkout, so friction inside checkout itself isn't visible to the SDK. Cart and product-page friction is, and that's where most stores lose people before checkout anyway.

## Webflow

1. Site settings > Custom code.
2. Paste the tag into **Footer code** and save.
3. Publish the site. Custom code only goes live on publish, and it doesn't run in the Designer preview.

Webflow only allows site-wide custom code on a paid site plan.

## Wix

1. Settings > Custom Code (under Advanced) > Add Custom Code.
2. Paste the tag, choose **All pages**, and place it in **Body - end**.
3. Apply, then publish.

Wix requires a premium plan and a connected domain before Custom Code is available.

## WordPress

Pick one:

- **A header and footer plugin** (WPCode or similar): add the tag to the site-wide footer.
- **A child theme**: print the tag from a `wp_footer` action in the child theme's `functions.php`.

Avoid pasting it into a parent theme's `footer.php`. The next theme update overwrites it and the install stops without any error. If a caching or optimization plugin combines or delays scripts, exclude `flusterduck.com/d.js` from it.

## Squarespace

1. Settings > Advanced > Code Injection.
2. Paste the tag into **Footer** and save.

Code Injection needs a plan that includes it.

## Google Tag Manager

1. New tag > Custom HTML. Paste the tag.
2. Trigger: **All Pages**.
3. Submit and publish the container.

The SDK finds its key even when GTM injects the tag dynamically, so nothing else is needed. If you fire it on a consent trigger instead of All Pages, it only collects from visitors who accept.

**Check my install** in onboarding reads your page's served HTML, and a GTM tag isn't in that HTML until the browser runs the container. So the check reports no tag even when it works. Open your site and click around instead: the first-signal screen updates the moment events arrive.

## Content-Security-Policy

If your site sends a CSP header, allow:

- `script-src https://flusterduck.com` (add `https://cdn.jsdelivr.net` if you use the Next.js `FlusterduckScript` component)
- `connect-src https://api.flusterduck.com`

## Pages behind a login

The SDK runs in your visitor's browser, so it works on signed-in pages the same as public ones. Two things can't see behind your login:

- **Check my install** in onboarding fetches your site's public URL. If that page redirects to a login screen, the check reports no tag even when the tag is there. Sign in and click around instead: the first-signal screen updates the moment events arrive.
- **Heatmap backdrops** are captured from public pages. For signed-in pages, upload your own screenshot from the heatmap view.

## Staging and production

Every event is stored under the environment of the key that sent it, and the key a site gets at creation is its production key. So a staging deploy carrying that tag counts as production: its sessions use your plan and its friction can open issues. Leave the tag off staging, or add staging as its own site if you want to watch it separately.

Events from `localhost` and other dev hosts, and from automated browsers, are dropped on a production key, so a local smoke test can't burn your session quota. Verify the install on the deployed site.

## More than one site

Each site gets its own publishable key. Grow includes 1 site, Scale 5 and Pro 10, and sessions are pooled across all of an organization's sites. Add sites in Settings > Sites.
