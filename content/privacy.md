+++
title = 'Privacy Policy'
type = 'legal'
draft = false
+++

This Privacy Policy explains how {{< cfg "app_name" >}} (the "Service"), operated by {{< cfg "entity" >}} ("we", "us", "our"), handles information when you sign in with your Google account and grant the Service access to Google data. By using the Service you agree to the practices described here.

## Who We Are

- **Service:** {{< cfg "app_name" >}}
- **Data controller:** {{< cfg "entity" >}}
- **Contact:** {{< cfg "contact_email" >}}

## Information We Access

When you authenticate with Google, you may authorize the Service to access the following, strictly through the Google OAuth scopes you approve on the consent screen:

- **Google account identity** (`openid`, `userinfo.email`, `userinfo.profile`) — your email address, name, and profile picture. Used to identify you and personalize your session.
- **Google Drive** (Drive scope) — files in your Google Drive that the feature you invoke needs to read or work with. Used only to perform the action you request during that session.
- **Gmail** (Gmail scope) — messages in your Gmail mailbox that the feature you invoke needs to read or act on. Used only to perform the action you request during that session.

We request only the scopes required for the feature you use, and you can decline or later revoke them at any time.

## How We Use the Information

- Information is processed **transiently, within your active session**, solely to provide the feature you requested.
- We do **not** store your Google data on our servers, we do **not** log the contents of your Drive files or Gmail messages, and we do **not** build advertising or user profiles from it.
- We do not use your Google data to train machine learning or AI models.

## Storage and Retention

The Service does not persist your Google user data. Access tokens are held only for the duration of your session and are discarded when the session ends. Nothing derived from your Google data is retained afterward.

## Sharing and Disclosure

We do not sell your data, and we do not share it with third parties, advertisers, or data brokers. We disclose information only where required by law or valid legal process.

## Google API Services User Data Policy (Limited Use)

The Service's use and transfer of information received from Google APIs will adhere to the [Google API Services User Data Policy](https://developers.google.com/terms/api-services-user-data-policy), including the **Limited Use** requirements. We only use Google user data to provide or improve user-facing features that are prominent in the Service's interface, and we do not transfer or use that data for advertising, credit assessment, or any purpose beyond those explicitly stated in this policy.

## Your Controls

You can review and revoke the Service's access to your Google account at any time from your Google Account permissions page: [https://myaccount.google.com/permissions](https://myaccount.google.com/permissions). Revoking access immediately stops the Service from accessing your Google data.

## Security

All communication with Google APIs occurs over encrypted transport (TLS/HTTPS). Because we do not retain Google user data at rest, there is no stored copy of that data to be exposed.

## Children

The Service is not directed to children under 13, and we do not knowingly collect data from them.

## Changes to This Policy

We may update this Privacy Policy from time to time. Material changes will be reflected by updating the effective date shown at the top of this page.

## Contact

Questions about this policy? Contact us at {{< cfg "contact_email" >}}.
