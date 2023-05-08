# abnamro-hass
This is a quick proof-of-concept to import back-transaction into Home Assistant and make it possible to categorize these transactions.
For my private banking-account there is no automatic bridge available. To get around this, i've chosen to use Playwright to semi-automatically download the transactions.
Each retrieved transaction will be passed to my Home Assistant via a Home Assistant webhook.

## Helpers
The following helpers need to be created in Home Assistant.

| Type | Name | Icon | Other |
|------|------|------|-----|
| Numeric | Transaction amount    | mdi:currency-eur       | Min=-100000, Max=+100000, Inputfield, Step=0.01, Unit=euro |
| Numeric | Transaction reference | mdi:id-card            | |
| Text    | Transaction details   | mdi:card-text-outline  | maxlength=100 |
| Select  | Transaction category  | mdi:shape-plus-outline | Options=["None","Food","Pets","Mortage","Internet"] |
| Numeric | Saldo                 | mdi:calculator"        | |
| Switch  | Transaction available |                        | |
| Button  | Transaction save      | mdi:content-save       | |
| Button  | Transaction ignore    | mdi:content-save-off   | |

## Dashboard
The following Dashboard configuration provides a card which is conditionally available.
```
views:
  - title: Finance
    cards:
      - type: conditional
        conditions:
          - entity: input_boolean.transaction_available
            state: 'on'
        card:
          type: horizontal-stack
          cards:
            - type: vertical-stack
              cards:
                - type: vertical-stack
                  cards:
                    - type: markdown
                      entity: input_text.transaction_details
                      content: '{{ states("input_text.transaction_details") }}'
                    - type: entity
                      entity: input_number.transaction_amount
                    - type: entity
                      entity: input_text.transaction_reference
                    - type: entity
                      entity: input_text.transaction_date
                    - type: entity
                      entity: input_text.transaction_id
            - type: vertical-stack
              cards:
                - type: entities
                  entities:
                    - entity: input_select.transaction_category
                - type: horizontal-stack
                  cards:
                    - show_name: true
                      show_icon: true
                      type: button
                      tap_action:
                        action: toggle
                      entity: input_button.transaction_save
                    - show_name: true
                      show_icon: true
                      type: button
                      tap_action:
                        action: toggle
                      entity: input_button.transaction_ignore
      - type: entity
        entity: input_number.saldo
```


## Automations

## Transaction queue clear
```
alias: Transaction clear at reboot
description: ""
trigger:
  - platform: homeassistant
    event: start
condition: []
action:
  - service: input_boolean.turn_off
    data: {}
    target:
      entity_id: input_boolean.transaction_available
mode: single

```

### Transaction process
```
alias: Transaction process
description: ""
trigger:
  - platform: webhook
    allowed_methods:
      - POST
    local_only: true
    webhook_id: transaction
condition: []
action:
  - service: input_text.set_value
    data:
      value: "{{ trigger.json.transaction_details }}"
    target:
      entity_id: input_text.transaction_details
  - service: input_text.set_value
    data:
      value: "{{ trigger.json.transaction_reference }}"
    target:
      entity_id: input_text.transaction_reference
  - service: input_text.set_value
    data:
      value: "{{ trigger.json.id }}"
    target:
      entity_id: input_text.transaction_id
  - service: input_text.set_value
    data:
      value: "{{ trigger.json.date }}"
    target:
      entity_id: input_text.transaction_date
  - service: input_number.set_value
    data:
      value: "{{ trigger.json.amount | float }}"
    target:
      entity_id: input_number.transaction_amount
  - service: input_select.select_option
    data:
      option: None
    target:
      entity_id: input_select.transaction_category
  - service: input_boolean.turn_on
    data: {}
    target:
      entity_id: input_boolean.transaction_available
  - wait_for_trigger:
      - platform: state
        entity_id:
          - input_button.transaction_save
          - input_button.transaction_ignore
          - input_select.transaction_category
  - service: input_boolean.turn_off
    data: {}
    target:
      entity_id: input_boolean.transaction_available
  - if:
      - condition: template
        alias: Transaction ignored
        value_template: "{{ wait.trigger.entity_id == 'input_button.transaction_ignore' }}"
    then:
      - stop: Gekozen voor ignore
  - service: input_number.set_value
    data:
      value: >-
        {{ (states('input_number.saldo') | float) + (trigger.json.amount | float) }}
    target:
      entity_id: input_number.saldo
mode: queued
max: 1000
```

