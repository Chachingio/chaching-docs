# Customer Portal Sign-In

Your customers sign in to the customer portal with a link sent to their email. There is no password.

## Send Your Customers to the Portal

Each account has its own sign-in page, at `/portal/{account_id}/sign-in` on the ChaChing app. Copy the full link from **Settings → Customer portal** in the dashboard and put it on your website or in your emails.

## How Sign-In Works

1. The customer enters their email on the sign-in page.
2. The page always reports that the request was received. It does not reveal whether the email belongs to a customer.
3. A known customer of an active account receives an email with a link to their portal. The link is valid for 2 hours.

## Not Part of the API Reference

The routes behind these pages (`POST /portal/login` and the `/portal/...` routes) serve the portal pages in your customers' browsers. They take no API key, are not in the [API Reference](./api.json), and are not meant for server-side integrations: every operation of the API Reference requires your secret key.

To send your own notifications to customers instead, see [Send Your Own Customer Notifications](./customer-notifications.md).
