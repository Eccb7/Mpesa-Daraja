
## Rails Controllers and Routes

Next, we create controllers to handle requests from your application (e.g., a frontend) to initiate M-Pesa actions and to receive the callbacks from Safaricom.

Generate a controller, for example, `MpesaApiController`:

```bash
rails generate controller Api::V1::MpesaApi --skip-template-engine --skip-assets --skip-helper
```

This creates `app/controllers/api/v1/mpesa_api_controller.rb`. We will place our M-Pesa related actions here.

```ruby
# app/controllers/api/v1/mpesa_api_controller.rb
module Api
  module V1
    class MpesaApiController < ApplicationController
      # Skip CSRF protection for API endpoints, especially webhooks
      # Consider more specific skipping if needed (e.g., only for callback actions)
      skip_before_action :verify_authenticity_token

      # --- Actions to Initiate M-Pesa Operations ---

      # Example: Get Access Token
      def get_token
        token = MpesaService.get_access_token
        if token
          render json: { access_token: token }, status: :ok
        else
          render json: { error: "Failed to get access token" }, status: :internal_server_error
        end
      end

      # Initiate STK Push
      def stk_push
        phone_number = params[:phone_number]
        amount = params[:amount]
        account_reference = params[:account_reference] || "RailsAppRef"
        transaction_desc = params[:transaction_desc] || "Payment"

        if phone_number.blank? || amount.blank?
          render json: { error: "Missing required parameters: phone_number, amount" }, status: :bad_request
          return
        end

        response = MpesaService.initiate_stk_push(phone_number, amount, account_reference, transaction_desc)
        render json: response, status: determine_status(response)
      end

      # Register C2B URLs
      def register_urls
        response = MpesaService.register_c2b_urls
        render json: response, status: determine_status(response)
      end

      # Simulate C2B Transaction
      def simulate_c2b
        phone_number = params[:phone_number]
        amount = params[:amount]
        bill_ref_number = params[:bill_ref_number] || "TestSimulateRails"

        if phone_number.blank? || amount.blank?
          render json: { error: "Missing required parameters: phone_number, amount" }, status: :bad_request
          return
        end

        response = MpesaService.simulate_c2b_transaction(amount, phone_number, bill_ref_number)
        render json: response, status: determine_status(response)
      end

      # Initiate B2C Payment
      def b2c_payment
        phone_number = params[:phone_number]
        amount = params[:amount]
        remarks = params[:remarks] || "Payment"
        occasion = params[:occasion] || "BusinessPayment"

        if phone_number.blank? || amount.blank?
          render json: { error: "Missing required parameters: phone_number, amount" }, status: :bad_request
          return
        end
        # WARNING: Security Credential encryption is crucial!
        response = MpesaService.initiate_b2c_payment(amount, phone_number, remarks, occasion)
        render json: response, status: determine_status(response)
      end

      # Initiate B2B Payment
      def b2b_payment
        receiver_shortcode = params[:receiver_shortcode]
        amount = params[:amount]
        remarks = params[:remarks] || "B2B Payment"
        account_reference = params[:account_reference] || "B2BRailsTransfer"

        if receiver_shortcode.blank? || amount.blank?
          render json: { error: "Missing required parameters: receiver_shortcode, amount" }, status: :bad_request
          return
        end
        # WARNING: Security Credential encryption is crucial!
        response = MpesaService.initiate_b2b_payment(receiver_shortcode, amount, remarks, account_reference)
        render json: response, status: determine_status(response)
      end

      # Check Account Balance
      def check_balance
        remarks = params[:remarks] || "Balance Check Rails"
        # WARNING: Security Credential encryption is crucial!
        response = MpesaService.check_account_balance(remarks)
        render json: response, status: determine_status(response)
      end

      # --- Webhook (Callback) Handler Actions ---

      def stk_callback
        # Safaricom sends POST request here
        callback_data = params.permit! # Allow all params for logging/initial processing
        Rails.logger.info "--- STK Push Callback Received ---"
        Rails.logger.info callback_data.to_json

        # Extract data safely
        body = callback_data[:Body] || {}
        stk_callback = body[:stkCallback] || {}
        result_code = stk_callback[:ResultCode]
        result_desc = stk_callback[:ResultDesc]
        checkout_request_id = stk_callback[:CheckoutRequestID]

        if result_code == 0
          # Success
          Rails.logger.info "STK Push Successful! CheckoutRequestID: #{checkout_request_id}"
          metadata = stk_callback.dig(:CallbackMetadata, :Item) || []
          amount = metadata.find { |item| item[:Name] == "Amount" }&.dig(:Value)
          receipt_number = metadata.find { |item| item[:Name] == "MpesaReceiptNumber" }&.dig(:Value)
          phone_number = metadata.find { |item| item[:Name] == "PhoneNumber" }&.dig(:Value)
          transaction_date = metadata.find { |item| item[:Name] == "TransactionDate" }&.dig(:Value)

          Rails.logger.info "  Amount: #{amount}, Receipt: #{receipt_number}, Phone: #{phone_number}, Date: #{transaction_date}"

          # TODO: Process successful payment
          # 1. Idempotency Check: Verify if this `checkout_request_id` has already been processed.
          #    transaction = Transaction.find_by(checkout_request_id: checkout_request_id)
          #    if transaction&.completed?
          #      Rails.logger.info "STK Transaction #{checkout_request_id} already completed."
          #      render json: { ResultCode: 0, ResultDesc: "Accepted" }, status: :ok
          #      return
          #    end
          #
          # 2. Find the original transaction record (create if not found, or handle error).
          #    transaction ||= Transaction.find_or_initialize_by(checkout_request_id: checkout_request_id)
          #    # You might also use merchant_request_id if checkout_request_id lookup fails
          #
          # 3. Update transaction status to 'Completed'.
          #    transaction.status = 'completed'
          #
          # 4. Store M-Pesa receipt number and other details.
          #    transaction.mpesa_receipt = receipt_number
          #    transaction.transaction_date = Time.strptime(transaction_date.to_s, '%Y%m%d%H%M%S') rescue Time.now # Parse date
          #    transaction.phone_number = phone_number
          #    transaction.paid_amount = amount
          #
          # 5. Save the transaction record.
          #    unless transaction.save
          #      Rails.logger.error "Failed to save successful STK transaction #{checkout_request_id}: #{transaction.errors.full_messages.join(', ')}"
          #      # Decide if you should still return success to Safaricom
          #    end
          #
          # 6. Fulfill the order/service associated with the transaction.
          #    # fulfill_order(transaction.order_id)
          #    Rails.logger.info "Order fulfillment triggered for transaction #{transaction.id}"
          #
          # 7. Optionally, notify the user.

        else
          # Failure/Cancelled
          Rails.logger.error "STK Push Failed/Cancelled for CheckoutRequestID: #{checkout_request_id}. Code: #{result_code}, Desc: #{result_desc}"
          # TODO: Process failed payment
          # 1. Idempotency Check: Verify if this `checkout_request_id` has already been marked failed/cancelled.
          #    transaction = Transaction.find_by(checkout_request_id: checkout_request_id)
          #    if transaction&.failed_or_cancelled?
          #      Rails.logger.info "STK Transaction #{checkout_request_id} already marked failed/cancelled."
          #      render json: { ResultCode: 0, ResultDesc: "Accepted" }, status: :ok
          #      return
          #    end
          #
          # 2. Find the original transaction record.
          #    transaction ||= Transaction.find_or_initialize_by(checkout_request_id: checkout_request_id)
          #
          # 3. Update status to 'failed' or 'cancelled'.
          #    transaction.status = (result_code == 1032) ? 'cancelled' : 'failed' # Example mapping
          #
          # 4. Store error code and description.
          #    transaction.error_code = result_code
          #    transaction.error_description = result_desc
          #
          # 5. Save the transaction record.
          #    unless transaction.save
          #      Rails.logger.error "Failed to save failed/cancelled STK transaction #{checkout_request_id}: #{transaction.errors.full_messages.join(", ")}"
          #    end
          #
          # 6. Optionally, notify the user of the failure.
        end

        # Respond to Safaricom
        render json: { ResultCode: 0, ResultDesc: "Accepted" }, status: :ok
      rescue StandardError => e
        Rails.logger.error "Error processing STK callback: #{e.message}"
        Rails.logger.error e.backtrace.join("\n")
        render json: { ResultCode: 1, ResultDesc: "Failed" }, status: :internal_server_error
      end

      def c2b_confirmation
        callback_data = params.permit!
        Rails.logger.info "--- C2B Confirmation Received ---"
        Rails.logger.info callback_data.to_json

        # Extract data
        trans_id = callback_data[:TransID]
        trans_amount = callback_data[:TransAmount]
        msisdn = callback_data[:MSISDN]
        bill_ref_number = callback_data[:BillRefNumber]

        # TODO: Process C2B confirmation
        # 1. Idempotency Check: Verify if this `trans_id` has already been processed.
        #    if C2bTransaction.exists?(transaction_id: trans_id)
        #      Rails.logger.info "C2B Transaction #{trans_id} already processed."
        #      render json: { ResultCode: 0, ResultDesc: "Accepted" }, status: :ok
        #      return
        #    end
        #
        # 2. Record the transaction details in your database.
        #    c2b_transaction = C2bTransaction.new(
        #      transaction_type: callback_data[:TransactionType],
        #      transaction_id: trans_id,
        #      transaction_time: callback_data[:TransTime], # Consider parsing
        #      amount: trans_amount,
        #      business_short_code: callback_data[:BusinessShortCode],
        #      bill_ref_number: bill_ref_number,
        #      invoice_number: callback_data[:InvoiceNumber],
        #      org_account_balance: callback_data[:OrgAccountBalance],
        #      third_party_trans_id: callback_data[:ThirdPartyTransID],
        #      msisdn: msisdn,
        #      first_name: callback_data[:FirstName],
        #      middle_name: callback_data[:MiddleName],
        #      last_name: callback_data[:LastName]
        #    )
        #    unless c2b_transaction.save
        #      Rails.logger.error "Failed to save C2B transaction #{trans_id}: #{c2b_transaction.errors.full_messages.join(", ")}"
        #      # Decide how to handle - still accept Safaricom's request?
        #    else
        #      Rails.logger.info "C2B Transaction #{trans_id} recorded successfully."
        #    end
        #
        # 3. Update account balances, order status, etc.
        #    # update_order_status(bill_ref_number, 'paid')
        #
        # 4. Optionally reconcile with validation data.

        render json: { ResultCode: 0, ResultDesc: "Accepted" }, status: :ok
      rescue StandardError => e
        Rails.logger.error "Error processing C2B Confirmation: #{e.message}"
        render json: { ResultCode: 1, ResultDesc: "Failed" }, status: :internal_server_error
      end

      def c2b_validation
        callback_data = params.permit!
        Rails.logger.info "--- C2B Validation Received ---"
        Rails.logger.info callback_data.to_json

        # Extract data
        bill_ref_number = callback_data[:BillRefNumber]
        msisdn = callback_data[:MSISDN]
        trans_amount = callback_data[:TransAmount]

        # TODO: Implement validation logic
        # 1. Retrieve necessary information based on `bill_ref_number`.
        #    order = Order.find_by(reference: bill_ref_number)
        #    unless order
        #      Rails.logger.warn "C2B Validation failed: Bill reference #{bill_ref_number} not found."
        #      render json: { ResultCode: "C2B00011", ResultDesc: "Rejected" }, status: :ok
        #      return
        #    end
        #
        # 2. Perform checks based on your business rules.
        #    expected_amount = order.amount
        #    is_active = order.active?
        #
        #    validation_passed = true
        #    rejection_reason = "Accepted"
        #    rejection_code = 0
        #
        #    unless is_active
        #      validation_passed = false
        #      rejection_reason = "Reference inactive"
        #      rejection_code = "C2B00012"
        #    end
        #
        #    # Use BigDecimal for amount comparison to avoid floating point issues
        #    if validation_passed && BigDecimal(trans_amount.to_s) != BigDecimal(expected_amount.to_s)
        #      validation_passed = false
        #      rejection_reason = "Incorrect amount"
        #      rejection_code = "C2B00013"
        #    end
        #
        #    # Add more checks (e.g., customer allowed?)
        #
        # 3. Respond based on validation result.
        #    if validation_passed
        #      Rails.logger.info "C2B Validation successful for #{bill_ref_number}"
        #      render json: { ResultCode: 0, ResultDesc: "Accepted" }, status: :ok
        #    else
        #      Rails.logger.warn "C2B Validation failed for #{bill_ref_number}: #{rejection_reason}"
        #      render json: { ResultCode: rejection_code, ResultDesc: rejection_reason }, status: :ok
        #    end
      rescue StandardError => e
        Rails.logger.error "Error processing C2B Validation: #{e.message}"
        render json: { ResultCode: 1, ResultDesc: "Failed" }, status: :internal_server_error
      end

      # --- Placeholder Callbacks for B2C, B2B, Balance --- 
      # (Implement similarly based on Daraja documentation for result/timeout data structures)

      def b2c_result
        callback_data = params.permit!
        Rails.logger.info "--- B2C Result Received ---"
        Rails.logger.info callback_data.to_json
        # TODO: Process B2C result (check Result.ResultCode, extract details)
        # 1. Extract key identifiers: ConversationID, OriginatorConversationID
        #    result = callback_data.dig(:Result) || {}
        #    result_code = result[:ResultCode]
        #    result_desc = result[:ResultDesc]
        #    originator_conversation_id = result[:OriginatorConversationID]
        #    conversation_id = result[:ConversationID]
        #    transaction_id = result[:TransactionID]
        #
        # 2. Idempotency Check: Use ConversationID or OriginatorConversationID
        #    if B2cTransaction.exists?(conversation_id: conversation_id)
        #      Rails.logger.info "B2C Result #{conversation_id} already processed."
        #      render json: { ResultCode: 0, ResultDesc: "Accepted" }, status: :ok
        #      return
        #    end
        #
        # 3. Find the original B2C request record using OriginatorConversationID.
        #    transaction = B2cTransaction.find_by(originator_conversation_id: originator_conversation_id)
        #    unless transaction
        #      Rails.logger.error "Original B2C transaction not found for #{originator_conversation_id}"
        #      # Log and accept, or handle as needed
        #      render json: { ResultCode: 0, ResultDesc: "Accepted" }, status: :ok
        #      return
        #    end
        #
        # 4. Update transaction status based on result_code.
        #    if result_code == 0
        #      transaction.status = "completed"
        #      transaction.transaction_id = transaction_id # M-Pesa transaction ID
        #      # Extract additional details from ResultParameters
        #      result_params = result.dig(:ResultParameters, :ResultParameter) || []
        #      transaction_receipt = result_params.find { |p| p[:Key] == "TransactionReceipt" }&.dig(:Value)
        #      # ... extract other relevant parameters
        #      transaction.mpesa_receipt = transaction_receipt
        #      Rails.logger.info "B2C Payment #{conversation_id} successful. Receipt: #{transaction_receipt}"
        #    else
        #      transaction.status = "failed"
        #      transaction.error_code = result_code
        #      transaction.error_description = result_desc
        #      Rails.logger.error "B2C Payment #{conversation_id} failed. Code: #{result_code}, Desc: #{result_desc}"
        #    end
        #
        # 5. Save changes and trigger follow-up actions.
        #    transaction.conversation_id = conversation_id # Store final ConversationID
        #    unless transaction.save
        #      Rails.logger.error "Failed to save B2C result for #{conversation_id}: #{transaction.errors.full_messages.join(", ")}"
        #    end
        #    # notify_user_of_payout_status(transaction) render json: { ResultCode: 0, ResultDesc: "Accepted" }, status: :ok
      end

      def b2c_timeout
        callback_data = params.permit!
        Rails.logger.info "--- B2C Timeout Received ---"
        Rails.logger.info callback_data.to_json
        # TODO: Process B2C timeout (mark transaction as timed out)
        # 1. Extract key identifiers: OriginatorConversationID
        #    result = callback_data.dig(:Result) || {}
        #    result_code = result[:ResultCode]
        #    result_desc = result[:ResultDesc]
        #    originator_conversation_id = result[:OriginatorConversationID]
        #    conversation_id = result[:ConversationID]
        #
        # 2. Idempotency Check: Use OriginatorConversationID
        #    if B2cTransaction.exists?(originator_conversation_id: originator_conversation_id, status: "timeout")
        #      Rails.logger.info "B2C Timeout #{originator_conversation_id} already processed."
        #      render json: { ResultCode: 0, ResultDesc: "Accepted" }, status: :ok
        #      return
        #    end
        #
        # 3. Find the original B2C request record.
        #    transaction = B2cTransaction.find_by(originator_conversation_id: originator_conversation_id)
        #    unless transaction
        #      Rails.logger.error "Original B2C transaction not found for timeout #{originator_conversation_id}"
        #      render json: { ResultCode: 0, ResultDesc: "Accepted" }, status: :ok
        #      return
        #    end
        #
        # 4. Update transaction status to 'timeout' if not already completed/failed.
        #    if transaction.pending? || transaction.processing? # Check current status
        #      transaction.status = "timeout"
        #      transaction.error_code = result_code
        #      transaction.error_description = result_desc || "Transaction timed out"
        #      transaction.conversation_id = conversation_id # Store if available
        #      unless transaction.save
        #        Rails.logger.error "Failed to save B2C timeout for #{originator_conversation_id}: #{transaction.errors.full_messages.join(", ")}"
        #      else
        #        Rails.logger.warn "B2C Payment #{originator_conversation_id} marked as Timeout."
        #        # Optionally notify user/admin
        #      end
        #    else
        #      Rails.logger.info "B2C Transaction #{originator_conversation_id} already has final status: #{transaction.status}"
        #    end render json: { ResultCode: 0, ResultDesc: "Accepted" }, status: :ok
      end

      def b2b_result
        callback_data = params.permit!
        Rails.logger.info "--- B2B Result Received ---"
        Rails.logger.info callback_data.to_json
        # TODO: Process B2B result
        # 1. Extract key identifiers: ConversationID, OriginatorConversationID
        #    result = callback_data.dig(:Result) || {}
        #    result_code = result[:ResultCode]
        #    result_desc = result[:ResultDesc]
        #    originator_conversation_id = result[:OriginatorConversationID]
        #    conversation_id = result[:ConversationID]
        #    transaction_id = result[:TransactionID]
        #
        # 2. Idempotency Check: Use ConversationID or OriginatorConversationID
        #    if B2bTransaction.exists?(conversation_id: conversation_id)
        #      Rails.logger.info "B2B Result #{conversation_id} already processed."
        #      render json: { ResultCode: 0, ResultDesc: "Accepted" }, status: :ok
        #      return
        #    end
        #
        # 3. Find the original B2B request record using OriginatorConversationID.
        #    transaction = B2bTransaction.find_by(originator_conversation_id: originator_conversation_id)
        #    unless transaction
        #      Rails.logger.error "Original B2B transaction not found for #{originator_conversation_id}"
        #      render json: { ResultCode: 0, ResultDesc: "Accepted" }, status: :ok
        #      return
        #    end
        #
        # 4. Update transaction status based on result_code.
        #    if result_code == 0
        #      transaction.status = "completed"
        #      transaction.transaction_id = transaction_id
        #      # Extract additional details from ResultParameters
        #      result_params = result.dig(:ResultParameters, :ResultParameter) || []
        #      transaction_receipt = result_params.find { |p| p[:Key] == "TransactionReceipt" }&.dig(:Value)
        #      # ... extract other relevant parameters
        #      transaction.mpesa_receipt = transaction_receipt
        #      Rails.logger.info "B2B Payment #{conversation_id} successful. Receipt: #{transaction_receipt}"
        #    else
        #      transaction.status = "failed"
        #      transaction.error_code = result_code
        #      transaction.error_description = result_desc
        #      Rails.logger.error "B2B Payment #{conversation_id} failed. Code: #{result_code}, Desc: #{result_desc}"
        #    end
        #
        # 5. Save changes.
        #    transaction.conversation_id = conversation_id
        #    unless transaction.save
        #      Rails.logger.error "Failed to save B2B result for #{conversation_id}: #{transaction.errors.full_messages.join(", ")}"
        #    end render json: { ResultCode: 0, ResultDesc: "Accepted" }, status: :ok
      end

      def b2b_timeout
        callback_data = params.permit!
        Rails.logger.info "--- B2B Timeout Received ---"
        Rails.logger.info callback_data.to_json
        # TODO: Process B2B timeout
        # 1. Extract key identifiers: OriginatorConversationID
        #    result = callback_data.dig(:Result) || {}
        #    result_code = result[:ResultCode]
        #    result_desc = result[:ResultDesc]
        #    originator_conversation_id = result[:OriginatorConversationID]
        #    conversation_id = result[:ConversationID]
        #
        # 2. Idempotency Check: Use OriginatorConversationID
        #    if B2bTransaction.exists?(originator_conversation_id: originator_conversation_id, status: "timeout")
        #      Rails.logger.info "B2B Timeout #{originator_conversation_id} already processed."
        #      render json: { ResultCode: 0, ResultDesc: "Accepted" }, status: :ok
        #      return
        #    end
        #
        # 3. Find the original B2B request record.
        #    transaction = B2bTransaction.find_by(originator_conversation_id: originator_conversation_id)
        #    unless transaction
        #      Rails.logger.error "Original B2B transaction not found for timeout #{originator_conversation_id}"
        #      render json: { ResultCode: 0, ResultDesc: "Accepted" }, status: :ok
        #      return
        #    end
        #
        # 4. Update transaction status to 'timeout' if not already completed/failed.
        #    if transaction.pending? || transaction.processing?
        #      transaction.status = "timeout"
        #      transaction.error_code = result_code
        #      transaction.error_description = result_desc || "Transaction timed out"
        #      transaction.conversation_id = conversation_id # Store if available
        #      unless transaction.save
        #        Rails.logger.error "Failed to save B2B timeout for #{originator_conversation_id}: #{transaction.errors.full_messages.join(", ")}"
        #      else
        #        Rails.logger.warn "B2B Payment #{originator_conversation_id} marked as Timeout."
        #      end
        #    else
        #      Rails.logger.info "B2B Transaction #{originator_conversation_id} already has final status: #{transaction.status}"
        #    end render json: { ResultCode: 0, ResultDesc: "Accepted" }, status: :ok
      end

      def balance_result
        callback_data = params.permit!
        Rails.logger.info "--- Account Balance Result Received ---"
        Rails.logger.info callback_data.to_json
        # TODO: Process Account Balance result (extract balance from Result.ResultParameters)
        # 1. Extract key identifiers: ConversationID, OriginatorConversationID
        #    result = callback_data.dig(:Result) || {}
        #    result_code = result[:ResultCode]
        #    result_desc = result[:ResultDesc]
        #    originator_conversation_id = result[:OriginatorConversationID]
        #    conversation_id = result[:ConversationID]
        #
        # 2. Idempotency Check: Use ConversationID or OriginatorConversationID
        #    if BalanceCheck.exists?(conversation_id: conversation_id)
        #      Rails.logger.info "Balance Result #{conversation_id} already processed."
        #      render json: { ResultCode: 0, ResultDesc: "Accepted" }, status: :ok
        #      return
        #    end
        #
        # 3. Find/Initialize the Balance Check record (optional, depends on tracking needs).
        #    balance_check = BalanceCheck.find_or_initialize_by(originator_conversation_id: originator_conversation_id)
        #
        # 4. Process based on result_code.
        #    if result_code == 0
        #      balance_check.status = "completed"
        #      # Extract balance details from ResultParameters
        #      result_params = result.dig(:ResultParameters, :ResultParameter) || []
        #      balance_string = result_params.find { |p| p[:Key] == "AccountBalance" }&.dig(:Value)
        #      Rails.logger.info "Account Balance Received: #{balance_string}"
        #      # TODO: Parse the complex balance_string (e.g., "Working|KES|...&Utility|KES|...")
        #      # Example parsing (simplified):
        #      if balance_string.present?
        #        balance_check.raw_balance_details = balance_string
        #        balance_string.split("&").each do |part|
        #          details = part.split("|")
        #          if details.length > 3
        #            account_type = details[0]
        #            available_balance = details[3]
        #            case account_type
        #            when "Working"
        #              balance_check.working_balance = available_balance
        #            when "Utility"
        #              balance_check.utility_balance = available_balance
        #            # Add other account types if needed
        #            end
        #          end
        #        end
        #      end
        #    else
        #      balance_check.status = "failed"
        #      balance_check.error_code = result_code
        #      balance_check.error_description = result_desc
        #      Rails.logger.error "Account Balance check #{conversation_id} failed. Code: #{result_code}, Desc: #{result_desc}"
        #    end
        #
        # 5. Save changes.
        #    balance_check.conversation_id = conversation_id
        #    unless balance_check.save
        #      Rails.logger.error "Failed to save Balance Check result for #{conversation_id}: #{balance_check.errors.full_messages.join(", ")}"
        #    end render json: { ResultCode: 0, ResultDesc: "Accepted" }, status: :ok
      end

      def balance_timeout
        callback_data = params.permit!
        Rails.logger.info "--- Account Balance Timeout Received ---"
        Rails.logger.info callback_data.to_json
        # TODO: Process Account Balance timeout
        render json: { ResultCode: 0, ResultDesc: "Accepted" }, status: :ok
      end

      private

      # Helper to determine HTTP status based on MpesaService response
      def determine_status(response)
        if response.is_a?(Hash) && response[:error]
          # Crude mapping, refine as needed
          case response[:status_code]
          when 400..499
            :bad_request
          when 500..599
            :internal_server_error
          else
            response[:error] == 'Failed to get access token' ? :internal_server_error : :unprocessable_entity
          end
        else
          :ok
        end
      end

    end
  end
end
```

