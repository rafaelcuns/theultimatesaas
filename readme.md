# The Ultimate SaaS — Enterprise-Grade Micro-Actions as a Service

**The Ultimate SaaS** is a satirical web platform designed to parody the modern startup ecosystem and subscription-based software culture. The project's goal is to apply the full architecture, security, and user experience of a cutting-edge enterprise product to solve completely trivial and useless problems behind aggressive paywalls. Beyond the comedic premise, this repository serves as a hands-on lab to master the end-to-end development lifecycle of a real SaaS—from authentication and role-based access control to international payment processing via webhooks.

## Features

* **Quantum Calculator (Addition Only):** Performs basic addition with an artificial loading delay and blurs the result for free-tier users.
* **Friday Checker:** Determines whether today is Friday. Free users only receive *"Maybe"*; paid subscribers unlock the definitive answer.
* **Dark Mode as a Service (DMaaS):** The classic light/dark theme toggle switch locked behind a monthly subscription.
* **Insecure Password Generator:** Exclusively generates `123456` on the free plan, requiring an upgrade to include random characters.
* **Enterprise Pricing Page:** A tiered pricing table (*Starter*, *Pro*, and *Enterprise*) packed with absurd perks and an annual billing toggle offering 0% off.

## Tech Stack

* **Frontend & Backend:** Next.js (App Router), React, TypeScript, and Tailwind CSS.
* **Authentication & Database:** Supabase (Auth and PostgreSQL).
* **Payments & Webhooks:** Lemon Squeezy (Merchant of Record for global checkout).
* **Infrastructure & Deployment:** Vercel.