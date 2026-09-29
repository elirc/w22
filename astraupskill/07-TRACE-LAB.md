# Trace lab

| Step | Valid 40 refund | Over 41 refund |
| --- | --- | --- |
| payment read | tx client, status SUCCEEDED | tx client, status SUCCEEDED |
| successful sum | 60 | 60 |
| remaining | 40 | 40 |
| assertion | passes | RouteError conflict |
| create row | called with tx | never called |
| outcome | refund workflow continues | no refund row |

Trace a missing payment: the tx read returns null and the service raises 404 before aggregation. Trace a FAILED payment: aggregation may run, but status assertion rejects before create. These branches separate CRUD absence, domain refusal, and amount refusal.

Add two more columns to your worksheet: the client identity and the first thrown value. Every repository operation in the revised path receives the same tx object. A missing-payment trace ends before sum; a policy refusal ends before create; an event failure ends after create and the first event attempt. These are different failure boundaries even though none returns a successful result. For concurrency, draw two transactions reading the same remaining amount and ask which database mechanism would prevent both from committing. The callback alone is not an answer.