Now, define the routes in `config/routes.rb`:

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # Define your application routes per the DSL in https://guides.rubyonrails.org/routing.html

  # API routes for M-Pesa
  namespace :api do
    namespace :v1 do
      namespace :mpesa do
        # --- Routes to trigger actions ---
        get  :get_token, to: "mpesa_api#get_token"
        post :stk_push, to: "mpesa_api#stk_push"
        post :register_urls, to: "mpesa_api#register_urls"
        post :simulate_c2b, to: "mpesa_api#simulate_c2b"
        post :b2c_payment, to: "mpesa_api#b2c_payment"
        post :b2b_payment, to: "mpesa_api#b2b_payment"
        post :check_balance, to: "mpesa_api#check_balance"

        # --- Callback routes for Safaricom (MUST match URLs in MpesaService) ---
        post :stk_callback, to: "mpesa_api#stk_callback"
        post :c2b_confirmation, to: "mpesa_api#c2b_confirmation"
        post :c2b_validation, to: "mpesa_api#c2b_validation"
        post :b2c_result, to: "mpesa_api#b2c_result"
        post :b2c_timeout, to: "mpesa_api#b2c_timeout"
        post :b2b_result, to: "mpesa_api#b2b_result"
        post :b2b_timeout, to: "mpesa_api#b2b_timeout"
        post :balance_result, to: "mpesa_api#balance_result"
        post :balance_timeout, to: "mpesa_api#balance_timeout"
      end
    end
  end

  # Defines the root path route ("/")
  # root "articles#index"
