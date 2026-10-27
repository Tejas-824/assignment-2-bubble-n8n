## Author Royalty Dashboard (Bubble + n8n)

This is a royalty dashboard I built in Bubble. The admin clicks a button, n8n gets Gemini to write a royalty summary for that author, and the result goes back into Bubble and gets saved in a Notification Log.

## Links

App: https://tejasworld024.bubbleapps.io/version-test/login (dev version, free plan)
Admin: admin@bookleaf.com / Admin@123
Author: priya.sharma@email.com / Author@123 (all authors use Author@123)
n8n workflow file: n8n-workflow.json

## 1. Database
Data type	What it stores
Author	author_id, name, email, phone, city, joined date, linked user, and totals (total_books, total_earned, total_paid, total_pending, payout_status)
Book	book_id, title, ISBN, genre, status, dates, MRP, copies sold, royalty earned / paid / pending, last payout date, print partner, platforms, and which Author it belongs to
Notification Log	which author, status (Pending / Success / Failed), email subject, email body, error message, completed time
User	Bubble’s login account, plus a role (author or admin) and a link to an Author

## Why I did it this way

One author has lots of books, so every Book points to its Author. It’s basically a foreign key.
The Notification Log is its own thing so each email attempt gets its own status.
I put the totals on Author so the admin list can show and filter them without recalculating each time.
A role on User was the easiest way to split admin and author access.
I kept author_id_text on Book as a plain text copy of the author ID. It helps me check the data and write privacy rules.
Books still in production have no MRP or dates. I left them empty on purpose.
## 2. Author and admin screens
Author: when an author logs in they only see their own books. Each book shows its details, royalty numbers and a colored badge:
Green: fully paid
Red: money pending and last payout was over 90 days ago, or never paid on a book older than 90 days
Yellow: money pending but still in the normal cycle
Grey: still in production
Earned, paid and pending totals sit at the top.
Admin: sees all authors with total books, earned and pending. You can search by name and filter by city and payout status. Click an author and you get a detail page with all their books and numbers.
Privacy rules: authors only see their own Books and their own Author record. Admin sees everything. Only admin can see the Notification Log. I tested it by logging in as different people. Priya sees 2 books, Sneha sees 3, admin sees all 18.
## 3. How Bubble and n8n talk to each other
Admin opens an author’s page and clicks Generate & Send Royalty Summary.
Bubble makes a Notification Log row with status Pending.
Bubble calls the n8n webhook using the API Connector. It sends the log id, author id, author name, and a text summary of their books.
n8n checks a secret header (x-api-key). No header, no entry, so the webhook isn’t open to anyone who finds the URL.
n8n passes the data to Gemini and gets back an email subject and body.
n8n then calls a Bubble backend workflow (update_notification_log) with an API token, which saves the subject and body and sets the status.
The log row flips from Pending to Success.

## Why I send the data in the webhook and don’t let n8n pull it from Bubble’s Data API

It’s simpler. n8n doesn’t need extra access to Bubble’s data.
It’s faster and there’s less that can break.
Bubble already knows which author and books to send.
The downside: if the data changes after the click, n8n won’t see it. That’s fine here.
## 4. Prompt strategy
I told Gemini to act like a BookLeaf support person, friendly but professional.
It starts with a greeting using the author’s name, then a short summary of each published book, then total earned, paid and pending.
If there’s money pending, it explains the quarterly payout, the 45-day window and the Rs. 1,000 minimum.
If everything is paid, it just says thanks.
Books in production get a one-line status update.
It can only use the data I send and can’t invent numbers. The output starts with “SUBJECT:” on the first line, then the body.
I don’t let the AI work out dates. I send the next payout date as plain text so it can’t mess it up. (check: right now it’s a fixed date. Say so if you didn’t make it dynamic.)
## 5. Error handling
Security: the n8n webhook needs a secret header, and the call back into Bubble uses a Bearer API token.
Double clicks: the button only makes a new log if that author doesn’t already have a Pending one. This stops duplicate emails. (check)
If the AI fails: n8n sends a Failed status back to Bubble with an error message, so the failure gets saved and isn’t silent. (check: only keep this if you actually built the error branch.)
If n8n never answers: the log just stays Pending, and the admin can see it in the Notification Log. (check)

## 6. Challenges and what I learned

Bubble was new to me, so I was learning it while building.

Data import: CSV upload needs a paid plan. I typed the data in by hand, then linked books to authors with a backend workflow.
Login redirect: conditions like “Current User’s role is author” didn’t work right after login. I moved the role check into each page’s page-load workflow.
Privacy rules: my first Book rule returned 0 books, because Bubble doesn’t allow searching with a rule based on “This Book’s author’s user”. I switched it to compare author IDs.
Backend workflows: totals came out empty until I turned on “Ignore privacy rules” for the workflow. I also fixed the overdue check, which was marking everyone with pending money as overdue.
n8n connection: my first call got a 403 because the secret header was saved as a blank placeholder. I re-created it with a real value.
Time: the longest part was figuring out where each setting lives in Bubble. I mostly learned by breaking things, reading the logs, and fixing them one by one.
Known issue

This was my first time using Bubble, so I was learning as I went.

One thing still doesn’t work. When I log in as admin, the app opens the author dashboard instead of the admin dashboard. I tried fixing the redirect with role conditions but couldn’t get it working before the deadline. The admin dashboard and the author detail page both work fine on their own. It’s only the redirect right after login that’s broken.