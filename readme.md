# Razorpay Payment Integration with Express.js

## How Razorpay works with a Node.js server:

1. `Setup:` Sign up on Razorpay and get your API keys.
2. `Install SDK:` Install the Razorpay Node.js SDK using npm install razorpay.
3. `Create Order:` On your Node.js server, create a payment order using the Razorpay API. This generates an order ID.
4. `Frontend Integration:` Use Razorpay's Checkout form on your frontend to handle payments. Pass the order ID and other payment details to the form.
5. `Payment Verification:` After a successful payment, Razorpay sends a payment ID and signature to your frontend. Send these to your server to verify the payment using a cryptographic signature.
6. `Optional Features:` You can also handle refunds and set up webhooks to handle payment events automatically.

## Prerequisites

This guide will help you set up Razorpay payment integration with Express.js.It includes creating orders, handling payments, verifying payments, and saving payment details in MongoDB.

1. Node.js installed
2. MongoDB installed and running
3. Razorpay account(for API keys)

## Setup

### 1. Install Razorpay

Install Razorpay using npm:

```
npm install razorpay
```

### 2. Create an Instance to Access Resources from Razorpay API

In your app.js file, create an instance of Razorpay:

```
require('dotenv').config();
const Razorpay = require('razorpay');

const razorpay = new Razorpay({
  key_id: process.env.RAZORPAY_KEY_ID,
  key_secret: process.env.RAZORPAY_KEY_SECRET,
});
```

#### Explanation

user jab bhi koi order krta hai to uss order me sari cheeje jaise amount, currency, order_id etc hoti hai. user ne order kiya wo server ke paas jata hai aur server iss code ki madad se razorpay ko request bhejta hai. razorpay se order create krwane ke liye.

### 3. Setup MongoDB Connection

Connect to MongoDB:

```
const mongoose = require("mongoose");

function connectToDb() {
  mongoose
    .connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    })
    .then(() => console.log("MongoDB connected"))
    .catch((err) => console.log(`db error: `, err));
}

module.exports = connectToDb;

```

### 4. Define the Payment Schema

First, define the Payment schema using Mongoose.Create a new file called models/Payment.js:

```
const mongoose = require('mongoose');

const paymentSchema = new mongoose.Schema({
orderId: {
type: String,
required: true,
},
paymentId: {
type: String,
},
signature: {
type: String,
},
amount: {
type: Number,
required: true,
},
currency: {
type: String,
required: true,
},
status: {
type: String,
default: 'pending',
},
}, { timestamps: true });

const Payment = mongoose.model('Payment', paymentSchema);

module.exports = Payment;
```

#### Explanation

- Kisi bhi payment ko verify krne ke liye orderId, paymentId and signature mandatory hota hai.
- Jab server user ko payment krne ka option deta hai usi time pr server apne paas ek status bana leta hai jiski default state pending hoti hai. user ke payment krne ke baad hamar server razorpay ke server ki madad se verify krta hai ki payment success huaa hai ya failed huaa hai aur usi ke aadhar pr iss status ko change kiya jata h.

### 5. Import the Payment Model in app.js

Import the Payment model in your `app.js` file:

```
const Payment = require('./models/payment');
```

### 6. Create a Checkout Page

Create an HTML file index.html for your frontend:

```
<!DOCTYPE html>
<html>

<head>
  <title>Razorpay Integration</title>
  <link rel='stylesheet' href='/stylesheets/style.css' />
</head>

<body>
  <h1>Razorpay Integration</h1>
  <p>Welcome to Razorpay Integration</p>

<button id="rzp-button1">Pay with Razorpay</button>

</body>

</html>
```

### 7. Make a POST Route for URL = '/create/orderId' on `app.js`

In your app.js file, create a POST route to create an order:

```
app.post('/create/orderId', async (req, res) => {
const options = {
amount: 5000 * 100, // amount in smallest currency unit
currency: "INR",
};
try {
const order = await razorpay.orders.create(options);
res.send(order);

    const newPayment = await Payment.create({
      orderId: order.id,
      amount: order.amount,
      currency: order.currency,
      status: 'pending',
    });

} catch (error) {
res.status(500).send('Error creating order');
}
});

```

