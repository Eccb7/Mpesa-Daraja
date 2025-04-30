
## Implementing Specific M-Pesa API Endpoints

Now, let's add functions to `mpesa_api/services.py` for each specific M-Pesa API endpoint required.

### 3.1. STK Push (Lipa Na M-Pesa Online)

This initiates a payment prompt on the customer's phone.

```python
# mpesa_api/services.py (add this function)

def initiate_stk_push(phone_number, amount, account_reference, transaction_desc):
    """Initiates an STK Push request."""
    timestamp = generate_timestamp()
    password = generate_stk_password(MPESA_SHORTCODE, MPESA_PASSKEY, timestamp)
    
    # Format phone number to Safaricom standard (254...)
    if phone_number.startswith("0"):
        phone_number = "254" + phone_number[1:]
    elif phone_number.startswith("+"):
        phone_number = phone_number[1:]

    # Construct the callback URL dynamically
    callback_url = f"{MPESA_CALLBACK_BASE_URL}/api/mpesa/stk_callback/"

    payload = {
        "BusinessShortCode": MPESA_SHORTCODE,
        "Password": password,
        "Timestamp": timestamp,
        "TransactionType": "CustomerPayBillOnline", # Or "CustomerBuyGoodsOnline"
        "Amount": str(amount), # Amount must be a string
        "PartyA": phone_number,
        "PartyB": MPESA_SHORTCODE,
        "PhoneNumber": phone_number,
        "CallBackURL": callback_url,
        "AccountReference": account_reference,
        "TransactionDesc": transaction_desc
    }

    endpoint = "/mpesa/stkpush/v1/processrequest"
    response = make_api_call(endpoint, method="POST", data=payload)
    return response

```

### 3.2. C2B (Customer to Business)

C2B involves customers sending money to your PayBill or Till Number. You need to register URLs to receive notifications (Confirmation and Validation).

**a) Register C2B URLs:**

```python
# mpesa_api/services.py (add this function)

def register_c2b_urls():
    """Registers Confirmation and Validation URLs for C2B."""
    payload = {
        "ShortCode": MPESA_SHORTCODE,
        "ResponseType": "Completed", # Or "Cancelled" if you want to cancel
        "ConfirmationURL": f"{MPESA_CALLBACK_BASE_URL}/api/mpesa/c2b_confirmation/",
        "ValidationURL": f"{MPESA_CALLBACK_BASE_URL}/api/mpesa/c2b_validation/"
    }
    endpoint = "/mpesa/c2b/v1/registerurl"
    response = make_api_call(endpoint, method="POST", data=payload)
    return response
```

**b) Simulate C2B Transaction (for Sandbox testing):**

```python
# mpesa_api/services.py (add this function)

def simulate_c2b_transaction(amount, phone_number, bill_ref_number="TestC2B"):
    """Simulates a C2B transaction for testing purposes."""
    # Format phone number
    if phone_number.startswith("0"):
        phone_number = "254" + phone_number[1:]
    elif phone_number.startswith("+"):
        phone_number = phone_number[1:]
        
    payload = {
        "ShortCode": MPESA_SHORTCODE,
        "CommandID": "CustomerPayBillOnline", # Or "CustomerBuyGoodsOnline"
        "Amount": str(amount),
        "Msisdn": phone_number,
        "BillRefNumber": bill_ref_number
    }
    endpoint = "/mpesa/c2b/v1/simulate"
    response = make_api_call(endpoint, method="POST", data=payload)
    return response
```

### 3.3. B2C (Business to Customer)

This allows you to send money from your M-Pesa shortcode to a customer's phone number (e.g., refunds, disbursements).

```python
# mpesa_api/services.py (add this function)

def initiate_b2c_payment(amount, phone_number, remarks, occasion="BusinessPayment"):
    """Initiates a B2C payment request."""
    # Format phone number
    if phone_number.startswith("0"):
        phone_number = "254" + phone_number[1:]
    elif phone_number.startswith("+"):
        phone_number = phone_number[1:]

    # Security Credential: Encrypt initiator password (Refer to Daraja docs for encryption method)
    # For sandbox, you might get a pre-generated one. For production, you need to encrypt.
    # This is a complex step involving public keys and certificates. 
    # Placeholder - replace with actual credential generation/retrieval
    security_credential = MPESA_INITIATOR_PASSWORD # WARNING: This is likely NOT the final encrypted credential
    # You MUST implement the proper encryption as per Safaricom documentation.

    payload = {
        "InitiatorName": MPESA_INITIATOR_NAME,
        "SecurityCredential": security_credential, 
        "CommandID": occasion, # e.g., "BusinessPayment", "SalaryPayment", "PromotionPayment"
        "Amount": str(amount),
        "PartyA": MPESA_SHORTCODE,
        "PartyB": phone_number,
        "Remarks": remarks,
        "QueueTimeOutURL": f"{MPESA_CALLBACK_BASE_URL}/api/mpesa/b2c_timeout/",
        "ResultURL": f"{MPESA_CALLBACK_BASE_URL}/api/mpesa/b2c_result/",
        "Occasion": occasion
    }
    endpoint = "/mpesa/b2c/v1/paymentrequest"
    response = make_api_call(endpoint, method="POST", data=payload)
    return response
```
**Important Note on B2C Security Credential:** The `SecurityCredential` is NOT just the plain initiator password. For the production environment (and often sandbox too), you need to encrypt your Initiator Password using the M-Pesa public key certificate. This involves using RSA encryption with OAEP padding. You'll need libraries like `cryptography` in Python. This process is critical and detailed in the official Safaricom Daraja documentation. The placeholder above is insufficient for a live system.

