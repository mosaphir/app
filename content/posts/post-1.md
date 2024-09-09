---
title: How to make a Telegram shopping bot with Google Apps Script and Google Sheets.
image: /images/blog/01.jpeg
author:
  name: Abdullah Al Shifat
  avatar: /images/author/abdullah.jpg
date: 2024-08-04T05:00:00Z
draft: false
---

To build a Telegram shopping bot with Google Apps Script and Google Sheets that supports payments via Crypto Pay API, we need to integrate several components:

1. **Google Sheets**: To manage products and order data.
2. **Google Apps Script**: To handle bot logic and interact with Telegram.
3. **Crypto Pay API**: For accepting cryptocurrency payments via Telegram.

### Steps:

#### 1. Set Up a Telegram Bot
1. Create a bot with [BotFather](https://t.me/BotFather), as explained earlier, and get the `BOT_TOKEN`.
2. Create an account at [Crypto Pay](https://crypto-pay.com/), which is a service to accept crypto payments on Telegram. Get the `API_TOKEN` from Crypto Pay.

#### 2. Create a Google Sheet for Products
1. Set up a Google Sheet to manage your products, with columns like:
   - `Product ID`
   - `Name`
   - `Description`
   - `Price (Crypto)`
   - `Stock`
   - `Image URL` (optional)

   Example data:

   | Product ID | Name       | Description  | Price (Crypto) | Stock | Image URL  |
   |------------|------------|--------------|----------------|-------|------------|
   | 1          | T-shirt    | Cool T-shirt | 0.002 BTC      | 10    | Image Link |
   | 2          | Sneakers   | Running shoes| 0.005 BTC      | 5     | Image Link |
   | 3          | Sunglasses | Stylish      | 0.001 BTC      | 8     | Image Link |

#### 3. Write Google Apps Script for the Bot

1. Open the Google Sheet, then go to **Extensions > Apps Script**.
2. In the Apps Script editor, replace any existing code with the following:

```javascript
// Global variables
const SHEET_ID = "your-google-sheet-id";  // Replace with your Google Sheet ID
const BOT_TOKEN = "your-telegram-bot-token"; // Replace with your Telegram bot token
const CRYPTO_PAY_API_TOKEN = "your-crypto-pay-api-token";  // Replace with your Crypto Pay API token
const SHEET_NAME = "Sheet1";  // Change if your sheet name is different

function doPost(e) {
  const contents = JSON.parse(e.postData.contents);
  
  if (contents.message) {
    const chatId = contents.message.chat.id;
    const messageText = contents.message.text.toLowerCase();
    
    if (messageText === '/start') {
      sendMessage(chatId, "Welcome to the crypto shopping bot! Type /products to see the available items.");
    } else if (messageText === '/products') {
      const products = getProducts();
      let response = "Here are the available products:\n\n";
      products.forEach(product => {
        response += `${product[1]}: ${product[3]} BTC - /buy_${product[0]}\n`;
      });
      sendMessage(chatId, response);
    } else if (messageText.startsWith('/buy_')) {
      const productId = messageText.split('_')[1];
      handlePurchase(chatId, productId);
    } else {
      sendMessage(chatId, "I didn't understand that. Type /products to see available products.");
    }
  }

  return ContentService.createTextOutput(JSON.stringify({ status: 'ok' })).setMimeType(ContentService.MimeType.JSON);
}

function sendMessage(chatId, text) {
  const url = `https://api.telegram.org/bot${BOT_TOKEN}/sendMessage`;
  const payload = {
    'method': 'post',
    'payload': {
      'chat_id': String(chatId),
      'text': text
    }
  };
  UrlFetchApp.fetch(url, payload);
}

function getProducts() {
  const sheet = SpreadsheetApp.openById(SHEET_ID).getSheetByName(SHEET_NAME);
  return sheet.getRange(2, 1, sheet.getLastRow() - 1, sheet.getLastColumn()).getValues();
}

function handlePurchase(chatId, productId) {
  const products = getProducts();
  const product = products.find(p => p[0] == productId);
  
  if (product && product[4] > 0) {
    const price = product[3].replace(' BTC', ''); // Get the product price in BTC
    const orderId = new Date().getTime(); // Unique order ID
    const paymentUrl = createCryptoPayInvoice(chatId, price, product[1], orderId);
    
    sendMessage(chatId, `You are buying ${product[1]} for ${product[3]} BTC. Complete the payment using this link: ${paymentUrl}`);
    // Reduce stock by 1 after confirming payment (this part requires additional logic)
  } else {
    sendMessage(chatId, `Sorry, ${product ? product[1] : 'this product'} is out of stock.`);
  }
}

function createCryptoPayInvoice(chatId, price, productName, orderId) {
  const url = "https://pay.crypto-pay.com/api/createInvoice";
  const payload = {
    "amount": parseFloat(price),  // Convert the price to a float
    "currency": "BTC",  // Define the crypto currency
    "order_id": String(orderId),  // Unique order ID for tracking
    "description": `Purchase of ${productName}`,
    "paid_btn_name": "Go to Store",  // Button after payment
    "paid_btn_url": "https://t.me/yourbot",  // Link after payment
    "is_fiat": false  // Make sure it's treated as crypto, not fiat
  };
  
  const options = {
    'method': 'post',
    'headers': {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${CRYPTO_PAY_API_TOKEN}`
    },
    'payload': JSON.stringify(payload)
  };
  
  const response = UrlFetchApp.fetch(url, options);
  const responseData = JSON.parse(response.getContentText());

  if (responseData.ok) {
    return responseData.result.pay_url;  // Return the payment URL
  } else {
    return "Error: Couldn't create invoice.";
  }
}
```

#### 4. Replace Placeholders
- Replace `your-google-sheet-id` with your actual Google Sheet ID.
- Replace `your-telegram-bot-token` with your Telegram bot token.
- Replace `your-crypto-pay-api-token` with your Crypto Pay API token.

#### 5. Deploy the Script as a Web App
1. In the Apps Script editor, go to **Deploy > Test deployments**.
2. Select **Deploy as web app**.
   - **Execute the app as**: Me
   - **Who has access**: Anyone (even anonymous).
3. Copy the `web app URL` after deployment.

#### 6. Set Up Telegram Webhook
Set the webhook for your bot using the web app URL:

```bash
curl -F "url=https://script.google.com/macros/s/your-deployment-id/exec" https://api.telegram.org/bot<YOUR_BOT_TOKEN>/setWebhook
```

Replace `your-deployment-id` with your web app URL.

#### 7. Test the Bot
1. Start chatting with your bot on Telegram.
2. Type `/start` to begin, and `/products` to see the available products.
3. Type `/buy_productID` to simulate a purchase, and you'll receive a payment link from Crypto Pay.
4. Follow the payment link and pay in cryptocurrency to complete the purchase.

#### 8. Automate Stock Reduction After Payment (Optional)
To automate stock reduction after payment confirmation, you'll need to:
- Set up a webhook with Crypto Pay to notify your bot of successful payments.
- Reduce stock in the Google Sheet after a successful payment.

You can follow the Crypto Pay API documentation for setting up payment notifications:
[Crypto Pay API Documentation](https://crypto-pay.com/docs).

### Conclusion

This bot provides a basic shopping experience where users can browse products, generate crypto payment invoices using the Crypto Pay API, and track orders. You can further extend it to handle more complex workflows like order tracking, automatic stock management, and payment confirmations using webhooks.
