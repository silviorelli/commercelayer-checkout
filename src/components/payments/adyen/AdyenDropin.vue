<template>
  <div class="payment-method">
    <v-radio
      :label="inputLabel('adyen')"
      :value="payment_option.component"
      color="primary"
      @change="setPaymentMethod"
      id="adyen-dropin-radio"
    ></v-radio>
    <div class="payment-method-fields" v-show="selected">
      <p>DEBUG: Adyen Dropin</p>
      <div id="adyen-dropin"></div>
      <div class="payment-error" id="adyen-dropin-error"></div>
    </div>
    <div id="adyen-action"></div>
  </div>
</template>

<script>
import { paymentMixin } from '@/mixins/paymentMixin'
import { collectBrowserInfo } from '@/utils/functions'

// DEBUG TODO remove
const paymentMethodMock = {
  "groups": [
    {
      "name": "Credit Card",
      "types": [
        "mc",
        "visa",
        "amex",
        "maestro",
        "amex_applepay",
        "cup",
        "diners",
        "discover",
        "jcb",
        "mc_applepay"
      ]
    }
  ],
  "paymentMethods": [
    {
      "details": [
        {
          "key": "number",
          "type": "text"
        },
        {
          "key": "expiryMonth",
          "type": "text"
        },
        {
          "key": "expiryYear",
          "type": "text"
        },
        {
          "key": "cvc",
          "type": "text"
        },
        {
          "key": "holderName",
          "optional": true,
          "type": "text"
        }
      ],
      "name": "Credit Card",
      "type": "scheme"
    },
    {
      "details": [
        {
          "items": [
            {
              "id": "1121",
              "name": "Test Issuer"
            },
            {
              "id": "1154",
              "name": "Test Issuer 5"
            },
            {
              "id": "1153",
              "name": "Test Issuer 4"
            },
            {
              "id": "1152",
              "name": "Test Issuer 3"
            },
            {
              "id": "1151",
              "name": "Test Issuer 2"
            },
            {
              "id": "1162",
              "name": "Test Issuer Cancelled"
            },
            {
              "id": "1161",
              "name": "Test Issuer Pending"
            },
            {
              "id": "1160",
              "name": "Test Issuer Refused"
            },
            {
              "id": "1159",
              "name": "Test Issuer 10"
            },
            {
              "id": "1158",
              "name": "Test Issuer 9"
            },
            {
              "id": "1157",
              "name": "Test Issuer 8"
            },
            {
              "id": "1156",
              "name": "Test Issuer 7"
            },
            {
              "id": "1155",
              "name": "Test Issuer 6"
            }
          ],
          "key": "issuer",
          "type": "select"
        }
      ],
      "name": "iDEAL",
      "supportsRecurring": true,
      "type": "ideal"
    },
    {
      "name": "PayPal",
      "supportsRecurring": true,
      "type": "paypal"
    },
    {
      "details": [
        {
          "key": "sepa.ownerName",
          "type": "text"
        },
        {
          "key": "sepa.ibanNumber",
          "type": "text"
        }
      ],
      "name": "SEPA Direct Debit",
      "supportsRecurring": true,
      "type": "sepadirectdebit"
    },
    {
      "name": "UnionPay",
      "supportsRecurring": true,
      "type": "unionpay"
    }
  ]
}