#### Explanation

Yaha humara server razorpay ke server ko ek order create krne ko kehta hai. aur ye order ko create krne ke liye aap `2. Create an Instance to Access Resources from Razorpay API` me likhe key_id aur key_secret ka use karo.

### 8. Add Axios CDN on Checkout Page

Ensure the Axios and Jquery CDN is added in the`index.html` file(already included in step 4):

```
<script src="https://checkout.razorpay.com/v1/checkout.js"></script>
<script src="https://cdn.jsdelivr.net/npm/axios/dist/axios.min.js"></script>

```

### 9. Generate Order Code in Checkout Page Inside Script Tags

Add the AJAX request to generate the order inside the <script> tags in index.html:

```

<script>
document.getElementById('rzp-button1').onclick = function(e) {
  axios.post('/create/orderId')
    .then(function (response) {
      var options = {
        "key": "YOUR_RAZORPAY_KEY_ID", // Enter the Key ID generated from the Dashboard
        "amount": response.data.amount, // Amount in currency subunits. Default currency is INR.
        "currency": response.data.currency,
        "name": "YOUR_COMPANY_NAME",
        "description": "Test Transaction",
        "image": "https://example.com/your_logo",
        "order_id": response.data.id,
        "handler": function(response) {
          axios.post('/api/payment/verify', {
            razorpayOrderId: response.razorpay_order_id,
            razorpayPaymentId: response.razorpay_payment_id,
            signature: response.razorpay_signature
          })
          .then(function (response) {
            alert('Payment verified successfully');
          })
          .catch(function (error) {
            console.error(error);
          });
        },
        "prefill": {
          "name": "Gaurav Kumar",
          "email": "gaurav.kumar@example.com",
          "contact": "9000090000"
        },
        "notes": {
          "address": "Razorpay Corporate Office"
        },
        "theme": {
          "color": "#3399cc"
        }
      };
      var rzp1 = new Razorpay(options);
      rzp1.on('payment.failed', function(response) {
        alert('Payment Failed');
        alert('Error Code: ' + response.error.code);
        alert('Description: ' + response.error.description);
        alert('Source: ' + response.error.source);
        alert('Step: ' + response.error.step);
        alert('Reason: ' + response.error.reason);
        alert('Order ID: ' + response.error.metadata.order_id);
        alert('Payment ID: ' + response.error.metadata.payment_id);
      });
      rzp1.open();
      e.preventDefault();
    })
    .catch(function (error) {
      console.error(error);
    });
};
</script>

```

#### Explanation

Jab user payment krne ke liye `rzp-button1` iss button pr click krta hai to, axios `/create/orderId` iss route pr jakr ek order create kr deta hai aur uss order ki details response ke roop me aa jati hai. ye poora data razorpay ke server pr jata hai.

### 10. Create POST Route for '/api/payment/verify' and Verify Payment Signature

In your index.js file, create a POST route to verify the payment signature:

```

router.post('/api/payment/verify', async (req, res) => {
const { razorpayOrderId, razorpayPaymentId, signature } = req.body;
const secret = process.env.RAZORPAY_KEY_SECRET

try {
const { validatePaymentVerification } = require('../node_modules/razorpay/dist/utils/razorpay-utils.js')

    const result = validatePaymentVerification({ "order_id": razorpayOrderId, "payment_id": razorpayPaymentId }, signature, secret);
    if (result) {
      const payment = await Payment.findOne({ orderId: razorpayOrderId });
      payment.paymentId = razorpayPaymentId;
      payment.signature = signature;
      payment.status = 'completed';
      await payment.save();
      res.json({ status: 'success' });
    } else {
      res.status(400).send('Invalid signature');
    }

} catch (error) {
console.log(error);
res.status(500).send('Error verifying payment');
}
});

```

```

```
