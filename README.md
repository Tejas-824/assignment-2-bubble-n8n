# Author Royalty Dashboard (Bubble + n8n)

A royalty dashboard I built in Bubble. The admin clicks a button, n8n asks Gemini to write a royalty summary for that author, and the result is saved in a Notification Log in Bubble.

## Links
- App: https://tejasworld024.bubbleapps.io/version-test/login (dev version, free plan)
- Admin: admin@bookleaf.com / Admin@123
- Author: priya.sharma@email.com / Author@123 (all authors use Author@123)
- n8n workflow file: n8n-workflow.json

## 1. Database
- **Author**: id, name, email, phone, city, joined date, linked user, and totals (books, earned, paid, pending, payout status)
- **Book**: details, dates, MRP, copies sold, royalty earned / paid / pending, last payout date, platforms, and the Author it belongs to
- **Notification Log**: author, status, email subject, email body, error message, completed time
- **User**: Bubble login, plus a role (author or admin) and a link to an Author

Why: one author has many books, so each Book points to its Author. The Notification Log is separate so every attempt has its own status. I stored the totals on Author so the admin list can filter them without recalculating.

## 2. Screens
- **Author**: sees only their own books, with details, royalty numbers and a badge: green (fully paid), yellow (pending, normal cycle), red (pending and last payout over 90 days ago, or never paid on a book older than 90 days), grey (still in production). Total earned, paid and pending are at the top.
- **Admin**: sees all authors with total books, earned and pending. Search by name, filter by city and payout status. "View details" opens the author's page with the same data and the Generate & Send button.
- **Privacy rules**: authors see only their own Author record and Books. Admin sees everything.

## 3. How Bubble and n8n work together
1. Admin clicks Generate & Send Royalty Summary.
2. Bubble creates a Notification Log row with status pending.
3. Bubble calls the n8n webhook with the log id, author id, name, next payout date and a text summary of the books.
4. n8n checks a secret header (x-api-key). Without it, the call is rejected.
5. n8n sends the data to Gemini and gets back a subject and body.
6. n8n replies to Bubble with status, subject and body.
7. Bubble saves them in the log, sets the status to Success and adds the time.

**Why I send the data in the webhook:** it is simpler and faster, n8n needs no access to Bubble's Data API, and Bubble already knows which author to send. The downside is that n8n doesn't see changes made after the click, which is fine here.

## 4. Prompt
Gemini acts as a BookLeaf support person: warm and professional. It greets the author by name, summarises each published book, then gives total earned, paid and pending. If money is pending, it explains the quarterly payout, the 45-day window and the Rs. 1,000 minimum, with the next payout date. If everything is paid, it says thanks. Books in production get a one-line update. It can only use the data I send and must not make up numbers.

## 5. Error handling
- **Security**: the webhook needs a secret header.
- **Double click**: Bubble only creates a log if none exists for that author, so a second click doesn't create a duplicate.
- **AI fails**: Message a model retries 3 times. If it still fails, an error branch in n8n replies "Failed". Bubble then sets the log to Failed with an error message.
- **n8n errors**: I turned on "Include errors in response" in the API Connector, so Bubble keeps going and marks the log Failed.
- **n8n never answers**: the log stays pending, so the admin can see it in the Notification Log.

## 6. What I could not do on the free plans
- **Callback to Bubble's API (assignment step 5)**: Bubble's API needs a paid plan. n8n got a 401 "upgrade your Bubble subscription". So instead of n8n updating the log, n8n replies to the webhook call and Bubble updates the log itself, with status and time.
- **CSV import**: needs a paid plan, so I added the data by hand and linked books to authors with a backend workflow.
- **Sending a real email**: not done (it was a bonus). The email is saved in the log.
- **Next payout date**: typed as a fixed date, not calculated.
- **Gemini free limits**: too many requests in a minute gives an error, so I added retry.
- **App is on the dev version**: "Upgrade to deploy" needs a paid plan.

## 7. Problems I ran into
- **Admin opened the author page**: the login always went to author_dashboard. I added a role condition to each redirect. It still failed because the saved role values had a hidden line break ("admin" + new line), so no comparison matched. I retyped the roles.
- **Admin saw no authors**: the "Admin sees all" privacy rule used the same role check, so it failed for the same reason.
- **Totals were empty**: the recalc workflow had only run for one author. I ran it for all authors once with a temporary button.
- **Status check**: n8n returns "Success" but Bubble was checking "success". Case matters.