export default {
  computed: {
    scriptSrc () {
      return `https://checkoutshopper-${process.env.VUE_APP_ADYEN_ENV}.adyen.com/checkoutshopper/sdk/3.8.1/adyen.js`
    },
    styleHref () {
      return `https://checkoutshopper-${process.env.VUE_APP_ADYEN_ENV}.adyen.com/checkoutshopper/sdk/3.8.1/adyen.css`
    },
    styleObj () {
      return {
        base: {
          fontSize: '16px',
          fontFamily: 'Roboto, sans-serif'
        },
        error: {
          color: process.env.VUE_APP_ERROR_COLOR
        }
      }
    },
    checkoutConfig () {
      console.log('DEBUG AdyenCard this.order.payment_source:', this.order.payment_source)
      // console.log('DEBUG this.order.payment_source.payment_methods', this.order.payment_source.payment_methods)

      return {
        locale: this.$i18n.locale,
        environment: process.env.VUE_APP_ADYEN_ENV,
        originKey: process.env.VUE_APP_ADYEN_ORIGIN_KEY,
        // paymentMethodsResponse: this.order.payment_source.payment_methods,
        paymentMethodsResponse: paymentMethodMock,
        // onChange: this.handleOnChange,
        onSubmit: this.handleOnChange,
        onAdditionalDetails: this.handleOnAdditionalDetails
      }
    }
  },
  mixins: [paymentMixin],
  methods: {
    setupPayment () {
      let script = this.getScript(this.scriptSrc)
      script.addEventListener('load', () => {
        // eslint-disable-next-line
        let checkout = new AdyenCheckout(this.checkoutConfig)
        checkout
          .create('dropin', {
            styles: this.styleObj
          })
          .mount('#adyen-dropin')

        let btn = document.getElementById('payment-step-submit')
        btn.onclick = () => {
          this.handlePayment(checkout)
        }
      })
    },
    handleOnChange (state, component) {
      if (state.isValid) {
        let browserInfo = collectBrowserInfo()

        this.$store.dispatch('updateOrderPaymentSource', {
          payment_request_data: {
            payment_method: state.data.paymentMethod,
            origin: window.location.origin,
            return_url: window.location.href,
            browser_info: {
              acceptHeader:
                'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8',
              colorDepth: browserInfo.colorDepth,
              javaEnabled: browserInfo.javaEnabled,
              language: browserInfo.language,
              screenHeight: browserInfo.screenHeight,
              screenWidth: browserInfo.screenWidth,
              timeZoneOffset: browserInfo.timeZoneOffset,
              userAgent: browserInfo.userAgent
            }
          }
        })
      }
    },
    handleOnAdditionalDetails (state, component) {
      this.$store
        .dispatch('updateOrderPaymentSource', {
          payment_request_details: state.data,
          _details: 1
        })
        .then(paymentSource => {
          // eslint-disable-next-line
          let checkout = new AdyenCheckout(this.checkoutConfig)
          this.handlePaymentResponse(paymentSource.payment_response, checkout)
        })
    },
    handlePayment (checkout) {
      this.loading_payment = true
      this.$store
        .dispatch('updateOrderPaymentSource', {
          _authorize: 1
        })
        .then(paymentSource => {
          this.handlePaymentResponse(paymentSource.payment_response, checkout)
        })
    },
    handlePaymentResponse (paymentResponse, checkout) {
      if (paymentResponse.action !== undefined) {
        // https://docs.adyen.com/checkout/components-web#step-4-additional-front-end
        checkout.createFromAction(paymentResponse.action).mount('#adyen-action')
      } else {
        // https://docs.adyen.com/checkout/components-web#step-6-present-payment-result
        switch (paymentResponse.resultCode) {
          case 'Authorised':
          case 'Pending':
          case 'Received':
            this.$store.dispatch('placeOrder')
            break
          case 'Error':
          case 'Refused':
            let cardError = document.getElementById('adyen-card-error')
            cardError.innerHTML = this.$t('errors.not_authorized')
            this.loading_payment = false
            break
        }
      }
    },
    checkStyle () {
      let links = document.getElementsByTagName('link')
      for (let i = 0; i < links.length; i++) {
        if (links[i].href === this.styleHref) return true
      }
      return false
    }
  },
  mounted () {
    // Add Adyen.css to document head
    if (!this.checkStyle()) {
      let style = document.createElement('link')
      style.rel = 'stylesheet'
      style.href = this.styleHref
      document.head.appendChild(style)
    }
  }
}
</script>

<style lang="scss">
.adyen-checkout__label__text {
  font-size: 1rem !important;
  margin-bottom: 0.5rem;
}
.adyen-checkout__input {
  @include hosted-field;
}
#adyen-action {
  .adyen-checkout__threeds2__challenge {
    border: 1px solid $v-border;
  }
  iframe[name='threeDSIframe'] {
    padding: 2rem;
  }
}
</style>