### 3.4. B2B (Business to Business)

Allows sending money from one business shortcode to another.

```python
# mpesa_api/services.py (add this function)

def initiate_b2b_payment(receiver_shortcode, amount, remarks, account_reference="B2BTransfer"):
    """Initiates a B2B payment request."""
    # Security Credential - Same encryption requirement as B2C
    security_credential = MPESA_INITIATOR_PASSWORD # WARNING: Placeholder - implement proper encryption

    payload = {
        "Initiator": MPESA_INITIATOR_NAME,
        "SecurityCredential": security_credential,
        "CommandID": "BusinessPayBill", # Or BusinessBuyGoods, DisburseFundsToBusiness, etc.
        "SenderIdentifierType": "4", # 4 for Shortcode
        "RecieverIdentifierType": "4", # 4 for Shortcode
        "Amount": str(amount),
        "PartyA": MPESA_SHORTCODE, # Sender
        "PartyB": receiver_shortcode, # Receiver
        "AccountReference": account_reference,
        "Remarks": remarks,
        "QueueTimeOutURL": f"{MPESA_CALLBACK_BASE_URL}/api/mpesa/b2b_timeout/",
        "ResultURL": f"{MPESA_CALLBACK_BASE_URL}/api/mpesa/b2b_result/"
    }
    endpoint = "/mpesa/b2b/v1/paymentrequest"
    response = make_api_call(endpoint, method="POST", data=payload)
    return response
```
**Note:** The `CommandID` and `IdentifierType` values depend on the specific B2B transaction type (PayBill, BuyGoods, etc.). Refer to the Daraja API documentation for the correct values.

### 3.5. Account Balance

Check the balance of your M-Pesa shortcode.

```python
# mpesa_api/services.py (add this function)

def check_account_balance(remarks="CheckBalance"):
    """Initiates an Account Balance query."""
    # Security Credential - Same encryption requirement as B2C/B2B
    security_credential = MPESA_INITIATOR_PASSWORD # WARNING: Placeholder - implement proper encryption

    payload = {
        "Initiator": MPESA_INITIATOR_NAME,
        "SecurityCredential": security_credential,
        "CommandID": "AccountBalance",
        "PartyA": MPESA_SHORTCODE,
        "IdentifierType": "4", # 4 for Shortcode
        "Remarks": remarks,
        "QueueTimeOutURL": f"{MPESA_CALLBACK_BASE_URL}/api/mpesa/balance_timeout/",
        "ResultURL": f"{MPESA_CALLBACK_BASE_URL}/api/mpesa/balance_result/"
    }
    endpoint = "/mpesa/accountbalance/v1/query"
    response = make_api_call(endpoint, method="POST", data=payload)
    return response
```




## 4. Django Views and URLs

Now, let's create Django views in `mpesa_api/views.py` to expose these functionalities via API endpoints and handle the callbacks from Safaricom.

