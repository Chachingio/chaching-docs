# Credit Card Surcharges

On an account that charges through iPOSpays (Dejavoo), every credit card charge ChaChing makes can carry a credit card surcharge: the first charge and every renewal of a subscription, one-time payments from the hosted checkout and from payment links, invoice payments from the payment page, the dashboard and the API, payment plan charges, and Google Pay payments made with a credit card. Debit card charges carry no surcharge. Whether a credit card charge is surcharged, and how much, is decided by the S.T.E.A.M configuration of your account in the iPOSpays portal. ChaChing records the surcharge separately and never changes an amount you set because of it: prices, invoice totals, `amount_due`, `amount_paid` and payment plan amounts stay as agreed.

Accounts that charge through NMI or Stripe never carry a surcharge. A payment recorded as external carries none, because nothing is charged to a card. A payment plan payment taken on your terminal with a credit card carries the surcharge the terminal added, and the invoices and transactions it paid report it.

The rule applies to every account that charges through iPOSpays; ChaChing has no setting that turns it off. An account with no fee configured in S.T.E.A.M is charged no surcharge.

---

## Configure your iPOSpays account

The surcharge is the fee configured in S.T.E.A.M for your terminal profile. Keep Tip and Tax disabled in S.T.E.A.M: the gateway adds the fee, the tip and the tax of your configuration together, and ChaChing already calculates the taxes of your invoices. Use the same fee on every terminal profile you use with ChaChing, and configure no fee for debit cards: ChaChing tells your customers that debit cards are not surcharged.

---

## Fields

