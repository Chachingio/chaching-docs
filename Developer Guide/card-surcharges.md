# Card Surcharges

On an account that charges through iPOSpays (Dejavoo), every card charge ChaChing makes can carry a card surcharge: the first charge and every renewal of a subscription, one-time payments from the hosted checkout and from payment links, invoice payments from the payment page, the dashboard and the API, payment plan charges, and Google Pay payments. Whether a charge is surcharged, and how much, is decided by the S.T.E.A.M configuration of your account in the iPOSpays portal. ChaChing records the surcharge separately and never changes an amount you set because of it: prices, invoice totals, `amount_due`, `amount_paid` and payment plan amounts stay as agreed.

Accounts that charge through NMI or Stripe never carry a surcharge. A payment recorded as external carries none, because nothing is charged to a card.

The rule applies to every account that charges through iPOSpays; ChaChing has no setting that turns it off. An account with no fee configured in S.T.E.A.M is charged no surcharge.

---

## Configure your iPOSpays account

The surcharge is the fee configured in S.T.E.A.M for your terminal profile. Keep Tip and Tax disabled in S.T.E.A.M: the gateway adds the fee, the tip and the tax of your configuration together, and ChaChing already calculates the taxes of your invoices. Use the same fee on every terminal profile you use with ChaChing.

---

## Fields

Two fields report the surcharge, always present and in integer cents, on the Transaction and Invoice objects and on every `invoice.*` webhook. The Installment and Payment objects of a payment plan carry them too; see [Payment Plans](./payment-plans.md#card-surcharges).

| **Field** | **Meaning** |
| --- | --- |
| `surcharge_amount` | The card surcharge added to the charge, or to the successful payments of an invoice. `0` when none was added. |
| `amount_charged` | The total charged to the card. It equals `amount` on a transaction and `amount_paid` on an invoice when nothing was added. |

For example, a $25.00 charge that carried a 75 cent surcharge reads `amount: 2500`, `surcharge_amount: 75`, `amount_charged: 2575` on the Transaction object, and the invoice it paid reads `amount_paid: 2500`, `surcharge_amount: 75`, `amount_charged: 2575`.

- `amount_charged` can exceed the amount plus `surcharge_amount` when the gateway adds another S.T.E.A.M amount for your account. That difference is included in `amount_charged` and is not itemised.
- A payment that did not succeed never shows a surcharge.
- `invoice.payment_succeeded` for a payment made through ChaChing describes the invoice after that payment, so its `amount_paid`, `surcharge_amount` and `amount_charged` include it.
- ChaChing does not refund a surcharge.

---

## What your customer sees before paying

ChaChing tells the payer that a card surcharge can be added before they pay. `<your account name>` is the name of your account; `The merchant` replaces it when your account has none.

| **Screen** | **Notice** |
| --- | --- |
| Payment page of an open invoice | `<your account name> can add a card surcharge to this amount when you pay by card. The page shows the exact total charged after you pay.` |
| Hosted checkout of a one-time price | `<your account name> can add a card surcharge to this amount when you pay by card. The page shows the exact total charged after you pay.` |
| Hosted checkout of a recurring price | `<your account name> can add a card surcharge to this amount when you pay by card. The same card surcharge can be added to each renewal charge. The page shows the exact total charged after you pay.` |
| Payment link page | `<your account name> can add a card surcharge to this amount when you pay by card.` |
| Dashboard **Charge customer** sheet | `A card surcharge configured on your Dejavoo account can be added to this amount.` |
| Dashboard **Record payment** dialog of a payment plan, with **Collect on a Dejavoo terminal** selected | `The terminal can add the card surcharge set for your account to this amount. The surcharge is not part of the plan.` The confirmation step then reads `Collect <amount> on <terminal name>? The customer pays it on the terminal; the terminal can add the card surcharge set for your account.`, with the amount you entered and the terminal you selected. |
| Dashboard **Collect deposit on terminal** dialog of a payment plan | `The terminal can add the card surcharge set for your account to this amount. The surcharge is not part of the plan.` |

On the hosted checkout of a price the notice is shown above the card fields and the Google Pay button. On the payment page of an open invoice it is shown in the invoice summary, under the amount due, beside the card fields on a wide screen and above them on a phone. On the payment link page it is shown under the amount. In the Google Pay sheet the total is shown as an estimate, because the surcharge is known only after the gateway answers.

These notices say that a surcharge can be added, not its rate: ChaChing cannot read the fee configured in your iPOSpays account. Card network rules ask you to disclose your surcharge before the sale; disclose your rate in your terms and at your own points of sale.

---

## What your customer sees after paying

- The payment page of a paid invoice shows **Surcharge** and **Total charged** under **Amount paid** when the payment carried a surcharge.
- The hosted checkout's confirmation shows **Total charged** when the total charged differs from the amount, and **Surcharge** when a surcharge above zero was added. A payment that carried nothing extra shows neither line.
- When an automatic charge fails, the failed-payment email says: `When your card is charged, <your account name> can add a card surcharge to this amount.`

---

## In the dashboard

The transactions list, the transaction details, the transactions export and the invoice details show the surcharge and the total charged for a charge that carried one. The dashboard's Total revenue and a customer's total spend include the surcharge.