```python
# mpesa_api/views.py
from django.shortcuts import render
from django.http import JsonResponse, HttpResponse
from django.views.decorators.csrf import csrf_exempt # Important for webhook endpoints
import json

# Import our M-Pesa service functions
from .services import (
    initiate_stk_push,
    register_c2b_urls,
    simulate_c2b_transaction,
    initiate_b2c_payment,
    initiate_b2b_payment,
    check_account_balance,
    get_mpesa_access_token # Can be useful for debugging or simple checks
)

# --- Views to Initiate M-Pesa Actions ---

# Example view to get access token (optional)
def get_token_view(request):
    if request.method == "GET":
        token = get_mpesa_access_token()
        if token:
            return JsonResponse({"access_token": token})
        else:
            return JsonResponse({"error": "Failed to get access token"}, status=500)
    return JsonResponse({"error": "Invalid request method"}, status=405)

# View to initiate STK Push
@csrf_exempt # Use csrf_exempt if called from external JS/frontend without CSRF token
def stk_push_view(request):
    if request.method == "POST":
        try:
            data = json.loads(request.body)
            phone_number = data.get("phone_number")
            amount = data.get("amount")
            account_reference = data.get("account_reference", "MyAppRef")
            transaction_desc = data.get("transaction_desc", "Payment")

            if not all([phone_number, amount]):
                return JsonResponse({"error": "Missing required fields: phone_number, amount"}, status=400)

            response = initiate_stk_push(phone_number, amount, account_reference, transaction_desc)
            return JsonResponse(response)
        except json.JSONDecodeError:
            return JsonResponse({"error": "Invalid JSON data"}, status=400)
        except Exception as e:
            return JsonResponse({"error": str(e)}, status=500)
    return JsonResponse({"error": "Invalid request method"}, status=405)

# View to register C2B URLs
@csrf_exempt
def register_urls_view(request):
    if request.method == "POST": # Or GET, depending on how you want to trigger it
        response = register_c2b_urls()
        return JsonResponse(response)
    return JsonResponse({"error": "Invalid request method"}, status=405)

# View to simulate C2B transaction
@csrf_exempt
def simulate_c2b_view(request):
    if request.method == "POST":
        try:
            data = json.loads(request.body)
            phone_number = data.get("phone_number")
            amount = data.get("amount")
            bill_ref_number = data.get("bill_ref_number", "TestSimulate")

            if not all([phone_number, amount]):
                return JsonResponse({"error": "Missing required fields: phone_number, amount"}, status=400)

            response = simulate_c2b_transaction(amount, phone_number, bill_ref_number)
            return JsonResponse(response)
        except json.JSONDecodeError:
            return JsonResponse({"error": "Invalid JSON data"}, status=400)
        except Exception as e:
            return JsonResponse({"error": str(e)}, status=500)
    return JsonResponse({"error": "Invalid request method"}, status=405)

# View to initiate B2C payment
@csrf_exempt
def b2c_payment_view(request):
    if request.method == "POST":
        try:
            data = json.loads(request.body)
            phone_number = data.get("phone_number")
            amount = data.get("amount")
            remarks = data.get("remarks", "Payment")
            occasion = data.get("occasion", "BusinessPayment")

            if not all([phone_number, amount]):
                return JsonResponse({"error": "Missing required fields: phone_number, amount"}, status=400)

            response = initiate_b2c_payment(amount, phone_number, remarks, occasion)
            # WARNING: Remember the Security Credential encryption requirement!
            return JsonResponse(response)
        except json.JSONDecodeError:
            return JsonResponse({"error": "Invalid JSON data"}, status=400)
        except Exception as e:
            return JsonResponse({"error": str(e)}, status=500)
    return JsonResponse({"error": "Invalid request method"}, status=405)

# View to initiate B2B payment
@csrf_exempt
def b2b_payment_view(request):
    if request.method == "POST":
        try:
            data = json.loads(request.body)
            receiver_shortcode = data.get("receiver_shortcode")
            amount = data.get("amount")
            remarks = data.get("remarks", "B2B Payment")
            account_reference = data.get("account_reference", "B2BTransfer")

            if not all([receiver_shortcode, amount]):
                return JsonResponse({"error": "Missing required fields: receiver_shortcode, amount"}, status=400)

            response = initiate_b2b_payment(receiver_shortcode, amount, remarks, account_reference)
            # WARNING: Remember the Security Credential encryption requirement!
            return JsonResponse(response)
        except json.JSONDecodeError:
            return JsonResponse({"error": "Invalid JSON data"}, status=400)
        except Exception as e:
            return JsonResponse({"error": str(e)}, status=500)
    return JsonResponse({"error": "Invalid request method"}, status=405)

# View to check account balance
@csrf_exempt
def check_balance_view(request):
    if request.method == "POST": # Or GET
        remarks = request.GET.get("remarks", "Balance Check") # Example using GET param
        response = check_account_balance(remarks)
        # WARNING: Remember the Security Credential encryption requirement!
        return JsonResponse(response)
    return JsonResponse({"error": "Invalid request method"}, status=405)


# --- Webhook (Callback) Handler Views ---
# IMPORTANT: These endpoints MUST be publicly accessible (e.g., via ngrok or deployment)
#            and MUST be exempted from CSRF protection.

@csrf_exempt
def stk_push_callback(request):
    if request.method == "POST":
        try:
            callback_data = json.loads(request.body)
            print("--- STK Push Callback Received ---")
            print(json.dumps(callback_data, indent=4))

            # Extract relevant data
            result_code = callback_data.get("Body", {}).get("stkCallback", {}).get("ResultCode")
            result_desc = callback_data.get("Body", {}).get("stkCallback", {}).get("ResultDesc")
            checkout_request_id = callback_data.get("Body", {}).get("stkCallback", {}).get("CheckoutRequestID")
            merchant_request_id = callback_data.get("Body", {}).get("stkCallback", {}).get("MerchantRequestID")

            if result_code == 0:
                # Success
                print("STK Push Successful!")
                metadata = callback_data.get("Body", {}).get("stkCallback", {}).get("CallbackMetadata", {}).get("Item", [])
                amount = next((item["Value"] for item in metadata if item["Name"] == "Amount"), None)
                receipt_number = next((item["Value"] for item in metadata if item["Name"] == "MpesaReceiptNumber"), None)
                phone_number = next((item["Value"] for item in metadata if item["Name"] == "PhoneNumber"), None)
                transaction_date = next((item["Value"] for item in metadata if item["Name"] == "TransactionDate"), None)
                
                print(f"  Amount: {amount}")
                print(f"  Receipt: {receipt_number}")
                print(f"  Phone: {phone_number}")
                print(f"  Date: {transaction_date}")
                
                # TODO: Process the successful payment
                # 1. Idempotency Check: Verify if this `checkout_request_id` or `merchant_request_id` has already been processed.
                #    transaction = TransactionModel.objects.filter(checkout_request_id=checkout_request_id).first()
                #    if transaction and transaction.status == 'Completed':
                #        print(f"Transaction {checkout_request_id} already processed.")
                #        return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"})
                #
                # 2. Find the original transaction record in your database.
                #    if not transaction:
                #        transaction = TransactionModel.objects.filter(merchant_request_id=merchant_request_id).first()
                #        if not transaction:
                #             print(f"ERROR: Transaction not found for CheckoutRequestID: {checkout_request_id}")
                #             # Decide how to handle - maybe log and accept, or create a pending record?
                #             return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"}) # Acknowledge Safaricom anyway
                #
                # 3. Update the transaction status to 'Completed'.
                #    transaction.status = 'Completed'
                #
                # 4. Store the M-Pesa receipt number and other relevant details.
                #    transaction.mpesa_receipt = receipt_number
                #    transaction.transaction_date = # Parse transaction_date if needed
                #    transaction.phone_number = phone_number # Store/verify payer phone
                #    transaction.paid_amount = amount
                #    transaction.save()
                #
                # 5. Fulfill the order, grant access, or trigger the next step in your business logic.
                #    print(f"Order fulfillment triggered for transaction {transaction.id}")
                #    # Example: fulfill_order(transaction.order_id)
                #
                # 6. Optionally, notify the user about the successful payment.

            else:
                # Failure or cancellation
                print(f"STK Push Failed/Cancelled: {result_desc} (Code: {result_code})")
                # TODO: Process the failed payment
                # 1. Idempotency Check: Verify if this `checkout_request_id` or `merchant_request_id` has already been marked as failed/cancelled.
                #    transaction = TransactionModel.objects.filter(checkout_request_id=checkout_request_id).first()
                #    if transaction and transaction.status in ["Failed", "Cancelled"]:
                #        print(f"Transaction {checkout_request_id} already marked as failed/cancelled.")
                #        return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"})
                #
                # 2. Find the original transaction record.
                #    if not transaction:
                #        transaction = TransactionModel.objects.filter(merchant_request_id=merchant_request_id).first()
                #        if not transaction:
                #             print(f"ERROR: Transaction not found for failed CheckoutRequestID: {checkout_request_id}")
                #             return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"}) # Acknowledge Safaricom
                #
                # 3. Update the transaction status to 'Failed' or 'Cancelled' based on `result_code`.
                #    transaction.status = 'Cancelled' if result_code == 1032 else 'Failed' # Example mapping
                #
                # 4. Log the error code and description for debugging.
                #    transaction.error_code = result_code
                #    transaction.error_description = result_desc
                #    transaction.save()
                #
                # 5. Optionally, notify the user about the payment failure.

            # Respond to Safaricom acknowledging receipt
            return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"})

        except json.JSONDecodeError:
            print("Error decoding STK callback JSON")
            return JsonResponse({"ResultCode": 1, "ResultDesc": "Failed"}, status=400)
        except Exception as e:
            print(f"Error processing STK callback: {e}")
            # Log the exception details
            return JsonResponse({"ResultCode": 1, "ResultDesc": "Failed"}, status=500)

    return JsonResponse({"error": "Invalid request method"}, status=405)

@csrf_exempt
def c2b_confirmation_callback(request):
    if request.method == "POST":
        try:
            callback_data = json.loads(request.body)
            print("--- C2B Confirmation Received ---")
            print(json.dumps(callback_data, indent=4))

            # Extract data (keys might vary slightly based on PayBill/BuyGoods)
            transaction_type = callback_data.get("TransactionType")
            trans_id = callback_data.get("TransID")
            trans_time = callback_data.get("TransTime")
            trans_amount = callback_data.get("TransAmount")
            business_short_code = callback_data.get("BusinessShortCode")
            bill_ref_number = callback_data.get("BillRefNumber") # Or InvoiceNumber
            org_account_balance = callback_data.get("OrgAccountBalance")
            third_party_trans_id = callback_data.get("ThirdPartyTransID")
            msisdn = callback_data.get("MSISDN") # Customer phone number
            first_name = callback_data.get("FirstName")
            # ... potentially MiddleName, LastName

            # TODO: Process the C2B payment confirmation
            # 1. Idempotency Check: Verify if this `trans_id` has already been processed.
            #    if C2BTransactionModel.objects.filter(transaction_id=trans_id).exists():
            #        print(f"C2B Transaction {trans_id} already processed.")
            #        return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"})
            #
            # 2. Record the transaction details in your database.
            #    C2BTransactionModel.objects.create(
            #        transaction_type=transaction_type,
            #        transaction_id=trans_id,
            #        transaction_time=trans_time, # Consider parsing to datetime object
            #        amount=trans_amount,
            #        business_short_code=business_short_code,
            #        bill_ref_number=bill_ref_number,
            #        invoice_number=callback_data.get("InvoiceNumber"), # If applicable
            #        org_account_balance=org_account_balance,
            #        third_party_trans_id=third_party_trans_id,
            #        msisdn=msisdn,
            #        first_name=first_name,
            #        middle_name=callback_data.get("MiddleName"),
            #        last_name=callback_data.get("LastName")
            #    )
            #    print(f"C2B Transaction {trans_id} recorded successfully.")
            #
            # 3. Update account balances, order status, or trigger relevant business logic.
            #    # Example: update_order_status(bill_ref_number, 'Paid')
            #
            # 4. Optionally, reconcile with validation data if you implemented validation.
            #    # (Validation might occur before confirmation)

            # Respond to Safaricom
            return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"})

        except json.JSONDecodeError:
            print("Error decoding C2B Confirmation JSON")
            return JsonResponse({"ResultCode": 1, "ResultDesc": "Failed"}, status=400)
        except Exception as e:
            print(f"Error processing C2B Confirmation: {e}")
            return JsonResponse({"ResultCode": 1, "ResultDesc": "Failed"}, status=500)

    return JsonResponse({"error": "Invalid request method"}, status=405)

@csrf_exempt
def c2b_validation_callback(request):
    if request.method == "POST":
        try:
            callback_data = json.loads(request.body)
            print("--- C2B Validation Received ---")
            print(json.dumps(callback_data, indent=4))

            # Extract data (similar to confirmation)
            bill_ref_number = callback_data.get("BillRefNumber")
            msisdn = callback_data.get("MSISDN")
            trans_amount = callback_data.get("TransAmount")

            # TODO: Implement validation logic
            # 1. Retrieve necessary information based on `bill_ref_number` (e.g., expected amount, customer details).
            #    try:
            #        order = OrderModel.objects.get(reference=bill_ref_number)
            #        expected_amount = order.amount
            #        is_active = order.is_active
            #    except OrderModel.DoesNotExist:
            #        print(f"Validation failed: Bill reference {bill_ref_number} not found.")
            #        return JsonResponse({"ResultCode": "C2B00011", "ResultDesc": "Rejected"}) # Example rejection code
            #
            # 2. Perform checks based on your business rules.
            #    - Amount Check: Is `trans_amount` equal to `expected_amount`?
            #    - Status Check: Is the order/reference still active/valid (`is_active`)?
            #    - Customer Check: Is `msisdn` allowed to pay for this reference (optional)?
            #
            #    validation_passed = True
            #    rejection_reason = "Accepted"
            #    rejection_code = 0
            #
            #    if not is_active:
            #        validation_passed = False
            #        rejection_reason = "Reference inactive"
            #        rejection_code = "C2B00012" # Example code
            #    elif float(trans_amount) != float(expected_amount):
            #        validation_passed = False
            #        rejection_reason = "Incorrect amount"
            #        rejection_code = "C2B00013" # Example code
            #    # Add more checks as needed
            #
            # 3. Respond based on validation result.
            #    if validation_passed:
            #        print(f"Validation successful for {bill_ref_number}")
            #        return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"})
            #    else:
            #        print(f"Validation failed for {bill_ref_number}: {rejection_reason}")
            #        return JsonResponse({"ResultCode": rejection_code, "ResultDesc": rejection_reason})

        except json.JSONDecodeError:
            print("Error decoding C2B Validation JSON")
            return JsonResponse({"ResultCode": 1, "ResultDesc": "Failed"}, status=400)
        except Exception as e:
            print(f"Error processing C2B Validation: {e}")
            return JsonResponse({"ResultCode": 1, "ResultDesc": "Failed"}, status=500)

    return JsonResponse({"error": "Invalid request method"}, status=405)

# --- Placeholder Callbacks for B2C, B2B, Balance --- 
# (Implement similarly to STK/C2B based on Daraja documentation for result/timeout data structures)

@csrf_exempt
def b2c_result_callback(request):
    if request.method == "POST":
        callback_data = json.loads(request.body)
        print("--- B2C Result Received ---")
        print(json.dumps(callback_data, indent=4))
        # TODO: Process B2C result (success or failure)
        # 1. Extract key identifiers: ConversationID, OriginatorConversationID
        #    result = callback_data.get("Result", {})
        #    result_code = result.get("ResultCode")
        #    result_desc = result.get("ResultDesc")
        #    originator_conversation_id = result.get("OriginatorConversationID")
        #    conversation_id = result.get("ConversationID")
        #    transaction_id = result.get("TransactionID")
        #
        # 2. Idempotency Check: Use ConversationID or OriginatorConversationID
        #    if B2CTransactionModel.objects.filter(conversation_id=conversation_id).exists():
        #        print(f"B2C Result {conversation_id} already processed.")
        #        return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"})
        #
        # 3. Find the original B2C request record using OriginatorConversationID.
        #    try:
        #        transaction = B2CTransactionModel.objects.get(originator_conversation_id=originator_conversation_id)
        #    except B2CTransactionModel.DoesNotExist:
        #        print(f"ERROR: Original B2C transaction not found for {originator_conversation_id}")
        #        # Log and accept, or handle as needed
        #        return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"})
        #
        # 4. Update transaction status based on result_code.
        #    if result_code == 0:
        #        transaction.status = "Completed"
        #        transaction.transaction_id = transaction_id # Store the M-Pesa transaction ID
        #        # Extract additional details from ResultParameters if needed
        #        result_params = result.get("ResultParameters", {}).get("ResultParameter", [])
        #        transaction_receipt = next((item["Value"] for item in result_params if item["Key"] == "TransactionReceipt"), None)
        #        recipient_balance = next((item["Value"] for item in result_params if item["Key"] == "B2CRecipientIsRegisteredMobileMoneyAccount"), None)
        #        # ... extract other relevant parameters
        #        transaction.mpesa_receipt = transaction_receipt
        #        print(f"B2C Payment {conversation_id} successful. Receipt: {transaction_receipt}")
        #    else:
        #        transaction.status = "Failed"
        #        transaction.error_code = result_code
        #        transaction.error_description = result_desc
        #        print(f"B2C Payment {conversation_id} failed. Code: {result_code}, Desc: {result_desc}")
        #
        # 5. Save changes and trigger any follow-up actions.
        #    transaction.conversation_id = conversation_id # Store the final ConversationID
        #    transaction.save()
        #    # Example: notify_user_of_payout_status(transaction)
        return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"})
    return JsonResponse({"error": "Invalid request method"}, status=405)

@csrf_exempt
def b2c_timeout_callback(request):
    if request.method == "POST":
        callback_data = json.loads(request.body)
        print("--- B2C Timeout Received ---")
        print(json.dumps(callback_data, indent=4))
        # TODO: Process B2C timeout (transaction took too long)
        # 1. Extract key identifiers: OriginatorConversationID (usually present in timeout)
        #    result = callback_data.get("Result", {})
        #    result_code = result.get("ResultCode") # Should indicate timeout
        #    result_desc = result.get("ResultDesc")
        #    originator_conversation_id = result.get("OriginatorConversationID")
        #    conversation_id = result.get("ConversationID") # May or may not be present
        #
        # 2. Idempotency Check: Use OriginatorConversationID
        #    if B2CTransactionModel.objects.filter(originator_conversation_id=originator_conversation_id, status="Timeout").exists():
        #        print(f"B2C Timeout {originator_conversation_id} already processed.")
        #        return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"})
        #
        # 3. Find the original B2C request record.
        #    try:
        #        transaction = B2CTransactionModel.objects.get(originator_conversation_id=originator_conversation_id)
        #    except B2CTransactionModel.DoesNotExist:
        #        print(f"ERROR: Original B2C transaction not found for timeout {originator_conversation_id}")
        #        return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"})
        #
        # 4. Update transaction status to 'Timeout' or 'Failed'.
        #    if transaction.status not in ["Completed", "Failed"]:
        #        transaction.status = "Timeout"
        #        transaction.error_code = result_code
        #        transaction.error_description = result_desc or "Transaction timed out"
        #        transaction.conversation_id = conversation_id # Store if available
        #        transaction.save()
        #        print(f"B2C Payment {originator_conversation_id} marked as Timeout.")
        #        # Optionally notify user/admin
        #    else:
        #        print(f"B2C Transaction {originator_conversation_id} already has final status: {transaction.status}")
        #
        return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"})
    return JsonResponse({"error": "Invalid request method"}, status=405)

@csrf_exempt
def b2b_result_callback(request):
    if request.method == "POST":
        callback_data = json.loads(request.body)
        print("--- B2B Result Received ---")
        print(json.dumps(callback_data, indent=4))
        # TODO: Process B2B result
        # 1. Extract key identifiers: ConversationID, OriginatorConversationID
        #    result = callback_data.get("Result", {})
        #    result_code = result.get("ResultCode")
        #    result_desc = result.get("ResultDesc")
        #    originator_conversation_id = result.get("OriginatorConversationID")
        #    conversation_id = result.get("ConversationID")
        #    transaction_id = result.get("TransactionID")
        #
        # 2. Idempotency Check: Use ConversationID or OriginatorConversationID
        #    if B2BTransactionModel.objects.filter(conversation_id=conversation_id).exists():
        #        print(f"B2B Result {conversation_id} already processed.")
        #        return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"})
        #
        # 3. Find the original B2B request record using OriginatorConversationID.
        #    try:
        #        transaction = B2BTransactionModel.objects.get(originator_conversation_id=originator_conversation_id)
        #    except B2BTransactionModel.DoesNotExist:
        #        print(f"ERROR: Original B2B transaction not found for {originator_conversation_id}")
        #        return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"})
        #
        # 4. Update transaction status based on result_code.
        #    if result_code == 0:
        #        transaction.status = "Completed"
        #        transaction.transaction_id = transaction_id
        #        # Extract additional details from ResultParameters
        #        result_params = result.get("ResultParameters", {}).get("ResultParameter", [])
        #        transaction_receipt = next((item["Value"] for item in result_params if item["Key"] == "TransactionReceipt"), None)
        #        # ... extract other relevant parameters like utility account balance, etc.
        #        transaction.mpesa_receipt = transaction_receipt
        #        print(f"B2B Payment {conversation_id} successful. Receipt: {transaction_receipt}")
        #    else:
        #        transaction.status = "Failed"
        #        transaction.error_code = result_code
        #        transaction.error_description = result_desc
        #        print(f"B2B Payment {conversation_id} failed. Code: {result_code}, Desc: {result_desc}")
        #
        # 5. Save changes.
        #    transaction.conversation_id = conversation_id
        #    transaction.save()
        return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"})
    return JsonResponse({"error": "Invalid request method"}, status=405)

@csrf_exempt
def b2b_timeout_callback(request):
    if request.method == "POST":
        callback_data = json.loads(request.body)
        print("--- B2B Timeout Received ---")
        print(json.dumps(callback_data, indent=4))
        # TODO: Process B2B timeout
        # 1. Extract key identifiers: OriginatorConversationID
        #    result = callback_data.get("Result", {})
        #    result_code = result.get("ResultCode")
        #    result_desc = result.get("ResultDesc")
        #    originator_conversation_id = result.get("OriginatorConversationID")
        #    conversation_id = result.get("ConversationID")
        #
        # 2. Idempotency Check: Use OriginatorConversationID
        #    if B2BTransactionModel.objects.filter(originator_conversation_id=originator_conversation_id, status="Timeout").exists():
        #        print(f"B2B Timeout {originator_conversation_id} already processed.")
        #        return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"})
        #
        # 3. Find the original B2B request record.
        #    try:
        #        transaction = B2BTransactionModel.objects.get(originator_conversation_id=originator_conversation_id)
        #    except B2BTransactionModel.DoesNotExist:
        #        print(f"ERROR: Original B2B transaction not found for timeout {originator_conversation_id}")
        #        return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"})
        #
        # 4. Update transaction status to 'Timeout' or 'Failed'.
        #    if transaction.status not in ["Completed", "Failed"]:
        #        transaction.status = "Timeout"
        #        transaction.error_code = result_code
        #        transaction.error_description = result_desc or "Transaction timed out"
        #        transaction.conversation_id = conversation_id # Store if available
        #        transaction.save()
        #        print(f"B2B Payment {originator_conversation_id} marked as Timeout.")
        #    else:
        #        print(f"B2B Transaction {originator_conversation_id} already has final status: {transaction.status}")
        return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"})
    return JsonResponse({"error": "Invalid request method"}, status=405)

@csrf_exempt
def balance_result_callback(request):
    if request.method == "POST":
        callback_data = json.loads(request.body)
        print("--- Account Balance Result Received ---")
        print(json.dumps(callback_data, indent=4))
        # TODO: Process Account Balance result
        # 1. Extract key identifiers: ConversationID, OriginatorConversationID
        #    result = callback_data.get("Result", {})
        #    result_code = result.get("ResultCode")
        #    result_desc = result.get("ResultDesc")
        #    originator_conversation_id = result.get("OriginatorConversationID")
        #    conversation_id = result.get("ConversationID")
        #
        # 2. Idempotency Check: Use ConversationID or OriginatorConversationID
        #    if BalanceCheckModel.objects.filter(conversation_id=conversation_id).exists():
        #        print(f"Balance Result {conversation_id} already processed.")
        #        return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"})
        #
        # 3. Find the original Balance Check request record (optional, depends on your tracking).
        #    try:
        #        balance_request = BalanceCheckModel.objects.get(originator_conversation_id=originator_conversation_id)
        #    except BalanceCheckModel.DoesNotExist:
        #        print(f"WARN: Original Balance Check request not found for {originator_conversation_id}")
        #        # You might still want to record the balance received
        #        balance_request = BalanceCheckModel(originator_conversation_id=originator_conversation_id)
        #
        # 4. Process based on result_code.
        #    if result_code == 0:
        #        balance_request.status = "Completed"
        #        # Extract balance details from ResultParameters (this structure can be complex)
        #        result_params = result.get("ResultParameters", {}).get("ResultParameter", [])
        #        balance_string = next((item["Value"] for item in result_params if item["Key"] == "AccountBalance"), None)
        #        # The balance string might look like: "Working|KES|45000.00|45000.00|0.00|0.00&Utility|KES|10000..."
        #        print(f"Account Balance Received: {balance_string}")
        #        # TODO: Parse the balance_string to extract specific account types and amounts
        #        # Example parsing (simplified):
        #        if balance_string:
        #            parts = balance_string.split("&")
        #            for part in parts:
        #                details = part.split("|")
        #                if len(details) > 2 and details[0] == "Working":
        #                    balance_request.working_balance = details[3] # Example: Available balance
        #                elif len(details) > 2 and details[0] == "Utility":
        #                    balance_request.utility_balance = details[3]
        #        # Store the raw string too for reference
        #        balance_request.raw_balance_details = balance_string
        #    else:
        #        balance_request.status = "Failed"
        #        balance_request.error_code = result_code
        #        balance_request.error_description = result_desc
        #        print(f"Account Balance check {conversation_id} failed. Code: {result_code}, Desc: {result_desc}")
        #
        # 5. Save changes.
        #    balance_request.conversation_id = conversation_id
        #    balance_request.save()
        return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"})
    return JsonResponse({"error": "Invalid request method"}, status=405)

@csrf_exempt
def balance_timeout_callback(request):
    if request.method == "POST":
        callback_data = json.loads(request.body)
        print("--- Account Balance Timeout Received ---")
        print(json.dumps(callback_data, indent=4))
        # TODO: Process Account Balance timeout
        return JsonResponse({"ResultCode": 0, "ResultDesc": "Accepted"})
    return JsonResponse({"error": "Invalid request method"}, status=405)

```

