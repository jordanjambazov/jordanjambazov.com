---
layout: post
title:  "Fibank Connector - Making sense out of my banking transactions"
date:   2022-03-12 17:35:00 +0200
categories: projects
---

In this blog post, I would like to share a connector to the Fibank platform exporting historical transactions, thus allowing me to easily feed transaction data into other software (Airtable, Google Spreadsheets, Odoo) and analyze it.

Why do I need a separate connector? Because the Fibank reporting has a few limitations.

- It is impossible to filter a date ranger bigger than 30 days
- One cannot filter by transaction type, party, etc
- The platform doesn't provide capabilities to export all transactions to a machine-readable format
- By reverse-engineering their API, I realized it provides valuable information not visible in their UI
- The connector would allow me to further enrich the payload of the transactions and adapt them to my needs

## Reverse-engineering the authentication

The authentication flow is a weird stateful mixture of OAuth2 with a session-based authentication. It uses HTML, JSON, and HTTP headers for transfer purposes. Check the following sequence diagram.

![Auth Sequence Diagram](/assets/images/fibank-connector-sequence-diagram.svg)

After parsing the access token from the redirect, you can use it for further authentication to the FiBank API. Check the [auth implementation of the connector](https://github.com/jordanjambazov/fibank-connector/blob/880b972ea13e8e9d5203c82036f7d5b6172c9961/connector/auth.py).

## Connecting to the API

For the use case of our connector, we are going to use the following authenticated endpoints:

- `GET sywspicklist/sywspicklist/getFilteredAccounts` - returns all accounts with the corresponding IBAN
- `GET sywsquery/sywsquery/GetCustBal?FromDate=<date>&ToDate=<date>&Iban=<iban>&StmtType=T&isAccountFromPSD2=false` - returns all transactions in the given timeframe

To authenticate we use the previously obtained `access_token`. [Check the connector client implementation here](https://github.com/jordanjambazov/fibank-connector/blob/04dbf55be31fb0a46d4d32299b4ed28e4117cb73/connector/client.py).

## Reconciling the transactions to a local DB

Now that we have the basic primitives to communicate with the Fibank API, we can prepare the [engine to reconcile the transactions](https://github.com/jordanjambazov/fibank-connector/blob/master/connector/engine/engine.py) within the connector DB.

```python
class Engine:
    def __init__(self):
        auth = Auth(settings.FIBANK_USERNAME, settings.FIBANK_PASSWORD)
        self._client = Client(auth)

    def reconcile_all(self):
        self.reconcile_accounts()
        self.reconcile_transactions()

    def reconcile_accounts(self):
        accounts = self._client.get_filtered_accounts()
        for account in accounts:
            # store account in DB

    def reconcile_transactions(self):
        for db_account in Account.objects.all():
            customer_balance = self._client.get_customer_balance(db_account.iban)
            statement = customer_balance['acc'][0]['stmt']
            for transaction in statement:
                # store transaction in DB
```

Now, as the transaction data is in our control, we could analyze it. Let's use SQL for this purpose.

```sql
$ mysql -u fibank -p -h localhost fibank
mysql> select count(*) from transaction_transaction;
+----------+
| count(*) |
+----------+
|     2837 |
+----------+
1 row in set (0.01 sec)
```

## Exposing the transaction data via HTTP

I also decided to expose the transaction data via HTTP to integrate easily with Airtable.

```python
def all_transactions(request):
    # Make sure to handle authentication
    transactions = Transaction.objects.all().order_by('time')
    return JsonResponse({
        'result': [serialize_tx(tx) for tx in transactions]
    })
```

## Deploying the connector to Heroku

Now we are ready to find a home for the connector. I have chosen Heroku due to the simplicity of spinning out a service there. The `Procfile` is as simple at this:

```
web: python manage.py runserver 0.0.0.0:$PORT
clock: python clock.py
```

For simple authentication purposes, the connector uses a randomly generated UUID. Generate one and store it in the `AUTH_SECRET` environment variable. The e-banking username and password go to `FIBANK_USERNAME` & `FIBANK_PASSWORD`.

We have a web process running the webserver. The [clock process](https://github.com/jordanjambazov/fibank-connector/blob/799a91cc2431b0d03325c6b9a6b41616ea9ba470/clock.py) periodically connects to the e-banking platform and reconciles the transactions. The actual process of [deploying the service to Heroku is outside the scope of this blog post, as you could find this information elsewhere](https://devcenter.heroku.com/articles/deploying-python).

## Integrating the connector with Airtable (or whatever)

In this case, I'm integrating with Airtable. Of course, you can integrate with any tool that you'd like. You can achieve a similar result with other software.

- Create a workspace and a table within it
- Use Data Fetcher to feed the transaction data from `https://<your_app>.herokuapp.com/all-transactions/`, make sure to pass the authorization token
  ![Fibank Connector - Airtable Data Fetcher](/assets/images/fibank-connector-airtable-data-fetcher.png)
- Run the request and populate the table
- Now that you have the data populated in Airtable, you can generate charts similar to this one (debit vs credit):
  ![Fibank Connector - Debit vs Credit](/assets/images/fibank-connector-airtable-chart.png)
- You can filter by transaction type, group by vendor, have better visibility on how much you spend in fees. This approach could help you to analyze your banking transactions.

## Conclusion

Even though this post presents the Fibank connector that I prepared, the approach would be similar for any other e-banking system. Let's generalize it as follows: 1) connect to the API, 2) reconcile the transactions, 3) re-expose the data in order to analyze it in other software, or simply use SQL to query it.
