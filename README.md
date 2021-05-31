# Example Payment Service BC Client
[Live demo page https://sandbox-pay.101digital.io/invoices?sharingKey=...](https://sandbox-pay.101digital.io/invoices?sharingKey=eyJhbGciOiJIUzI1NiJ9.eyJyZXNvdXJjZUlkIjoiNzA3ZmNiZTEtMmY3YS00NzhhLTk0NTQtOGEwMDY4ZDZjNDE3IiwiaXNzIjoiMTAxRCIsImV4cCI6MTYxNTEwMjE2MSwidXNlcklkIjoiIiwib3JnSWQiOiIifQ.w_J95T2_ROoMLDrtv2wd6syJcsTDAkajjFupd106T_E)  


[Swagger documents](https://payment-service-bc.apicafe.io/apis/payments/payment-service-bc)  
 *Credentials required to access, Please request if you dont have any.*
# Table of Contents
1. [Getting Started](#getting-start)  
   1.1. [Get code](#clone)  
   1.2. [Setup local environment](#setup)  
   1.3. [Testing](#test)  
2. [Pseudo Code ](#code)  
   2.1. [Get avaialbe payment methods](#paymentMethods)  
   2.2. [Make payment](#makePayment)  
   2.3. [Submit additional details](#submitAdditionalDetails)  
3. [Examples](#examples)  
   3.1. [Available payment method response](#paymentMethodsResponse)  
   3.1. [Simple Card payment](#card)  
   3.2. [3D Secure Card payment](#card3d)  
   3.3. [Poli payment ](#poli)  
   3.4. [Paypal payment](#paypal)  


# Getting start <a id="getting-start"></a>

1. Clone code <a id="clone"></a>
```
git clone git@github.com:101digital/example-payment-service-bc-app.git /tmp/web-client
```
2. Setup local environment <a id="setup"></a>
```
cd tmp/web-client
yarn install
```
3. Start local application
```
yarn start
```
4. Open http://localhost:4000 And start testing using Test Credicard from https://docs.adyen.com/development-resources/test-cards/test-card-numbers <a id="test"></a>

5. Test with [live demo page](https://sandbox-pay.101digital.io/invoices?sharingKey=eyJhbGciOiJIUzI1NiJ9.eyJyZXNvdXJjZUlkIjoiNzA3ZmNiZTEtMmY3YS00NzhhLTk0NTQtOGEwMDY4ZDZjNDE3IiwiaXNzIjoiMTAxRCIsImV4cCI6MTYxNTEwMjE2MSwidXNlcklkIjoiIiwib3JnSWQiOiIifQ.w_J95T2_ROoMLDrtv2wd6syJcsTDAkajjFupd106T_E)  


# Pseudo Code <a id="code"></a>
See also: [`src/index.js`](https://github.com/101digital/example-payment-service-bc-app/blob/master/src/pages/index.js)
**baseUrl:** `https://sandbox.101digital.io/payment-service-bc/1.0.0`

1. WebDropIn config
https://docs.adyen.com/checkout/drop-in-web?tab=codeBlockxh6WB_7
```javascript
const configuration = {
    paymentMethodsResponse: {}, // The `/paymentMethods` response from the server.
    clientKey: process.env.REACT_APP_ADYEN_API_KEY , // Web Drop-in versions before 3.10.1 use originKey instead of clientKey.
    locale: "en-US",
    environment: "test",
    onSubmit: (state, dropin) => {
        console.log(state)
        // Your function calling your server to make the `/payments` request
        makePayment(state)
          .then(response => {
            if (response.paymentId) {
                localStorage.setItem('paymentId', response.paymentId) //Store paymentId in localstore for next API call
            }
            console.log(response)

            if (response.action) {
              // Drop-in handles the action object from the /payments response
                dropin.handleAction(response.action);
            } else {
              dropin.setStatus('success', { message: 'Payment successful!' });

            }
          })
          .catch(error => {
            throw Error(error);
          });
      },
    onAdditionalDetails: (state, dropin) => {
      if (state) {
        state.data.paymentId = localStorage.getItem('paymentId') || ''

        makeDetailsCall(state.data)
        .then(response => {
          console.log(response)
          if (response.action) {
            // Drop-in handles the action object from the /payments response
            dropin.handleAction(response.action);
          } else {
            // Your function to show the final result to the shopper
            dropin.setStatus('success', { message: 'Payment status: ' + response.status });
          }
        })
        .catch(error => {
          throw Error(error);
        });
      }
    },
    onError(error) {
      console.error(error)
    }
   };
```
**Notes:**
`configuration.paymentMethodsConfiguration` should be taken from `GET /paymentMethods`
```javascript
    async componentDidMount() {
       let paymentMethodsResponse = await getPaymmentMethod()
       configuration.paymentMethodsResponse = paymentMethodsResponse
       configuration.paymentMethodsConfiguration = paymentMethodsResponse.paymentMethodsConfiguration

        const checkout = new AdyenCheckout(configuration)
        const dropin = checkout.create('dropin').mount('#dropin-container')
    }
```

For PAYPAL, `paymentId` Should be stored in browser/client for next API(`/payments/details`) call
```javascript
 localStorage.setItem('paymentId', response.paymentId)
```

For POLI and CARD_PAYMENT: `paymentId` will append to redirectUrl `http://localhost:4000/checkout.html?paymentId=<paymentId>`. Client can obtain the `paymentId` from this query paramter to call next API(`/payments/details`) 

2. Get available payment method <a id="paymentMethods"></a>
https://docs.adyen.com/api-explorer/#/CheckoutService/v66/post/paymentMethods
`GET https://sandbox.101digital.io/payment-service-bc/1.0.0/paymentMethods`

```javascript
const getPaymmentMethod = async ()=> {
  let resp = await axios.get(`${process.env.REACT_APP_BASE_URL}/paymentMethods?documentId=${process.env.REACT_APP_DOCUMENT_ID}&documentType=${process.env.REACT_APP_DOCUMENT_TYPE}`)
  return resp.data
}
```

3. Make Payment<a id="makePayment"></a>
https://docs.adyen.com/api-explorer/#/CheckoutService/v66/post/payments
`POST https://sandbox.101digital.io/payment-service-bc/1.0.0/payments`

```javascript
const makePayment = async (state) => {

  let resp = await axios.post(`${process.env.REACT_APP_BASE_URL}/payments`, {
    documentId: process.env.REACT_APP_DOCUMENT_ID,
    documentType: process.env.REACT_APP_DOCUMENT_TYPE,
    paymentMethod: state.data.paymentMethod,
    browserInfo: state.data.browserInfo,
    origin: window.location.origin,
    returnUrl: "http://localhost:4000/checkout.html",
    redirectFromIssuerMethod: 'GET'
  })
  }
```

4. Submit additional payment detail<a id="submitAdditionalDettails"></a>
https://docs.adyen.com/api-explorer/#/CheckoutService/v66/post/payments/details
`POST https://sandbox.101digital.io/payment-service-bc/1.0.0/payments/details`

```javascript
const makeDetailsCall = async (data) => {
  let resp = await axios.post(`${process.env.REACT_APP_BASE_URL}/payments/details`, data)
  return resp.data
}
```

# Examples <a id="examples"></a>
1. Get Payment Method `GET /paymentMethods` <a id ="paymentMethodsResponse"></a>
**Response**
```json
{
  "paymentMethods": [
    {
      "brands": [
        "visa",
        "mc",
        "amex",
        "jcb"
      ],
      "name": "Credit Card",
      "details": [
        {
          "type": "cardToken",
          "key": "encryptedCardNumber"
        },
        {
          "type": "cardToken",
          "key": "encryptedSecurityCode"
        },
        {
          "type": "cardToken",
          "key": "encryptedExpiryMonth"
        },
        {
          "type": "cardToken",
          "key": "encryptedExpiryYear"
        },
        {
          "optional": true,
          "type": "text",
          "key": "holderName"
        }
      ],
      "type": "scheme"
    },
    {
      "name": "PayPal",
      "type": "paypal"
    },
    {
      "name": "POLi",
      "type": "poli"
    },
    {
      "name": "UnionPay",
      "type": "unionpay"
    }
  ],
  "paymentMethodsConfiguration": {
    "paypal": {
      "environment": "test",
      "amount": {
        "currency": "AUD",
        "value": 200.0
      },
      "merchantId": "NC29SFD2TP7ZC",
      "countryCode": "AU"
    },
    "card": {
      "enableStoreDetails": true,
      "holderNameRequired": true,
      "hideCVC": false,
      "hasHolderName": true
    }
  }
}
```

2. **Simple Card**  <a id="card"></a>
---

   **Init payment `POST /payments`**
   **Request Body**:

   ```json
{
  "documentId": "dkjJHFJ87238476FDG",
  "documentType": "INVOICE",
  "paymentMethod": {
    "type": "scheme",
    "holderName": "test",
    "encryptedCardNumber": "adyenjs_0_1_25$nCmMJjZw6btyZ6eN+1q0hLAfvwyITOzhK9tu+5KilEUpCCEvJ/QoedDBTyJcwS6AWnGHznMht4Rj2G5Wh7cnZhtwY07dNnjOG9oXJ1BodKePq53BAMeUQny1AbzmIXhaHvKJtYVvJMs4WH9DrTpEiSObXIJ7TMPMJec5oJt3298SN4dK/0faSEaZ/2HWaI2VrrAyyn/2hFwg/c5gyDF7unAHmnjFz/uG+q716qirP3SpM9BrX7CNneh6CdTGBZRfPpFmDZluybKyhHISpbCeSO4gret3lGwLDwxRYjEh7HsL4CBcFPrY4mZZBb/oH+odRN7pd77C7cP/Iun85Tw4hA==$jOOZvrrnhqECQr9kAc3sek5WCoh6aeJr1psqYCkXV33Jmmdw+FFTH+wwmbdxFe+XnPyeuCA6Folcf/ng+Bs7bDQqtJ/a1kfy8tSsJdOSwGlSFwwqAsThRqmSQCgjbZSrz9YIVUA3VUkO99MseDAh6PZWVJyZn9bGtS5HJBN99JuWLA37XqHt64YY5egHOYe+Lc2eIyLGdRbyss+Z9EhwsvP9XoKLciP6Rqa917RhbOKWEBMeKkbbuA5fSaqse2CfDSa2m5mE7Gvgvm6ELglR9+41C92E2jMn7PqPodlFImVS2QcN4NG7JVYPmaQxp8rD7YEn2hMepvDIlUSbPDit5/eEwHJggL100oMiS3Mxy718077zVQzkWSZhVUxEp1wyeUPA0XiZYjnRvtaffvWuCBoCSnRArsvbW1p1qihNYBy8Lws8m/IIbDuR0VyfNn6uxvj+5FP52NxgHXtUm6NzImXebJhs2XNIJzB2TNwpdBL87WIFnjDTTmys5RYg3Hek7fU5DsgWqZc4cNlmcm+AMHCOTHP7rTtZzlPG62ZQ01vQVHkpTYmPZyID5qQvVvCiK+zZlx9gdEMKrjF4HCsehhTGM1Xyr2osUNaE+lHvHOHg9jF6wAyTgqYC4wuoR2fJHgrOBvpU51hh+Dz8dy11Hgje1jJv09+5kWYX9PGGgL9SJbcJ6a9qGNqM2TX3VloR05z5sgdJ3Y8Z+okav6+fztawdSz/",
    "encryptedExpiryMonth": "adyenjs_0_1_25$Y/yRQDHDYA/Opnpw7Cmixufe9/1dBGTvWLdiZte2GA56mvmlmdNwPyzEit3w+ne0Fj5kpkhVFsiNGJIvdZguTxspHhwWotO2p//vK+aq1e9MVr+MXw4VuGRdRcpr9BNA84FztTE19ZoK+f41gGiapfcNH9Sx0tcUD4q7LJPJIB60xYKBd/O2bBfvpuR/FtXuiKNG5/TwFEXSZfwyxJXKldi5RG/HPgFWam6RXEUrWWNImeSZjs52QQ8bEU/U0QvSTs7L89GY+ZPMg2sE+vGSoVWvN3i9eUEL9DTFYMzQo/dFTglfkYY+UKfOc6EbHG2HxQi3cGbaR+og/s6CVPw+nw==$Qwg2NUzI8lPs68yiJFbOqjpysAhMnsDPJ2c2zSu2XwronOMpx9MX6oSuntzac3uvzod9dB4/u3KaeKhjT6vtOfMNvNpUq6nRcoLX+ZFeT65yOD+jafH2XCXwsJJMhF5BwnvVNPsM+2MQ8yOV7rsiy+ZOYp7Uc0/bl4RQrUciMT9XT0iNFTF7YkO/ucCZ6bopxMkYxxtdEyObu3N84HxeTnw9mcIFj5GRN7uiRxJrsmpqtxw9nWTBhGPTZGeAg1aip6H1k8caJO7TDqQ/EwPgLXxGVLAdxJDTAGQxuU5Q/1oaX68WYNIUDp4PwzkiHGFjwx9BVuUgZ/F2Ote4hIjSvmcNh9ECm5CldaexfeG9HP8onRW6GtoB8CBgyJ3AkAh+SOCPPODjD3GB/tqiIoX1r1I0kaIhzZyvLxkLmuvZxUt+ZBI=",
    "encryptedExpiryYear": "adyenjs_0_1_25$n9sEOc3t6bB1Emga05NI+LwP/tvjxh2Q5ecL0/zxrImoK9NFdHQOZKT3T6AdMgCBBilsOVEiMR7FvXKCgpJUqn+SIhlDrjwOkNsE++3zJjB6is72xaTsYbgXZs+X+D275OLt+CMzadzprP0LOQShzjHNdA+y0G3QdonH9AsevQB+a6iJQJQWzSU9Cdmiu9/UxSso98zyVBq2TJykumI1gZN7TxuFUvplS8BWwEoGt+zF7vAym9wNnxYU2GabTMEQCEIvmOKMWWb8pj9Vmp4rg11e9RBLpRNauGaQhKtAJy7FUhddIrzMD5Bk63tGUMdaMiFxXCRXMIL28yfL7uphhA==$kSvbDsBaPIKIgEsfwZKdRBvS57jmvFA7sykJl6X2BuwEvE8/RPsk66pjeIIQEFCX6FqTgPR+KQHa3Oo10713jCbr9l8wG59tr5QEpVFJG4sQaQ1FUgkdaPreHEpaNrmdjaigVbVRYTdIlUOFnzMf1Gp3sxHJzwZaTKis4mF7Q3uuMTKLaGqwmNcbg/Gdz9AGDCVkURI4qIUDvT9I0/e/1lDB8F+MUmys9nI6Lvc3dTkX5Y4IlVd3gRWbb3NrgUhGQ7Ql+dfAavQ1feyEK9Tp7Hbqz60zRf3rRVqJX2pK7u9TwUPkqHzWbDEMFvDFKJ2QTL1ceqiVnldmRrU2JO/AOl2z1oVpejJmMoi/tIemEsyBYVl0ojTrAPmeR8ZsXn/ydVNAdqE9lktRAaRxk6BaRyaXQuiiVWTPOTuTR0d54V8VtQ3v",
    "encryptedSecurityCode": "adyenjs_0_1_25$SdaJArrf/BQfMVIwbUecbwJ2H6LKYu3unKd7SpB7TkpXeSPPQupCFX63e7n67BEocOAQAEiZ2RPJh1zA09eClQrA1JhZtducLHQZOCtgUhSqRMmauDP6TwYWHQllVsK/EUf/XiYUNvlz2vtRuQ9yoimfyKb9DvlIOKPndjkHAYgSJhyOHRfYP7bvCYACFmPv72CToDC9Bgt6lyL0W9OIWNN/JBbPWJIwJubrMD6x300MSupseB8m9/uchA3c7gDGOJ3+3mM5iNzbD7bNs4E6lzezU8eSy3J+0sZrJxfTwczpbDMn6o7ckLERaOsjsvgPnJDZOIZrLH4kn0UjbExAKQ==$MTkTYp6Lf4oKAzNlW7IV56B3i0vMW5rQ+DpqstPfBNAnrnSLwtN457ETM9ghqWkvSYUJrn+vYIiOeyzxqiHL8ZvXtXUM+wwkXR0HuW+bImT7k3mZ/S1hGlAEpzQYupt8AoxgyZrYuSrxeOwbPeNTPBwrSZ9K3eAd3W3VvxGDnpcsnprOmnORR9p1SB8k8PzxvSSxBuQiTyd2l3CClxqiSRpE6avk165JGurN41y9m3U6liRfMW8fJJyRtDlgHBmYMQ/a+P+HgHkTDu6TUqIFaKOa7TkPT+cn4I+cBBZsSnVFQmEJmRkTeWFYFv92azAHtpChF4zQsJGD2Ao3d1UwuqsPD0Y2Gr+uEGlm5QdAESV1qXo2Mxsxf+PzejC1YaVLUvGD+3tu3G2Z+VeoqKUQphl2iC3q+dv42vC/DQ==",
    "brand": "visa"
  },
  "browserInfo": {
    "acceptHeader": "*/*",
    "colorDepth": 24,
    "language": "en-US",
    "javaEnabled": false,
    "screenHeight": 1024,
    "screenWidth": 1280,
    "userAgent": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/88.0.4324.50 Safari/537.36",
    "timeZoneOffset": -480
  },
  "origin": "http://localhost:4000",
  "returnUrl": "http://localhost:4000/checkout.html",
  "redirectFromIssuerMethod": "GET"
}
```
**Response Body**
```json
{
  "paymentId": "6034cd86-2ac9-45d8-8905-35b38035ef98",
  "pspReference": "882609828909879A",
  "status": "AcceptedSettlementInProcess"
}
```

2. **Card with 3D secure** <a id="card3d"></a>

  Init payment `POST /payments`
      **Request Body**:
```json
{
        "documentId": "dkjJHFJ87238476FDG",
        "documentType": "INVOICE",
        "paymentMethod": {
          "type": "scheme",
          "holderName": "test",
          "encryptedCardNumber": "adyenjs_0_1_25$f7mxoGs427JmogG5lH2FyxQRNZMmXrMrt5GpttPAxSoFvkTlHYUMT5CPA9fgZlB+w0hxtZ5/IJLqXurcXkS7Qk7N6n4FHthC5omUwDVA7wdNEY0QtzdjIn8jDsKzGbMpj/vkKl6IRIdXg/K6JH+ESm24nggg5VSSmBy7YXFZTPO3L2rVKkmb5lOz6/JknHmvVeOxiEvWy4xLjw8M6QX+Opnec+3w4OzvpubE9epMNtbzUFRRbFXdcsuqWlDU1E4dFfw0MNgJ4s7dAAB9IxXsYUliVJlcJjD8qX8ZHeDxFNi6ROo3CHl1AV2YI++KgqtyoO8ADcUALggI7OXZiqiPQg==$3Duwh4oF6DG23hbeGY4QxNkA79OH/a7xnhnfSTL+/e9t0GmQUAEs2MngNg53L4fu6BkZHytZcdzDYisF4UIiSGeFvQGZXhG4YKhODjC/SWCBfmGHHg1sKeweKl9aJmw6GokcoI5EKKxKEN7f1bK0acVerTpIftIbwG3mlDiV9FhUOIA8Ic80XcLLwJC4ggNifVteoeqy0oxhHS0V2Ub6Vy63hKK/u6UyBkE9LWbW/RWF2z5ptHjIEy24/HJRr3CUjGbTiy/0uAS7GWTLIYjpkldO+OzfEj4ZbGA5pXqVqZTgIAG+cO04hZAqzjcLeeleIeKpn25sxiQwpiX1FH+jVVb9fbCTxHopQ0Eerc2u+myItkv3yIdY6/SWaZIMXC+fCsRzizqY3F+QPFTjSXVOBAjsn2L2vCJitNid05NFEE9v0E05CrokZDlv5fZA0OyyPGSw2Saa5FMbAFoF0fEaO8jgj/9smRlhvum2gA+NO38YYEULC7/5dOxfEgHuiypYNSdIjemSW9T+sSMU8JiHJN7KSSZq5P7/S5o6jxMx12Shj8n1+lKeWae75TpJADP9D8J21kO8Kf0+NP2HDeBEqvNK2DOlm/yqt70U2RWic8CUPlP5OKBiv4pU0oCie7p1nebtjyTjjg6IFgft7NAHiwsT4y8Cvy2H/0IY4WyiScjycg8eLN/fo64JlD6h0ELGHtbm4TjITqR8xpPOyg==",
          "encryptedExpiryMonth": "adyenjs_0_1_25$EoUv315Lfw613WHtTRukltAkx8rWdZQjNy6P4CqZ62uJMCsesklDHc9Ym+QlFxtY1CHH+ZaHZ8ooj7KkqOgbd1is10QL08ALz3AmHl5WvNNPyNOPn0X0Wv2KdFDgupn6FKrnLBWvqbQzG0cnCRachwMdsZLPhp5EduJFpMI+dCC+nSDiOE01j0y/dAiNOv1hPYBg1k0WU/Gfoa1AvqPk1JgLw3RFJBCm9UAq7foG56WfS99oKwC/DU+wrXyIp7AAfMcLGM8pmP7llh+eSkgxuAhb4gIXAIXm9+j/ttWemwht/taz6k9h5e5TRdz0SHfbSzNwZZcICcTgaX8M4hiuQQ==$XNQXwjIwfbD5YgIBLOZ+gr8ei61FRkdsLwo1WMRiSIhS1n3tkmDPw/mxAfHjKhBrzFrhF2+w/1u3rP7ltKlBVrWhmYRYyN5Nojxj70i3i/DyURhrYswhhqD5J1W1G9vAOV+tdLrqTuZIGLb/GfMdeb7or47vVQJPrRT9++Ol37GgW20NvaLIj1LXD9BKlv6H/b7riQUBQx/pv42uney8JpDx4VwI+lWxY4rtyBkRQutvYEkxFyKZQAVLWDrbIq2GuVZRWus4vYl0tTPPPSiF3LkoH8n8SQA4lP6Z0CXNKp/reoJj8rfymCv3eM0Je4du2MWPA7lpscDGk9JktS7qOFZirzIwaiKz6itXcpPfQaLlWVt/UbMNGpeoB6q57yGMdGLheaxwt4pbarDTBJVu1YK4LpduD1T690/jFsBm4IHiveQ=",
          "encryptedExpiryYear": "adyenjs_0_1_25$0TT/uzBrSkbdYfF/koRw8ehZLk1P9GtvHV4SmDWbuh5kdvTYRISpUr7sH+gPXxxFYSqP7UJhllhz2i9WMAZbfElK+9MppVPnYI74oZEtIe5vwnDpiTapI1NtGRMON965u5oRP3UsIptDbeztmHjzqDI8Scw4G4yowrlI1bZjTOcdr57UD9JMPWcKzsgArr1OwUwcYwNWKxmWpf93SVbzLUi84b8nALwSI6gGO0wYgrsNMnv6c1kOy5+RrMuBl6b+9RcgfcsAhNhYtjdQHwqr/Nt2XY/vLAYmVvHLN5tslVLDDkQI1ZXAqoeaCxsNaDWFC3zJYf4wHXaVpNyV7fJWOQ==$wPSYJaoa7IssYkhjQDPJkmBHbmiKRQw+EZdQsqjEn95D+hMxRW2zDq8bgebZzas/uu0/e4k+N5Y/9Ab//Kfn2JgW+qs7Ka5FzhMi3p4UeyviYIiguZxf8Lwjzu1NnpMUo6mGsYN76slPari5URPkuN1UemAWlPe9/5ifQFDsFvxiJsc44ALBZ9vD1R9cKjcPmiMtLkmmY/hYGqLx5G6+KtGsHJoLuo8x5zTUgJGbsoiuqcYB9yLFHRXROQTXVRTJamhHuSl5w5FLKDp1MeplEs1066mkyW+9Zbm19/ttp64PfUdz1SCkR5WchgRX3ikUrS402Y85ATFVsJtw7oLdA9Xmp4udrpB3TMth9JhWCSHzLyY/1iGfyMVDqgbAAkGSdPYY8iIvQZeJGiVRGxplKvr5Ob3UB9Qkh3TMyXwUgbuklq9O",
          "encryptedSecurityCode": "adyenjs_0_1_25$BgK3NVYOMt+Fk8Zr3+zp5Rol790J8K1S1QfhxZfsKS0N9s5JcrdGZmOAoS9R+VSLYwFnmwlawKoXeAZRSJUamaEHHZipyMVlY3zIvNCseIzRltUGYIoPXCSBthHufzq6i4llc5vt9jyd7CSkpf/tMNqKM8kSU/fq0AhPQTTaU6cZs6gH9TlvDZDbpLe0CYQbFv6nYbiHAdiO1cB6OwmdU0P6oZZIn/1Ki5b6K2AFL4purEt9x7JgY52qQ5VoMJ1DSF6guF0tNYRwCp7nUv3fpC6ldr7tJ4+Nvd8IBVa2Kw+B/PMIu1oLIbQFA6OlI/dx8jA2dHhW7VZJh13/clXoaw==$f/NvBpVgF4AS/Rq67RGzRg96OWAQXXbzHdlVGWOYiOmhHwODqLYFJbSYIPYejQmT6qoWRirDUFYLUcQGk1qV62KVvp0+SBvTdTZh7Owe3P+Ui12TZc9xoiX3RJgd0uh1I8TzFJkEzfITk1zDoYWE0jKycp/jbb0xVCIugHeXFFtis8l8LAG6fXUWFTaYyZw2e+TKBVCHW9Juw7O9m39Qf7GNJZNXdli2Vx5JAGTOBHxxIJ+U+YJL/JkoIgZyHAIi1gqkFK4tICIhDStiIpi7NouX0r1dqFtYB6SsKXyBgp6lHMxwx0S/Bf0Of6DZhqiUZTCWWoYtQzEQNQiXrLTo7KRAKCo4mcAb+QERmMCxlY+ifLS3Dg97XZiprioNesxhdVjyWf6m6k+NohPXAec7hPv05vfxvjCr9tDLbQ==",
          "brand": "visa"
        },
        "browserInfo": {
          "acceptHeader": "*/*",
          "colorDepth": 24,
          "language": "en-US",
          "javaEnabled": false,
          "screenHeight": 1024,
          "screenWidth": 1280,
          "userAgent": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/88.0.4324.50 Safari/537.36",
          "timeZoneOffset": -480
        },
        "origin": "http://localhost:4000",
        "returnUrl": "http://localhost:4000/checkout.html",
        "redirectFromIssuerMethod": "GET"
      }
```

**Response Body**:
```json
{
  "paymentId": "d2322d8c-2a29-41f2-bc4c-26e9cb4bd03e",
  "status": "Pending",
  "action": {
    "data": {
      "MD": "M2RzMi45M2M2NzQ3YzJhNzI2NWY2MGYzMGI3NTE1MDcyMTQ3MTcxMjE2NDQ3OWI2ODZlODc2ZTZiZjdiNGRjZjgwYzQ2LjIuH4sIAAAAAAAAABXOXWuDMBgF4P-S66nJm2CNMIatMuY-oRYceBNf42wXG9dFt1L635ddnnPg4VwIIylhlOX7j71T5q0qnqq82Lw-kxvChN8G56a0iZrIWFRmsN8uFZTSJsJB46edXTi40dxN6jzqo3vobjvgAF2CASiQgWA9BC0KH2MtsRVtR7n2OHh7mttwgTChLKZcckjihEspwsOqCtYx-xqX4vd-KHm-ret-R_Hn5YCm5I-2Lle2z-DdZP83qbcuBOfTSR_x7N1sl_t-UWbWJAX_ll6vf2Y653jqAAAA",
      "PaReq": "BQABAgB37-f-c7RVoRQr0i2dB-IQjfKUFcOjUjzGxkESjf-kQsyZSMbAEgud2mrZ05BSGbE5WXOTWVv3i4428c1Dzm_w8XIsefm9RTVfOAbZSgXQC98wZFCH2itOlPTYIYg0Y28esm_4v8vvsDlmoi6Sfycxr33kaD1h_ToLM02k9GDHHJrbSPTA0mt_9hHH46sKbuKlZAPmYKgYfIH26xNAEy-2DJ3w5qBJ2lcSfNXeI-1bUYClm7bYtg0YpgZBYBuZrYkyhv2gT0F7epw2deG09SZvVC1Lt2Pb3vcENuQcAcPLBDYd7wNomXtCkEDmiSR8rmkcgn8tmy4Q1VcRHUKOLpeCxY0uYQPlC8JmMuamhn5eI-aRKyTjumaEmGjg4zoNa-qxH8X3X4UnBv0ewpyBzB3qgcjZNUNKxZwCOwY4kwrKq11woScY6AmNsk-UyH8upEwRycJ5I9RUNrVhlMHnzG2yayFUgOT8HAFoyvxkALV9NuAuelwIPhKl1QE-ceJVCxKktkZqjK6sWML9w9FJYsr9HjkMRdSivqbX4VaCAThy6qyOvZG2Yn2zok7BHbQ3ej-N_n2yjNAjf-rciXPOuwWM_HU49mOIWZow8P594vzYdYQHeWqDsyUccjhn785cQuKB8WCP2WiXO4_bQRRazNvTpy1kp4iHWdP9u0CpqJhcExAjcx2KME08xbAfQYTjkQTcAEp7ImtleSI6IkFGMEFBQTEwM0NBNTM3RUFFRDg3QzI0REQ1MzkwOUI4MEE3OEE5MjNFMzgyM0Q2OERBQ0M5NEI5RkY4MzA1REMifUzIqYILxnpMnucbDMGmA1NpCSzGfj3ws_-ED5DcaiqdUKtavyxIl5_e4evnxRyUr5DmPsok_ZtTtWvJr7x9O-d0Qfn9fLz0vnfnm-mBaPmHT0Cta845ayUGefyihr9FZfKiSrJ3p7aAvbqE6Vb_vCxVUTNazAThSXUWhVxvA1EQdiqrEYSkp87gfO0PjLOKJdMZCaLxkf9yvl5s2d-DJB0oa_5Yp9a13EuUVO-h00UXSRa24EDbZW-KR6z7S9MAQOxkOZD5gNOqW3iNjJlgdx4wP0sexqTAmCgEHWc5DsjLBkzgUy9fdzgUcW1g53pfczGuleJ12oaW0etDk7INuetMK2BILHk0dCE0S1v44URIVZ20tBVZawsTlvAu3CLjD4g0OHnGULiCUPCv8M8z1da4Yq6Zu1e0889MORErI9NoyX_qgYS7g-bpTYgd7RouMi4FAAEBADN3sQjkS5OOIIQ5Qj0n5F2ycCMH7euJw_gv_r8HEq32jFNVsXm0Rwt0w8b_sSZxs_Mgihse8aKt-NlgPM5G00wxFUX0BTaChsxXtwkAv3hhl9u4Nn99wNYSY-aUAnyGsx7Vfw-3HprCw_-suJonKtrsRJYLnrY9kt5tRJTUXkfYPtP5khZNTzKiUjJkYFTrFnhPPkggMqwotPoesNO6NHpVzwLs2pw_Opxr0sZ0Q4J4FXPxtyYnUUbimNMm-UwE8xCZh-i7TYbgPyIqF6cDix3BUCzMsN3_-gibdE9dbBSOCMLXtkBDiJamj-om84mryQ0BfL4COJN79O2RRjiQe-4QF_29-LhO-e2WS2aBZ256_gAABujadSZCXHpHVbSeLjs6kQ4PLRWae4s8hY87bj-M3YYcncdNUyB9xG_QX61Y-75RqdnRs0XrIOIeHMIhwJHyBmuZDzU3O0fhrpaFkYPMGT_8jKJpOmSYm7OeIL_qAUYk0WzvkUH2RJMa7FJf_PtrPCu2Y7N_5eGonQT6lWKDMrH-7hlkzymBVQ_jfCGvTZOat75rHhBbblBua1O9eA_YilYS6cFNFyItg3qoS4WETmGkRfyYG5T33zPX17xsDjcmY1q36wmt6kdbbAdiOCcOxh_fpnhbwcwZhoCSg2vok6kR_QipDeNR1VHSfjp_tIB-CwyTvAKFJxTCtUMtvxInLh0pXZ9wTj-Y7U15QH-SdETmy2cDxBdicNFVl3_bETVx1rCrJcyeS2S27OgPCH9iCwO_qWewLkvUeRmAL6tvUzRToVNYBP6Vwaoyda8y4PnmH5Kk9dixIaa-YXOJoyvubo7rVcrhXJNpJn2HEQsrZHxVrjM3ZyTR4Tstcx-gTfse-rDPi2QRPrZ9izIFd-oY5j9RKF2ZpdPVasCU_zZWbh1UX5bN7DBOv-QU5DLn6qmeHsu2rRxXMYGzz2soj6aW34CQJV95xiTixT6aR2mD0BoRAPmHX77wS2l3wBzEbPJs15RfMH7zLPdcq_gCK_OEPkNhY_qOyd0o52zpif96A31NPVK1dSaVG3V3XX8JpvSZBPTX2JTQSbYU_xFUlcMb_ljzjD6Gj_QbD3cWhl_vpUQemAAVzW15AtnYCwEHjHKZfDCOqkVX7V4KXEfiXZy0GW6KRTfz9C2LFmHVH3mO2XKVfnHH64HpozvOPa25yOdpTx27ivCTJRHQESGAOmMQNMBHk8llvAtAo2czr-9mpY1nPOEzJUFgtHvbtClq_8WRxWTa0ZTjz_QdKJFLxf-1jmWw4t73_Z-xgNdYsjwwcX1M525lBfitfkTtyhNwyKZFJGS5EXZaM_7qZIphmelTpMlWrwS2KIJgiL4KJ-MljCYUiCknGIAdPQEA4nMVFVwp34xBgqRqDcW0tW0b4Hh8FRONhonQN9Jyc6-rfwaD0jEu_Q4nrcvV7Q_b2BRj9Ii0kGw4bs8DdKtOYuIMuAUK6W1CKjGKv__PAJ1OS5dvAdzzzQjWwHHQtbC8NmqKFl46iaiAZ4JIjxUPDSfpE3OTVIUtydrResvJqi2X2kKb7sK7ADvUFrbFHL-IMMRkN6ldDOaipxGr4-GSxQAOyu2_q3h77-HNJ3pvAotkDdtt5pyYOsZ5ClNpYcK4_yizWB8L4k7wqHdFsAThpu1aGCRRCriiV0an_kzyl-QENuNuyR9jJFIfo2dmcWIPjeYtAbZfbyECxuJ6TjhpEanDqECgyywNKbNF1lV1TcN_T5O87HpWfidQvY7RmfXGtTRGMfoZNpUsVRAR-RImx16fcqgdKGrbRCsR_NACcakRxMHVkzk0RhVz07NzKG9mURAotXSS38yKok78stDS3CjBw-yudqrFNIvDG80cAqkFlWa4te6VHRuLJByernx8AFpo_cBK0Jo11EVyYPJvCiMFZ08R485YVggUTHAxsIuwhE_lkQccoo9f9VmIzvRsqH2ynzGfFImQGh_1Hf_JTj-PN_L0SvlgZzcXI1ropp40KBLGqi7wdNLz_S-LrqqmnHE7hQ6hCFBAO6wakFf2JEcy7lgIHiePpWoZ2zNc7MA2dfpRRed8iTLCLa9Akmd0b8acS6s6IcQr4w5MQ7ri5yZbk-70QuvMuCbecrEvNPa6I3cSARKNOMovU7icIo4trpBUfmEDvGWCA5Z8Y_lbcynCJR4GXVhKli-7CgwSCoje_QI2qEVRatn2eY9RD31lZa3FNsxBbuQDL97VtnKszQtSHFgb",
      "TermUrl": "https://checkoutshopper-test.adyen.com/checkoutshopper/threeDS/return/aW50ZWdyYXRpb249YXBpJnBzcFJlZmVyZW5jZT04NTI2MDk4MzAzNTAzMjRBJnNpZz0ybnA1TUdTMDdfOTdpY3VfVnpOWHo4LW1NWlZsNHEtTTVFeFRkUnYxTnFN"
    },
    "method": "POST",
    "paymentData": "Ab02b4c0!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",
    "paymentMethodType": "scheme",
    "resendInterval": 0,
    "resendMaxAttempts": 0,
    "type": "redirect",
    "url": "https://checkoutshopper-test.adyen.com/checkoutshopper/threeDS2.shtml"
  }
}
```

 **Complete payment `POST /payments/details`**

 **Request Body**:
```json
{
  "paymentId": "d2322d8c-2a29-41f2-bc4c-26e9cb4bd03e",
  "details": {
    "MD": "M2RzMi45M2M2NzQ3YzJhNzI2NWY2MGYzMGI3NTE1MDcyMTQ3MTcxMjE2NDQ3OWI2ODZlODc2ZTZiZjdiNGRjZjgwYzQ2",
    "PaRes": "BQABAgBLxQjK5smxovrr25KkQi5SO5NzB5mHWRqMRamQa3K4l5ZdHHRCE80MYdla8CR66ev3__eOHkr_SEAKOXnuz8xzLvRbFFfDyclnOWtOt3dKNTJf_sBuYp1oA14H0D7ayinMT1Qcxf_YvCtVEAy1anuanD5i_72pwrBKv5R7_DGkrAfT6DaJa4yj5wiBh80NTplvRuwdbDq2JMCK8ukugKUCrhSCIPvzTSmUZWFgWEhL3jFpYr28JMIkf8oeXiL1zBr3eK1MzBZodpbXEVskynnEFn0SsLGqcwANrRUBSCyYAyoCCZgUsFmK0CfjavLamls3XniauI12DO-hIOLxE6GEITLeQzstiQ0H64eRpINVM-Is6DCkzCwfdN1TzRaI6egoGHrFQ2bZtSPqIUWtfXkS-vj6IeaeL5ng17x15N1EWHiRszOlGyfH_P76vXuR8DT3IFoQHRd8I8nsrQIPJRcAu9nEgFLfYMGWogFaonFWi9X4nFE5D_9Gq14-ptdUMTaNTDH1xgsHqE4ZdGGo0zVj7H3i5vlv8defnK9fDuiQDPFyxMYPE6lgH3WQvuNhdOLYB8ubM2HNbCxUraAJKjoX7wY2P-kHFC1BgdEOVON4agTaM1RrEH2fhIKncVCoO4lDViVR3tFd7hHN1MIX4V2amSJGyzs8WtfDnlWIyVNO5hCl1K7E_x3iYrx__RNwNby-AEp7ImtleSI6IkFGMEFBQTEwM0NBNTM3RUFFRDg3QzI0REQ1MzkwOUI4MEE3OEE5MjNFMzgyM0Q2OERBQ0M5NEI5RkY4MzA1REMifSeF1AkLO5v9g61aQNOPs6pkLqnGrTWxwhS1V9nOm-a0Ld1gPB11Q6ERQRlF0ezyz8B9ryEhlqG81uSMkY11lGPTDHuNneU0DIyNGDODmXmW2jI1O-sgkzUaw-ULOvtDPtWtGcL7vZGrZGa1sm_GGbhNLjIuBQABAQAzd7EI5EuTjiCEOUI9J-RdsnAjB-3ricP4L_6_BxKt9oxTVbF5tEcLdMPG_7EmcbPzIIobHvGirfjZYDzORtNMMRVF9AU2gobMV7cJAL94YZfbuDZ_fcDWEmPmlAJ8hrMe1X8Ptx6awsP_rLiaJyra7ESWC562PZLebUSU1F5H2D7T-ZIWTU8yolIyZGBU6xZ4Tz5IIDKsKLT6HrDTujR6Vc8C7NqcPzqca9LGdEOCeBVz8bcmJ1FG4pjTJvlMBPMQmYfou02G4D8iKhenA4sdwVAszLDd__oIm3RPXWwUjgjC17ZAQ4iWpo_qJvOJq8kNAXy-AjiTe_TtkUY4kHvuEBf9vfi4TvntlktmgWduev4AAAbo2nUmQlx6R1W0ni47OpEODy0VmnuLPIWPO24_jN2GHJ3HTVMgfcRv0F-tWPu-UanZ0bNF6yDiHhzCIcCR8gZrmQ81NztH4a6WhZGDzBk__IyiaTpkmJuzniC_6gFGJNFs75FB9kSTGuxSX_z7azwrtmOzf-XhqJ0E-pVigzKx_u4ZZM8pgVUP43whr02Tmre-ax4QW25QbmtTvXgP2IpWEunBTRciLYN6qEuFhE5hpEX8mBuU998z19e8bA43JmNat-sJrepHW2wHYjgnDsYf36Z4W8HMGYaAkoNr6JOpEf0IqQ3jUdVR0n46f7SAfgsMk7wChScUwrVDLb8SJy4dKV2fcE4_mO1NeUB_knRE5stnA8QXYnDRVZd_2xE1cdawqyXMnktktuzoDwh_YgsDv6lnsC5L1HkZgC-rb1M0U6FTWAT-lcGqMnWvMuD55h-SpPXYsSGmvmFziaMr7m6O61XK4VyTaSZ9hxELK2R8Va4zN2ck0eE7LXMfoE37Hvqwz4tkET62fYsyBXfqGOY_UShdmaXT1WrAlP82Vm4dVF-WzewwTr_kFOQy5-qpnh7Ltq0cVzGBs89rKI-mlt-AkCVfecYk4sU-mkdpg9AaEQD5h1--8Etpd8AcxGzybNeUXzB-8yz3XKv4AivzhD5DYWP6jsndKOds6Yn_egN9TT1StXUmlRt1d11_Cab0mQT019iU0Em2FP8RVJXDG_5Y84w-ho_0Gw93FoZf76VEHpgAFc1teQLZ2AsBB4xymXwwjqpFV-1eClxH4l2ctBluikU38_QtixZh1R95jtlylX5xx-uB6aM7zj2tucjnaU8du4rwkyUR0BEhgDpjEDTAR5PJZbwLQKNnM6_vZqWNZzzhMyVBYLR727Qpav_FkcVk2tGU48_0HSiRS8X_tY5lsOLe9_2fsYDXWLI8MHF9TOduZQX4rX5E7coTcMimRSRkuRF2WjP-6mSKYZnpU6TJVq8EtiiCYIi-CifjJYwmFIgpJxiAHT0BAOJzFRVcKd-MQYKkag3FtLVtG-B4fBUTjYaJ0DfScnOvq38Gg9IxLv0OJ63L1e0P29gUY_SItJBsOG7PA3SrTmLiDLgFCultQioxir__zwCdTkuXbwHc880I1sBx0LWwvDZqihZeOomogGeCSI8VDw0n6RNzk1SFLcna0XrLyaotl9pCm-7CuwA71Ba2xRy_iDDEZDepXQzmoqcRq-PhksUADsrtv6t4e-_hzSd6bwKLZA3bbeacmDrGeQpTaWHCuP8os1gfC-JO8Kh3RbAE4abtWhgkUQq4oldGp_5M8pfkBDbjbskfYyRSH6NnZnFiD43mLQG2X28hAsbiek44aRGpw6hAoMssDSmzRdZVdU3Df0-TvOx6Vn4nUL2O0Zn1xrU0RjH6GTaVLFUQEfkSJsden3KoHShq20QrEfzQAnGpEcTB1ZM5NEYVc9OzcyhvZlEQKLV0kt_MiqJO_LLQ0twowcPsrnaqxTSLwxvNHAKpBZVmuLXulR0biyQcnq58fABaaP3AStCaNdRFcmDybwojBWdPEePOWFYIFExwMbCLsIRP5ZEHHKKPX_VZiM70bKh9sp8xnxSJkBof9R3_yU4_jzfy9Er5YGc3FyNa6KaeNCgSxqou8HTS8_0vi66qppxxO4UOoQhQQDusGpBX9iRHMu5YCB4nj6VqGdszXOzANnX6UUXnfIkywi2vQJJndG_GnEurOiHEK-MOTEO64ucmW5Pu9ELrzLgm3nKxLzT2uiN3EgESjTjKL1O4nCKOLa6QVH5hA7xlggOWfGP5W3MpwiUeBl1YSpYvuwoMEgqI3v0CNqhFUWrZ9nmPUQ99ZWWtxTbMQW7kAy_e1bZyrM0LUhxYGw"
  }
}
```

**Response Body**:
```json
{
  "paymentId": "d2322d8c-2a29-41f2-bc4c-26e9cb4bd03e",
  "pspReference": "851609831914407C",
  "status": "Accepted"
}
```

3. **POLI** <a id="poli"></a>
 **Init payment `POST /payments`**
    **Request Body**:
    ```json
   {"documentId":"dkjJHFJ87238476FDG","documentType":"INVOICE","paymentMethod":{"type":"poli"},"origin":"http://localhost:4000","returnUrl":"http://localhost:4000/checkout.html","redirectFromIssuerMethod":"GET"}
    ```
    **Response Body**
    ```json
    {"paymentId":"ee057d2a-a93e-4a97-a2c5-0fcee6e18722","status":"Pending","action":{"method":"GET","paymentData":"Ab02b4c0!BQABAgAf2P2i67XqJ/gDJ8dXJP1FDa3vWV1WMcN/89j9/bnKWy8Qh2PxHXhQyeUE4dP56DSvAXoYUQMRCqF9RGGD8GiStkZzqdghBtjngLHIwjLbokD72uollHXwuSt+HTHTo9y8IjqdAs/rJtt6rYCSt+TDneLtwBDd+HT0SBooTop6NOxOzCl0ov/FtPzS/wRammEPUzTw92cdGkIwSdjwSOrGl2MhcQdcpJ81VZ6E3JILkUouVfAHZQrPGpbRIf2JxZMiywlgo2ojIMO2/VeUJNapc7jY5UmgJNDqJnzZKr9wd48hvOeugTSItTPg6D2K2ffFfbdWGToIA+pjegLMgk6Ye2tw+VNHpHGqvKO0NnfFAk0VtvavZ04NddCdOG/DR0tbfumsoCybtTzl2jifmoYBYNgCPZCQNPxoqOwD308nCQETu5PYVe6qKaFqJe/quanyQkkwg01vu7VwaYKB1YQVba5CpRC7u13WtiglftTLfzlO6Pw9mav1T6OpOLIGRXB7ScbSjpXdD4gK9SYKvv26y7HiOkeSVvQVOEOQCdVHQPso6W1f+ueruZax3INWgsCM5JAave65zmY5A9ZwezaK2AEBdxcqhQDh1dWHLVWRDNkhdvPUYyxFWz8Vv50VXIruwdnJ8nOZaJHDGhlMdedwat8wXHJPxIE6sQixeEheSBCHLhURaSHgHCzk/bNNxKqJAEp7ImtleSI6IkFGMEFBQTEwM0NBNTM3RUFFRDg3QzI0REQ1MzkwOUI4MEE3OEE5MjNFMzgyM0Q2OERBQ0M5NEI5RkY4MzA1REMifdeKci6AfZlKO2O+IUDdyMPjBQqDLB64fabCJF+4VgY2qi4Zp/e0JIYB1qTB2PPiNyHidN9bXSIH7I1xHKGO06Zn77KwKz27A1bP9jAY6ppz6KQcjpu7wbGjLrnRgFyfr2G7P2vkgmbrhd1D1LmfckhNcstHcS2Cb1JimC4TX7vrfniuGVzKCdwuc5CWPszyV6HlGHW3c4jlhiBNV97CxJ1n3UF2iTVazu15s7pPsQ3P2COT1hXubmBve7/53x5oH9LoxTiOGEPayOpeQ8t9PeA1U3HVVXpyWUcSI8jD/hs6+tmF4txxAV5NDtfPX4FAJFMQE0rjVasmUcb2FlIQFhDwGkb/NlJt5Tb0ZLQH0d4d3hzuL8k3qts9T80wQ9Li2TvZ9ozUvYpPkH7MSe05QACTDr5gkrV5QU2tC3FrX866A92LTkaFOEbtCoRe68cYZzqQx5jx+fIOaXDFaJn5fu1vzEhAZtaB7IzSECiWy0+Zavvox+dRPI9ebsE2X3s7RMKd7xrwVOvU81omsV4BvdmtMfvE3B71NWwTDmPxK8ZBNc51kQXp+SazFx5bLVW8H8UXG3Jhp1WGWfGGPPOfbtY8tFdo1VD/4fgSh8ou4XKqiFuYYPajzISX4xKiROQdV0S61gn12t+khcy0gzzf5FRTvRctH5bh7MI2iEJYNgc6fAn4dbOWLvFx8NBqqYE/ZHN9GRjq5id68Ua49Zj1sS/KAlTUGn88u25t3RN1Kmu1T4XfZ2IhYRqLme14XU8205u5FuCpDTBxCR9GQdLYh0mMed9bc4mxk+QtGhiu8+kGDhmSjzXuYrLchXHU4kROswFL/O8ohYAJ5/VAtMijQOvBVsS8/Kv05ThhTEeKV1sqhGXMvlFi+dKA5f+yTXRafjJrep0yXniDmTnmNzbTPFXtK0g7IBSLNCHLGSFpquwuYo4+KaLAG59QXu/25wBETWnVsYvEOLfROZMN+rEiF0+I0PMxr2gth6hbW6kE+lvGzmx5T+1y5GICWke/Hx+RrhmAi9M5DvvM3fcHcEP/t319oPrDh1RENPOi/r+9mf87l4+c+fPD8w==","paymentMethodType":"poli","resendInterval":0,"resendMaxAttempts":0,"type":"redirect","url":"https://test.adyen.com/hpp/checkout.shtml?u=skipDetails&p=eJxtUttymzAQ-Rp4gxHC3B54IAanvuRix06dvnQUsRjZIGEhO*HvK7CbdlrPaHQ5e2DPnl3SNBWjRDHBp7wQdsXeJZHdI6khJnkH3NqTM7FIw6xryCS3P3kF2Woodhwb2cjUGM-HIoe4ERUz6UlK4LQbkGSTmjVIWhKuEkrFiavYQU7KdkyR6nmdLdZpNn56*CJNuYKdHFLaqmsgHn-LxvOnzfrnffaYrabjm8zzVZHvfYVXUECvA*IIIeR6QRj9FVMnyVOiSBx6jo*i0HVC7IWj6I-aF7aL30pZb5ZFaeA7uhaHjed-6msia-Jdn-y4SuazYhmQ5bEbl4uuSQw3NRvS1aCrrYdisU6OTAntZrWIS6Wa1nATA0-0oiXQgzipthRNA9JS0Cp76IRNRf0-QSMtyDOj0Orr8yXNlGsu47sV5EwCVTpydvRWCar9vXAeQJUiN9zJP53Qam-1wsC*HAzSkjWlF21gr1ft9bq9689L0aoBH*kKB-i3XrtUddWHJlcvpnn-SgGQF*RYz1jkgjUiUWARTD0LFRTABycMMDZbaPtWvpKK5Ux12kDsWEgvb41C7Z2L9Ia9H*bVlX4MJKH9GMSZ9qKvEcz2wPhlJE-v9hnbIdJtdiMXh37oRtHI3gdr6853jvU5*7wvZ276st0WG0Q-Hve0mrlzsZ0FokjwW5X8AvbVFmM"}}
    ```

 **Complete payment `POST /payments/details`**

   **Request Body**:
```json
{
  "paymentId": "ee057d2a-a93e-4a97-a2c5-0fcee6e18722",
  "details": {
    "payload": "Ab02b4c0!BQABAgBiai8eB5I++Wh42FyUGnE25dOKJCkmHJdL+FQFNyU9YVR+9249pYig+OEqx139hBJEpB+sOtZY2pJNWuWxPQxZsm5d9pKT1eO9Prcv9ymZ/o3/xZ1UeCEHgMlkO4qHWuKjo2w/2Z1sLkNNd6r+z+mVEiEAMFc4jR83ggzukIS1j6FhVB7VmX+JdyVtPkHB9MR/R/3NJDjY4Es81dZP1ERq5wW7QiK4/4PeRblcznL2pVGxtwB7lQpuvd56O1jwXy/4Vk6d29qrEZ/Mi3igpv9XLSXorapFpYYP5tSQVQAXsTq9hR5RmiEM1l0vM30pHih19Bu6Rh8Or4Vc89HrAzBMwXxgkeDnY2ey8VDMgNkuxwUQDwDJCfEzvCNCLvL3yeUJ323hD7ELKMUC/vzMBPHAyi4lf6HgCekKDm8iqL5/zzCZG4iZwoc1KnYtfoVJjQYxOq2qRCYyaiiIROyYzmHpxpU6VYUyo4H2uALwATcIzjlC3HFd9mDtqMn4ULQVujzrBA1C/BvHYdaIpoSYgBPu2OvS5eOk7/7UOj3T/JzXDNxjRDi5QFNIMfmtLzP5V5mQlABcSUiLR7PSYeltoYz9RZlE++HzrAJ/uH0y8H+5uQZHgkLMsnCtFiU9W0K4LjPzB/pVafWlEm2yCmRn9OblFVaT09NggpcyVMNJ1hiTyhAQct4THnmip14a75vMirytAEp7ImtleSI6IkFGMEFBQTEwM0NBNTM3RUFFRDg3QzI0REQ1MzkwOUI4MEE3OEE5MjNFMzgyM0Q2OERBQ0M5NEI5RkY4MzA1REMifRV7RWS4QPjj3AguXZihrc8NSj1grg5DdpQgBFqowPnU9us9qf/e1YZJapvkWkL85xmPktHRRW3N656d3GN9DhI4sJqtv+nn8WFIkC9Dp1fZtjyNbaSazTcKFdxBLYxLDsim4rAPKX0HlkAMLw3nX5jpCdEnnXWbM43VhZWuywB5i4zQ9F78iyvCrXMmyVn62je4/2qN5g4FNVEPCaMotqi6A2fQ9bAgjXUMsW0aablDsI2rC/CcU+VoFHxy27zQO9AspApuSC3sTLEZbRDKADxwoF9EbZVBw0Q=",
    "type": "complete"
  }
}
```
**Response Body**:
```json
{
  "paymentId": "ee057d2a-a93e-4a97-a2c5-0fcee6e18722",
  "pspReference": "851609831914407C",
  "status": "Accepted"
}
```


4. **PAYPAL** <a id="paypal"></a>
 **Init payment `POST /payments`**
    **Request Body**:
    ```json
    {
      "documentId": "dkjJHFJ87238476FDG",
      "documentType": "INVOICE",
      "paymentMethod": {
        "type": "paypal",
        "subtype": "sdk"
      },
      "origin": "http://localhost:4000",
      "returnUrl": "http://localhost:4000/checkout.html",
      "redirectFromIssuerMethod": "GET"
    }
    ```
    **Response Body**
   ```json
   {
     "paymentId": "50f9f5fe-2930-49ba-ba81-04572d9fea43",
     "status": "Pending",
     "action": {
       "paymentData": "Ab02b4c0!BQABAgAzBYGIu/mZsZgHzZrtSCvB7RtNbChgr7fduPqpGtQJJzswHbj8nKiNmWfMLXyTS9KtWElwzSfPbmPwmCZ/PiJKVzGNJLD08279PfIrChFge5MwYnBbjtcjRWI6Pllf/vTlv2s9wZzgRtP8XoRWdULD4HKc12LqO3Y/2hq8IKSyAeq66nzHaR3OZt6Hmo+FIDRDn+tkfpvipamU/yqUXog8aNuxrCwAdcTNF9ctFeFzSQza3dmQh84WrGqiqRurjVu6rCnjEj/5qxQXDjrqVm0IHnvSerjTlGXtXri7wG/jpTYLjkhK2DsI5gs/TFc2iqk9zkszKQUWDVIVQKL4vcmp41AI21HJaeLC+dl1WoZHkLg/JbUgsAgoltqdU1fATkABw/M+CtlJmHAB2pUS73g6Q8Fhospv4G2ku+QRlqOiZk2MixPckW6LmKS8yuuQCniw+jhdbmQgSAAPl6iqQNZHiwHnd6ddAucou8nMflIRv9S5kPNA7I/jcQNrV20qu+BIvw0WAS2zTdJNl2uZu1jX89vfe8vcxid4ycu6Aew6OzOxcr+CZpGuUCVKGFVuFyXXY1pL4YYPBDGEZE91FJHCKNTbeF6qrM9Qw4wKzi6SOyqAJGZC8iIQM4kRB79VQiVBGEAAK6Khy0RgXzwl3hSX5zlc5GU5t57m+vfULufrnBALyV5mV/hjXLaRumAufDnwAEp7ImtleSI6IkFGMEFBQTEwM0NBNTM3RUFFRDg3QzI0REQ1MzkwOUI4MEE3OEE5MjNFMzgyM0Q2OERBQ0M5NEI5RkY4MzA1REMifciFfvQTrNYBSqCv/qWJHasZvX1uYu77E3mhQrAKBmepvMmojeMQDNb5no7JlxtpJ81f0qe4bPp3GgPRLoHrl2FinqKgY/yKvQ76mwdx1SjgC1IpEaHjE/sYAEVeKgpatwudKtdpS3y9+GZEWhAkhg/cgBGaL9Y3CF0ka1vDitLJRpWf9Zgto7tci9ZawODlK/+OmE6n3u4DMvaNiIAKVwhPzc48LbzePADh2DSuzOKpX/65CwxpCmkDjWm5yBsWp9qQSZW544blvEWHCZ9A78LW3vWsLxGIsuLjyWSkdFDGSUzg1dW2newYP1XIFyVLNoytcb6AlqAhkDlOYjbYSQwv/PNivr47rjT0CXon5zTkzotQ9rTOCEpIzyohKV16Nz+x4CyL1dxv51n13fz5FZ3BlgO69fGkB1AegsuUER5I+REjBt89kwhC3LjXNRMIKE2PwFeJ8p4NY2IxqOtdxBTGsRROMTbviciydft7clbV/0m839sOGMT2dhLb7xqeudJ59X5fGeFQoT19zYJZksXXZVSCoPCoNT2X7GRJg1PWVE5m+f0FCe5lH+39aErieBmdiUpe2BKpk2I+7odXAh22v6l90o7ZMnehuwAIYIiRg469az2S3M9bTSeB",
       "paymentMethodType": "paypal",
       "sdkData": {
         "token": "EC-7CT52962Y8151653E"
       },
       "resendInterval": 0,
       "resendMaxAttempts": 0,
       "type": "sdk"
     }
   }
    ```

 **Complete payment `POST /payments/details`**
    **Request Body**:
```json
{
  "details": {
    "orderID": "EC-7CT52962Y8151653E",
    "payerID": "AAF597AAMG8ML",
    "paymentID": null,
    "billingToken": null,
    "facilitatorAccessToken": "A21AAJyR8ShqyEHao1PwkOSBgqpw4tKx8JCxE8wFqe3E2ccKIkSC9QS7TO4cSr8Dqe5DKONEtO5X_oMa56IzSuikFjHkAZFNA"
  },
  "paymentData": "Ab02b4c0!BQABAgAzBYGIu/mZsZgHzZrtSCvB7RtNbChgr7fduPqpGtQJJzswHbj8nKiNmWfMLXyTS9KtWElwzSfPbmPwmCZ/PiJKVzGNJLD08279PfIrChFge5MwYnBbjtcjRWI6Pllf/vTlv2s9wZzgRtP8XoRWdULD4HKc12LqO3Y/2hq8IKSyAeq66nzHaR3OZt6Hmo+FIDRDn+tkfpvipamU/yqUXog8aNuxrCwAdcTNF9ctFeFzSQza3dmQh84WrGqiqRurjVu6rCnjEj/5qxQXDjrqVm0IHnvSerjTlGXtXri7wG/jpTYLjkhK2DsI5gs/TFc2iqk9zkszKQUWDVIVQKL4vcmp41AI21HJaeLC+dl1WoZHkLg/JbUgsAgoltqdU1fATkABw/M+CtlJmHAB2pUS73g6Q8Fhospv4G2ku+QRlqOiZk2MixPckW6LmKS8yuuQCniw+jhdbmQgSAAPl6iqQNZHiwHnd6ddAucou8nMflIRv9S5kPNA7I/jcQNrV20qu+BIvw0WAS2zTdJNl2uZu1jX89vfe8vcxid4ycu6Aew6OzOxcr+CZpGuUCVKGFVuFyXXY1pL4YYPBDGEZE91FJHCKNTbeF6qrM9Qw4wKzi6SOyqAJGZC8iIQM4kRB79VQiVBGEAAK6Khy0RgXzwl3hSX5zlc5GU5t57m+vfULufrnBALyV5mV/hjXLaRumAufDnwAEp7ImtleSI6IkFGMEFBQTEwM0NBNTM3RUFFRDg3QzI0REQ1MzkwOUI4MEE3OEE5MjNFMzgyM0Q2OERBQ0M5NEI5RkY4MzA1REMifciFfvQTrNYBSqCv/qWJHasZvX1uYu77E3mhQrAKBmepvMmojeMQDNb5no7JlxtpJ81f0qe4bPp3GgPRLoHrl2FinqKgY/yKvQ76mwdx1SjgC1IpEaHjE/sYAEVeKgpatwudKtdpS3y9+GZEWhAkhg/cgBGaL9Y3CF0ka1vDitLJRpWf9Zgto7tci9ZawODlK/+OmE6n3u4DMvaNiIAKVwhPzc48LbzePADh2DSuzOKpX/65CwxpCmkDjWm5yBsWp9qQSZW544blvEWHCZ9A78LW3vWsLxGIsuLjyWSkdFDGSUzg1dW2newYP1XIFyVLNoytcb6AlqAhkDlOYjbYSQwv/PNivr47rjT0CXon5zTkzotQ9rTOCEpIzyohKV16Nz+x4CyL1dxv51n13fz5FZ3BlgO69fGkB1AegsuUER5I+REjBt89kwhC3LjXNRMIKE2PwFeJ8p4NY2IxqOtdxBTGsRROMTbviciydft7clbV/0m839sOGMT2dhLb7xqeudJ59X5fGeFQoT19zYJZksXXZVSCoPCoNT2X7GRJg1PWVE5m+f0FCe5lH+39aErieBmdiUpe2BKpk2I+7odXAh22v6l90o7ZMnehuwAIYIiRg469az2S3M9bTSeB",
  "paymentId": "50f9f5fe-2930-49ba-ba81-04572d9fea43"
}
  ```
   **Response Body**
```json
{
  "paymentId": "50f9f5fe-2930-49ba-ba81-04572d9fea43",
  "pspReference": "882609832254653C",
  "status": "Pending"
}
 ```