**Key Points for Webhooks:**

*   **`@csrf_exempt`:** This decorator is crucial because Safaricom's servers won't send a CSRF token with their POST requests. Without it, Django will reject the callback. Use this decorator *only* on webhook endpoints.
*   **Public Accessibility:** The URLs defined for callbacks (`MPESA_CALLBACK_BASE_URL` + path) must be reachable from the public internet. During development, `ngrok` is an excellent tool for this.
*   **JSON Parsing:** Callbacks arrive as JSON payloads in the request body.
*   **Response:** You *must* respond to Safaricom's callback with a JSON object containing `{"ResultCode": 0, "ResultDesc": "Accepted"}` (or similar success message) to acknowledge receipt. Failure to respond correctly might lead to Safaricom retrying the callback or deactivating the URL.
*   **Error Handling:** Implement robust error handling (`try...except`) to catch issues during processing and still provide the required response to Safaricom.
*   **Logging:** Log incoming callback data and any processing errors thoroughly for debugging.
*   **Asynchronous Processing:** For production systems, consider processing the callback data asynchronously (e.g., using Celery) to respond quickly to Safaricom and avoid timeouts if your processing logic is complex.

Now, create `mpesa_api/urls.py` to map URLs to these views:

```python
# mpesa_api/urls.py
from django.urls import path
from . import views

# Define a namespace for clarity, especially if you have multiple apps
app_name = 'mpesa_api'

urlpatterns = [
    # --- API endpoints to trigger actions ---
    path('get_token/', views.get_token_view, name='get_token'),
    path('stk_push/', views.stk_push_view, name='stk_push'),
    path('register_urls/', views.register_urls_view, name='register_urls'),
    path('simulate_c2b/', views.simulate_c2b_view, name='simulate_c2b'),
    path('b2c_payment/', views.b2c_payment_view, name='b2c_payment'),
    path('b2b_payment/', views.b2b_payment_view, name='b2b_payment'),
    path('check_balance/', views.check_balance_view, name='check_balance'),

    # --- Callback URLs for Safaricom --- 
    # These paths MUST match the ones you register/provide in API calls
    path('stk_callback/', views.stk_push_callback, name='stk_push_callback'),
    path('c2b_confirmation/', views.c2b_confirmation_callback, name='c2b_confirmation'),
    path('c2b_validation/', views.c2b_validation_callback, name='c2b_validation'),
    path('b2c_result/', views.b2c_result_callback, name='b2c_result'),
    path('b2c_timeout/', views.b2c_timeout_callback, name='b2c_timeout'),
    path('b2b_result/', views.b2b_result_callback, name='b2b_result'),
    path('b2b_timeout/', views.b2b_timeout_callback, name='b2b_timeout'),
    path('balance_result/', views.balance_result_callback, name='balance_result'),
    path('balance_timeout/', views.balance_timeout_callback, name='balance_timeout'),
]

```

