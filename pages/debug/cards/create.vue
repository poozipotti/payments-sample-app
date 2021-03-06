<template>
  <v-layout>
    <v-row>
      <v-col cols="12" md="4">
        <small v-if="isSandbox">
          Please see the
          <a
            href="https://developers.circle.com/docs/test-card-numbers"
            ref="noopener"
            target="_blank"
          >
            api docs
          </a>
          on how to trigger error responses.
        </small>
        <v-menu>
          <template v-slot:activator="{ on }">
            <div class="d-flex flex-row-reverse">
              <v-btn
                v-if="isSandbox"
                small
                color="blue-grey lighten-1"
                dark
                v-on="on"
              >
                Prefill form
              </v-btn>
            </div>
          </template>
          <v-list>
            <v-list-item
              v-for="(item, index) in prefillItems"
              :key="index"
              @click="prefillForm(index)"
            >
              <v-list-item-title>{{ item.title }}</v-list-item-title>
            </v-list-item>
          </v-list>
        </v-menu>

        <v-form>
          <v-text-field v-model="formData.cardNumber" label="Card Number" />

          <v-select
            v-model="formData.verification"
            :items="verificationMethods"
            label="Verification Method"
          />

          <v-text-field
            v-if="showCVVField"
            v-model="formData.cvv"
            label="CVV"
          />

          <v-text-field v-model="formData.expiry.month" label="Expiry Month" />

          <v-text-field v-model="formData.expiry.year" label="Expiry Year" />

          <v-text-field v-model="formData.name" label="Full name" />

          <v-text-field v-model="formData.line1" label="Address Line 1" />

          <v-text-field v-model="formData.line2" label="Address Line 2" />

          <v-text-field v-model="formData.postalCode" label="Postalcode" />

          <v-text-field v-model="formData.city" label="City" />

          <v-text-field v-model="formData.district" label="District" />

          <v-text-field v-model="formData.country" label="Country Code" />

          <v-text-field v-model="formData.phoneNumber" label="Phone" />

          <v-text-field v-model="formData.email" label="Email" />

          <v-btn
            depressed
            color="primary"
            :loading="loading"
            @click.prevent="makeApiCall()"
          >
            Make api call
          </v-btn>
        </v-form>
      </v-col>
      <v-col cols="12" md="8">
        <RequestInfo
          :url="requestUrl"
          :payload="payload"
          :response="response"
        />
      </v-col>
    </v-row>
    <ErrorSheet
      :error="error"
      :show-error="showError"
      @onChange="onErrorSheetClosed"
    />
  </v-layout>
</template>

<script lang="ts">
import { Component, Vue } from 'nuxt-property-decorator'
import { mapGetters } from 'vuex'
import uuidv4 from 'uuid/v4'
import openPGP from '@/lib/openpgp'
import { getLive } from '@/lib/apiTarget'
import { exampleCards } from '@/lib/cardTestData'
import { CreateCardPayload } from '@/lib/cardsApi'
import RequestInfo from '@/components/RequestInfo.vue'
import ErrorSheet from '@/components/ErrorSheet.vue'
import CardInput from '@/components/CardInput.vue'
import CVVInput from '@/components/CVVInput.vue'
import ExpiryInput from '@/components/ExpiryInput.vue'

@Component({
  components: {
    RequestInfo,
    ErrorSheet,
    CardInput,
    ExpiryInput,
    CVVInput
  },
  computed: {
    ...mapGetters({
      payload: 'getRequestPayload',
      response: 'getRequestResponse',
      requestUrl: 'getRequestUrl'
    })
  }
})
export default class CreateCardClass extends Vue {
  // data
  formData = {
    cardNumber: '',
    cvv: '',
    expiry: {
      month: '',
      year: ''
    },
    name: '',
    country: '',
    district: '',
    line1: '',
    line2: '',
    city: '',
    postalCode: '',
    verification: 'cvv',
    phoneNumber: '',
    email: ''
  }
  verificationMethods = ['none', 'cvv']
  rules = {
    isNumber: (v: string) =>
      v === '' || !isNaN(parseInt(v)) || 'Please enter valid number',
    required: (v: string) => !!v || 'Field is required'
  }
  prefillItems = exampleCards
  error = {}
  loading = false
  showError = false
  expiryLabels = {
    month: 'Expiry Month',
    year: 'Expiry Year'
  }
  isSandbox: Boolean = !getLive()

  onErrorSheetClosed() {
    this.error = {}
    this.showError = false
  }

  prefillForm(index: number) {
    this.formData = exampleCards[index].formData
  }

  get showCVVField() {
    return this.formData.verification === 'cvv'
  }

  async makeApiCall() {
    this.loading = true

    const { email, phoneNumber, cardNumber, cvv, ...data } = this.formData
    const { expiry, verification, ...billingDetails } = data

    let cardDetails: {
      number: string
      cvv?: string
    } = {
      number: cardNumber.trim().replace(/\D/g, '')
    }

    if (verification === 'cvv') {
      cardDetails = {
        ...cardDetails,
        cvv
      }
    }

    const payload: CreateCardPayload = {
      idempotencyKey: uuidv4(),
      expMonth: parseInt(expiry.month),
      expYear: 2000 + parseInt(expiry.year),
      verification,
      keyId: '',
      encryptedData: '',
      billingDetails,
      metadata: {
        email,
        phoneNumber,
        sessionId: 'xxx',
        ipAddress: '172.33.222.1'
      }
    }

    try {
      if (verification === 'cvv' || verification === 'avs') {
        const publicKey = await this.$cardsApi.getPCIPublicKey()
        const encryptedData = await openPGP.encrypt(cardDetails, publicKey)
        const { encryptedMessage, keyId } = encryptedData

        payload.keyId = keyId
        payload.encryptedData = encryptedMessage
      }

      await this.$cardsApi.createCard(payload)
    } catch (error) {
      this.error = error
      this.showError = true
    } finally {
      this.loading = false
    }
  }
}
</script>
