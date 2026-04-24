# User Guide — Frequently Asked Questions

### Why is my balance shown with a `~` prefix?

The total mixes currencies. The app converts each foreign account at today's rate, so the number is close but not exact to the cent.

### What does the "Stale rate" warning mean?

Your rates are older than eight hours. You can still record expenses — they are tagged with a "cached rate" label. Open the app online briefly to refresh.

### My expense shows a different amount than my bank debited — why?

The app uses the interbank rate plus your bank's average commission. Your bank may round or fee slightly differently. Tap **Refine amount** on the expense and enter the exact figure from your bank app.

### How accurate is the AI bank commission lookup?

It is a suggestion. Gemini returns the typical commission for that bank's card. Check and edit the percentage before saving.

### Can I use the app without internet?

Yes. Recording, editing, deleting, the dashboard, statistics and settings all work offline. Only the rate-history chart and the AI lookup need internet.

### How do I record a cash payment abroad?

Open the transaction form and flip the **Card / Cash** toggle to **Cash**. Enter the rate from the exchange office. No bank commission is applied in cash mode.

You can also record the bureau exchange separately in **Settings → Currency Exchange**. After that, the rate is available in the transaction form automatically — new cash expenses in that currency use it without you typing the rate again.

### Can I change my base currency after setup?

Not yet. Changing the base currency after setup requires recalculating all historical transactions at their original rates, which is planned for a future release. During onboarding, choose your base currency carefully.

### How do I fix a transaction I recorded incorrectly?

Open the transaction list, tap the wrong entry, tap **Edit**, change any field and save. To remove it, tap **Delete** instead.

### What happens if I delete an account?

All transactions on that account are also removed. The app asks to confirm before doing this.

### Why does the rate-history chart show a flat yellow line?

The rate did not move during the period, usually a weekend or holiday. It is not an error — the line is flat because the rate was flat.

### How often are exchange rates updated?

Every eight hours. The backend takes the median of three providers, so an outage in one provider does not break the app.

### What is the "Refine amount" button for?

It lets you enter the exact figure your bank charged. The app stores your amount as the truth, so reports show real spending to the cent.
