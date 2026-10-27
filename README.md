# Author Royalty Dashboard (Bubble + n8n)

I built an author royalty dashboard in Bubble. Admins can click a button, and n8n uses Gemini AI to write a royalty summary for the author. The result comes back to Bubble and is saved in a Notification Log.

**Links**
- App: https://tejasworld024.bubbleapps.io/version-test/login (development version, free plan)
- Admin: admin@bookleaf.com / Admin@123
- Author: priya.sharma@email.com / Author@123 (all authors use Author@123)
- n8n workflow file: `n8n-workflow.json`

## 1. Database

| Data type | What it stores |
|---|---|
| Author | author_id, name, email, phone, city, joined date, linked user, and totals (total_books, total_earned, total_paid, total_pending, payout_status) |
| Book | book_id, title, ISBN, genre, status, dates, MRP, copies sold, royalty earned / paid / pending, last payout date, print partner, platforms, and the Author it belongs to |
| Notification Log | which author, status (Pending / Success / Failed), email subject, email body, error message, completed time |
| User | Bubble's login account, plus a role (author or admin) and a link to an Author |

**Why I built it this way**
- One Author has many Books, so each Book points to its Author. This is like a foreign key in a normal database.
- The Notification Log is separate, so every email attempt is saved with its own status.
- I added totals on Author so the admin list can show and filter without recalculating each time.
- The role field on User keeps admin and author access simple.
- I kept `author_id_text` on Book as a plain-text copy of the author ID. It helps with data checks and privacy rules.
- Books still in production have empty MRP and dates. I left them empty on purpose.

## 2. Author and admin screens

- **Author:** logs in and sees only their own books. Each book shows its details, royalty figures, and a badge:
  - Green: fully paid
  - Red: pending and last payout over 90 days ago, or never paid on a book older than 90 days
  - Yellow: pending but still in the normal cycle
  - Grey: in production
  
  Totals for earned, paid, and pending are shown at the top.
- **Admin:** sees all authors with total books, earned, and pending. They can search by name and filter by city and payout status. Clicking an author opens a detail page with all their books and figures.
- **Privacy rules:** authors can only see their own Books and Author record. Admin can see everything. The Notification Log is admin only. I tested this by logging in as different authors (Priya sees 2 books, Sneha sees 3, admin sees 18).

## 3. How Bubble and n8n work together

1. The admin opens an author's detail page and clicks **Generate & Send Royalty Summary**.
2. Bubble creates a Notification Log row with status **Pending**.
3. Bubble calls the n8n webhook using the API Connector. It sends the log id, author id, author name, and a text summary of the author's books.
4. n8n checks a secret header (`x-api-key`). Without it, the webhook rejects the call, so it is not open to everyone.
5. n8n sends the data to Gemini and gets back the email subject and body.
6. n8n calls a Bubble backend workflow (`update_notification_log`) with an API token. It saves the subject and body and sets the log status.
7. The log row changes from Pending to Success.

**Why I send the data in the webhook (and don't fetch it from Bubble's Data API)**
- It is simpler. n8n does not need extra Bubble API access to read data.
- It is faster, with fewer calls that could fail.
- Bubble already knows which author and books to send.
- The trade-off: if the data changes after the click, n8n will not see it. That is fine for this use case.

## 4. Prompt strategy

- The prompt tells Gemini to act as a BookLeaf support person with a warm, professional tone.
- It asks for a greeting with the author's name, a short summary of each published book, then total earned, paid, and pending.
- If money is pending, it explains the quarterly payout, the 45-day window, and the Rs. 1,000 minimum.
- If everything is paid, it thanks the author.
- Books in production get a one-line status update.
- It must use only the data I send and never invent numbers. It outputs "SUBJECT:" on the first line, then the body.
- Dates are not worked out by the AI. I send the next payout date as text, so it cannot get the date wrong. **(check: currently a fixed date; say so if you did not make it dynamic)**

## 5. Error handling

- **Webhook security:** secret header on n8n, and a Bearer API token for the call back into Bubble.
- **Double clicks:** the button creates a new log only if there is no Pending log for that author. This stops duplicate emails. **(check)**
- **If the AI fails:** n8n sends a Failed status back to Bubble with an error message, so the failure is saved and not silent. **(check: only keep this if you built the error branch)**
- **If n8n does not answer:** the log stays Pending, and the admin can see it in the Notification Log. **(check)**

## 6. What is not finished

- Bulk trigger for all authors with pending royalty.
- Sending a real email by SMTP (the email text is saved in the log instead).
- The next payout date is typed in and not calculated.
- The app runs on Bubble's free development version, since "Deploy" needs a paid plan.

## 7. Challenges and what I learned

Bubble was new to me, so I learned it while building.
- **Data import:** CSV upload needs a paid plan. I entered the data by hand, then linked books to authors with a backend workflow.
- **Login redirect:** conditions like "Current User's role is author" did not work right after login. I moved the role check to the page-load workflow of each page.
- **Privacy rules:** my first Book rule returned 0 books, because Bubble does not allow search with a rule based on "This Book's author's user". I changed it to compare author IDs.
- **Backend workflows:** totals came out empty until I turned on "Ignore privacy rules" for the workflow. I also fixed the overdue check, which was marking everyone with pending money as overdue.
- **n8n connection:** my first call returned 403 because the secret header was saved as a blank placeholder. I re-created it with a real value.
- **Time:** the longest part was learning where each setting lives in Bubble. I learned most by breaking things, reading the logs, and fixing them step by step.

## Known issue 

This was my first time using Bubble, so I was learning it while building the project.

One thing is not working properly: when I log in as admin, the app opens the author dashboard instead of the admin dashboard. I tried fixing the redirect after login with role conditions, but I could not get it working before the deadline. The admin dashboard page and the author detail page both work on their own. The only problem is the redirect right after login.