end
```

This sets up namespaced API routes like `/api/v1/mpesa/stk_push` and `/api/v1/mpesa/stk_callback`. The controller handles parameter extraction, calls the `MpesaService`, and renders JSON responses. Crucially, the callback actions log the incoming data, include `TODO` comments for processing logic, and respond to Safaricom with the required `{"ResultCode": 0, "ResultDesc": "Accepted"}`.

## 5. Security and Final Considerations (Rails)

*   **Environment Variables:** Use `dotenv-rails` and keep `.env` out of version control.
*   **CSRF Protection:** `skip_before_action :verify_authenticity_token` is used broadly here. For better security, you might apply it only to the specific callback actions if other actions are called from within your Rails app with CSRF tokens.
*   **Security Credential Encryption:** This cannot be stressed enough. Implement the RSA encryption for B2C/B2B/Balance credentials using `OpenSSL` and the correct M-Pesa certificate.
*   **Parameter Handling:** Use `params.require` and `params.permit` more strictly in production controllers once you finalize the expected parameters.
*   **Error Handling & Logging:** Rails logging is used here. Enhance it with more context and potentially use dedicated logging gems or services.
*   **Idempotency:** Use unique IDs (`CheckoutRequestID`, `TransID`, `ConversationID`) from callbacks to prevent double-processing. Check if a transaction with that ID has already been processed before acting.
*   **Database Models:** Use ActiveRecord models to store transaction data, status, receipt numbers, etc., linking them to users or orders.
*   **Background Jobs:** For potentially long-running callback processing, use a background job system like Sidekiq or Delayed Job to process the data asynchronously. This allows the controller to respond quickly to Safaricom.
*   **Testing:** Use RSpec or Minitest to write tests for your `MpesaService` and controller actions. Mock the HTTP requests to avoid hitting the actual M-Pesa API during tests.

This guide provides the Rails structure for M-Pesa integration. Always refer to the official Safaricom Daraja API documentation for the latest specifications and requirements.