Finally, include these app-specific URLs in your main project's `mpesa_project/urls.py`:

```python
# mpesa_project/urls.py
from django.contrib import admin
from django.urls import path, include # Make sure include is imported

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/mpesa/', include('mpesa_api.urls', namespace='mpesa_api')), # Include your app's URLs
    # ... other project urls
]
```

Now your Django application has endpoints like `/api/mpesa/stk_push/` to initiate payments and `/api/mpesa/stk_callback/` to receive the results from Safaricom.

## 5. Security and Final Considerations

*   **Environment Variables:** Never commit your API keys, secrets, or passkeys directly into your code. Use environment variables (`.env` file and `python-dotenv` or system environment variables) and ensure the `.env` file is in your `.gitignore`.
*   **Security Credential Encryption:** Reiterate the importance of correctly implementing the RSA encryption for the `SecurityCredential` in B2C, B2B, and Account Balance requests for production environments. Consult the official Daraja documentation for the precise steps.
*   **Input Validation:** Always validate input received in your API views (amounts, phone numbers, references) before passing it to the M-Pesa services.
*   **Error Handling & Logging:** Implement comprehensive error handling and logging in both your service functions and view handlers. This is crucial for debugging issues with API calls and callbacks.
*   **Idempotency:** Consider how to handle potential duplicate callbacks from Safaricom. Use unique transaction identifiers (like `CheckoutRequestID` or `TransID`) to ensure you don't process the same transaction twice.
*   **Database Models:** You'll likely need Django models to store transaction details, statuses, M-Pesa receipt numbers, customer information, etc., to track payments and reconcile data received from callbacks.
*   **Testing:** Thoroughly test each endpoint and callback flow in the Safaricom sandbox environment before going live.

This guide provides a solid foundation for integrating M-Pesa Daraja with Django. Remember to consult the official Safaricom Daraja API documentation for the most up-to-date details, specific error codes, and security requirements.