Two fields report the surcharge, always present and in integer cents, on the Transaction and Invoice objects and on every `invoice.*` webhook. The Installment and Payment objects of a payment plan carry them too; see [Payment Plans](./payment-plans.md#credit-card-surcharges).

| **Field** | **Meaning** |
| --- | --- |
| `surcharge_amount` | The credit card surcharge added to the charge, or to the successful payments of an invoice. `0` when none was added, which is always the case for a debit card charge. |
| `amount_charged` | The total charged to the card. It equals `amount` on a transaction and `amount_paid` on an invoice when nothing was added. |

For example, a $25.00 credit card charge that carried a 75 cent surcharge reads `amount: 2500`, `surcharge_amount: 75`, `amount_charged: 2575` on the Transaction object, and the invoice it paid reads `amount_paid: 2500`, `surcharge_amount: 75`, `amount_charged: 2575`.

- `amount_charged` can exceed the amount plus `surcharge_amount` when the gateway adds another S.T.E.A.M amount for your account. That difference is included in `amount_charged` and is not itemised.
- A payment that did not succeed never shows a surcharge.
- `invoice.payment_succeeded` for a payment made through ChaChing describes the invoice after that payment, so its `amount_paid`, `surcharge_amount` and `amount_charged` include it.
- ChaChing does not refund a surcharge.

---

## What your customer sees before paying

Before the payer pays, ChaChing tells them that a credit card surcharge can be added and that debit cards are not surcharged. `<your account name>` is the name of your account; `The merchant` replaces it when your account has none.

| **Screen** | **Notice** |
| --- | --- |
| Payment page of an open invoice | `<your account name> can add a credit card surcharge to this amount when you pay with a credit card. Debit cards are not surcharged. The page shows the exact total charged after you pay.` |
| Hosted checkout of a one-time price | `<your account name> can add a credit card surcharge to this amount when you pay with a credit card. Debit cards are not surcharged. The page shows the exact total charged after you pay.` |
| Hosted checkout of a recurring price | `<your account name> can add a credit card surcharge to this amount when you pay with a credit card. Debit cards are not surcharged. The same surcharge can be added to each renewal charged to a credit card. The page shows the exact total charged after you pay.` |
| Payment link page | `<your account name> can add a credit card surcharge to this amount when you pay with a credit card. Debit cards are not surcharged.` |
| Dashboard **Charge customer** sheet | `A credit card surcharge set on your Dejavoo account can be added to this amount when the card is a credit card. Debit cards are not surcharged.` |
| Dashboard **Record payment** dialog of a payment plan, with **Collect on a Dejavoo terminal** selected | `The terminal can add the credit card surcharge set for your account to this amount when the customer pays with a credit card. Debit cards are not surcharged. The surcharge is not part of the plan.` The confirmation step then reads `Collect <amount> on <terminal name>? The customer pays it on the terminal; the terminal can add the credit card surcharge set for your account when the customer pays with a credit card.`, with the amount you entered and the terminal you selected. |
| Dashboard **Collect deposit on terminal** dialog of a payment plan | `The terminal can add the credit card surcharge set for your account to this amount when the customer pays with a credit card. Debit cards are not surcharged. The surcharge is not part of the plan.` |

On the hosted checkout of a price the notice is shown above the card fields and the Google Pay button. On the payment page of an open invoice it is shown under the amount due and above the invoice's line items and totals, beside the card fields on a wide screen and above them on a phone. On the payment link page it is shown under the amount. On these three pages the notice uses the color of the page's body text. In the Google Pay sheet the total is shown as an estimate, because the surcharge is known only after the gateway answers.

These notices do not state your rate: ChaChing cannot read the fee configured in your iPOSpays account. What the card networks ask you to disclose yourself is described in [Your disclosure obligations](#your-disclosure-obligations).

---

## What your customer sees after paying

- The payment page of a paid invoice shows **Credit card surcharge** and **Total charged** under **Amount paid** when the payments carried a surcharge.
- The payment page of an open invoice that was partly paid shows **Credit card surcharge** and **Total charged** under **Total** when those payments carried a surcharge.
- The invoice email and the invoice PDF show **Credit card surcharge** and **Total charged** under **Amount paid** when the invoice's payments carried a surcharge. On an invoice that is still open, **Amount paid** is then the amount already paid, and **Amount remaining** in the email and **Amount due** in the PDF are what is left to pay.
- The hosted checkout's confirmation shows **Total charged** when the total charged differs from the amount, and **Credit card surcharge** when a surcharge above zero was added. A payment that carried nothing extra shows neither line.
- When an automatic charge fails, the failed-payment email says: `If the card is a credit card, <your account name> can add a credit card surcharge to this amount. Debit cards are not surcharged.`

---

## In the dashboard

The transactions list, the transaction details, the column picker of the transactions export and the invoice details show the surcharge and the total charged for a charge that carried one, under the label **Credit card surcharge**; in the exported file the columns are `surcharge` and `amountCharged`, named by their column ids. The **Pricing** table of the invoice details and the downloaded invoice show **Credit card surcharge** and **Total charged** under **Total** when the invoice's payments carried a surcharge. The dashboard's Total revenue and a customer's total spend include the surcharge of the card charges ChaChing sends to iPOSpays; the surcharge of a payment taken on your terminal is not included in them yet.

---

## Your disclosure obligations

This section applies to accounts that charge through iPOSpays (Dejavoo). It quotes the card networks' published rules and is not legal advice; confirm the rules and laws that apply to your business with your acquirer.

**State your rate, and that it is not greater than your cost of acceptance.** ChaChing's notices do not state your rate, so you state it. Visa's rules (ID# 0027545) say: "A Merchant must, at both the point of entry into the Merchant Outlet and the Point-of-Transaction, clearly and prominently disclose any Credit Card Surcharge that will be assessed." The disclosure at the Point-of-Transaction must include "The exact amount or percentage of the Credit Card Surcharge", "A statement that the Credit Card Surcharge is being assessed by the Merchant and is only applicable to credit Transactions" and, in the US, "A statement that the Credit Card Surcharge amount is not greater than the applicable Merchant Discount Rate for Visa Credit Card Transactions at the Merchant". For online sales, Visa names the point of entry as "The first page that references credit card brands accepted" and the Point-of-Transaction as the "Checkout page". State your rate, as a percentage or an amount, and that it is not greater than your cost of acceptance, at checkout and on your own sites and apps, starting on the first page that names the card brands you accept.

**Notify Mastercard and your acquirer 30 days ahead.** Mastercard's surcharge rules say the requirements "include advance notice to both Mastercard and the merchant's acquirer of the merchant's intention to impose a surcharge no less than thirty days before the merchant implements a surcharge." Visa asks the same of your acquirer: "Notify your acquirer at least 30 days prior to commencing surcharging".

**Put up signs where customers pay in person.** For sales on your terminals, Visa's rules ask for a sign at the "Main entrance(s) of the Merchant Outlet, in a minimum 32-point Arial font, but in any case no smaller or less prominent than surrounding text" and at "Every customer checkout or payment location, in a minimum 16-point Arial font, but in any case no smaller or less prominent than surrounding text". Visa publishes sample signs on its surcharging page.

**Check your state's law.** Some US states restrict surcharging. Mastercard: "Nothing in the Mastercard rules affects any obligation of a merchant to comply with applicable state or federal laws, including ... state laws that may prohibit or restrict surcharging". Visa: "Compliance with Visa's requirements does not imply compliance with any relevant State laws."

Sources:

- [Visa Core Rules and Visa Product and Service Rules](https://usa.visa.com/content/dam/VCOM/download/about-visa/visa-rules-public.pdf) (ID# 0027545 and Table 5-4)
- [Visa U.S. Merchant Surcharge Q&A](https://usa.visa.com/content/dam/VCOM/global/support-legal/documents/merchant-surcharging-qa-for-web.pdf)
- [Visa surcharging rules and sample signage](https://usa.visa.com/support/small-business/regulations-fees.html)
- [Mastercard merchant surcharge rules](https://www.mastercard.com/us/en/business/support/merchant-surcharge-rules.html)
