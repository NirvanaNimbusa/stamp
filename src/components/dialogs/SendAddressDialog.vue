<template>
  <q-card class="q-px-sm q-pb-md dialog-medium">
    <q-card-section>
      <div class="text-h6">
        {{ $t('sendAddressDialog.sendToAddress') }}
      </div>
    </q-card-section>
    <q-card-section>
      <q-input
        class="text-bold text-h6"
        v-model="address"
        filled
        dense
        :placeholder="$t('sendAddressDialog.enterBitcoinCashAddress')"
        ref="address"
      />
    </q-card-section>
    <q-card-section>
      <q-input
        class="text-bold text-h6"
        v-model="amount"
        type="number"
        filled
        dense
        :placeholder="$t('sendAddressDialog.enterAmount')"
      />
    </q-card-section>
    <q-card-actions align="right">
      <q-btn
        flat
        :label="$t('sendAddressDialog.cancel')"
        color="primary"
        v-close-popup
      />
      <q-btn
        flat
        :disable="!isValid"
        :label="$t('sendAddressDialog.send')"
        color="primary"
        @click="send()"
        v-close-popup
      />
    </q-card-actions>
  </q-card>
</template>

<script>
import { sentTransactionNotify, errorNotify } from '../../utils/notifications'

import { Address, Transaction, Script } from 'bitcore-lib-cash'

export default {
  components: {
  },
  data () {
    return {
      address: '',
      amount: null
    }
  },
  computed: {
    isValid () {
      if (!this.amount) {
        return false
      }
      try {
        Address(this.address, 'testnet') // TODO: Make this generic
        return true
      } catch {
        return false
      }
    }
  },
  methods: {
    async send () {
      try {
        const output = new Transaction.Output({
          script: Script(new Address(this.address)),
          satoshis: this.amount
        })

        const { transaction, usedIDs } = await this.$wallet.constructTransaction({ outputs: [output] })
        const txHex = transaction.toString()

        try {
          const electrumHandler = await this.$electrumClientPromise
          await electrumHandler.request('blockchain.transaction.broadcast', txHex)
          sentTransactionNotify()
          console.log('Sent transaction', txHex)
          // TODO: we shouldn't be dealing with this here. Leaky abstraction
          await Promise.all(usedIDs.map(id => this.$wallet.storage.deleteOutpoint(id)))
        } catch (err) {
          usedIDs.forEach(id => {
            this.$wallet.unfreezeUTXO(id)
          })
          throw err
        }
      } catch (err) {
        console.error(err)
        errorNotify(new Error('Failed to send transaction'))
        // Unfreeze UTXOs if stealth tx broadcast fails
      }
    }
  },
  mounted () {
    this.$refs.address.$el.focus()
  }
}
</script>