## main.py
```
import nest_asyncio; nest_asyncio.apply()  # This is needed to use sync API in repl
from playwright.sync_api import sync_playwright
import base64
from urllib.parse import urlparse
import requests
import urllib3
import mt940
import locale
from datetime import date 

locale.setlocale(locale.LC_ALL, 'en_US.UTF-8')

def transmitToWebhook(webhook_data:object) -> None:
    urllib3.disable_warnings()
    response = requests.post("https://<<your-hass-servername>>:8123/api/webhook/transaction", json=webhook_data, verify=False)
    if response.status_code != 200:
        print("Unexpected Status Code: ", response.status_code)
        exit(-1)


def processMT940Data(data: str, last_run_date: date) -> None:
    transactions = mt940.parse(data)

    for transaction in transactions:        
        transaction_date = transaction.data["date"]
        if transaction_date >= last_run_date:
            if transaction_date != date.today():
                webhook_data = {
                    "status": transaction.data["status"],
                    "funds_code": transaction.data["funds_code"],
                    "amount": transaction.data["amount"].amount,
                    "id": transaction.data["id"],
                    "customer_reference": transaction.data["customer_reference"],
                    "bank_reference": transaction.data["bank_reference"],
                    "extra_details": transaction.data["extra_details"],
                    "currency": transaction.data["currency"],
                    "date": transaction.data["date"].isoformat(),
                    "entry_date": transaction.data["entry_date"].isoformat(),
                    "guessed_entry_date": transaction.data["guessed_entry_date"].isoformat(),
                    "transaction_reference": transaction.data["transaction_reference"],
                    "transaction_details": transaction.data["transaction_details"],
                }
                print("Transmitting transaction for date: ", transaction_date, webhook_data)
                transmitToWebhook(webhook_data)
            else:
                print("Skipping transactions of today: ", transaction_date)
        else:
          print("Skipping already processed transaction: ", transaction_date)
    

def downloadMT940File() -> str:
    with sync_playwright() as pw:
        browser = pw.chromium.launch(
            # we can choose either a Headful (With GUI) or Headless mode:
            headless=False,
        )
        context = browser.new_context(
            viewport={"width": 1920, "height": 1080}
        )

        page = context.new_page()
        page.goto("https://www.abnamro.nl/portalserver/my-abnamro/self-service/download-transactions/index.html")
        
        page.wait_for_selector('button[data-translate="button:download"]')
        format_selector = page.locator('select[data-ng-model="downloadMutations.pdfFormatOptions.default"]')
        format_selector.select_option(value="MT940") 

        download_button_selector = page.locator('button[data-translate="button:download"]')
        with page.expect_download() as download_transaction:
            download_button_selector.click()

        download = download_transaction.value
        download.save_as(download.suggested_filename)        
        path = download.path()
        
        parsed_url = urlparse(download.url)
        if parsed_url.scheme == "data":
            base64data = parsed_url.path.replace("application/octet-stream;base64,","")
            data = base64.b64decode(base64data)
            return data 

        else:
            print ("unexpected url format", parsed_url)
            exit(-1)

last_run_date = date.today()
try:
    last_run_file = open("last_run.txt", "r")
    last_run_date = date.fromisoformat(last_run_file.readline())
    last_run_file.close()
except ValueError:
    print("Invalid date file, using today")
except IOError:
    print("No date file present, using today")
    

# Use this to do the actual download process
data = downloadMT940File()
processMT940Data(data, last_run_date)

# Or use this to use an already download file, for testing purposes
# f = open("MT940230508123123.STA", "r")
# data = f.read()
# processMT940Data(data, last_run_date)


last_run_file = open("last_run.txt", "w")
last_run_file.writelines( date.today().isoformat())
last_run_file.close()
```
