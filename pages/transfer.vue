<template>
  <section>
    <Loader v-model="isLoading" :status="status" />
    <nuxt-link
      v-if="$route.query.target"
      :to="`/rmrk/u/${destinationAddress}`"
      class="linkartist">
      <b-icon icon="chevron-left" size="is-small" class="linkartist--icon" />
      Go to artist's profile
    </nuxt-link>
    <div class="box">
      <p class="title is-size-3">
        Transfer {{ unit }}
        <span v-if="isKSM" class="has-text-primary"
          >${{ $store.getters['fiat/getCurrentKSMValue'] }}</span
        >
      </p>
      <b-field>
        <Auth />
      </b-field>
      <div v-if="$route.query.target" class="box--target-info">
        Your donation will be sent to:
        <a
          :href="`https://kusama.subscan.io/account/${$route.query.target}`"
          target="_blank"
          rel="noopener noreferrer"
          class="box--target-info--url">
          <Identity
            ref="identity"
            :address="$route.query.target"
            inline
            show-onchain-identity />
        </a>
      </div>

      <div class="is-flex is-align-items-center">
        <b-field>
          {{ $t('general.balance') }}
          <Money :value="balance" inline />
        </b-field>
      </div>

      <b-field>
        <AddressInput v-model="destinationAddress" :strict="false" />
      </b-field>
      <DisabledInput
        v-show="correctAddress && correctAddress !== destinationAddress"
        :label="$t('general.correctAddress')"
        :value="correctAddress" />
      <div class="box--container mb-3">
        <b-field>
          <BalanceInput
            v-model="price"
            label="Amount"
            :calculate="false"
            @input="onAmountFieldChange" />
        </b-field>
        <b-field v-if="isKSM">
          <ReadOnlyBalanceInput
            v-model="usdValue"
            label-input="USD Value (approx)"
            label="USD"
            @input="onUSDFieldChange" />
        </b-field>
      </div>

      <div class="buttons">
        <b-button
          type="is-primary"
          icon-left="paper-plane"
          :loading="isLoading"
          :disabled="disabled"
          outlined
          @click="submit">
          {{ $t('general.submit') }}
        </b-button>
        <b-button
          v-if="transactionValue"
          type="is-success"
          class="tx"
          icon-left="external-link-alt"
          outlined
          @click="getExplorerUrl">
          {{ $t('View Transaction') }} {{ transactionValue.substring(0, 6)
          }}{{ '...' }}
        </b-button>
        <b-button
          v-if="transactionValue"
          @click="toast('URL copied to clipboard')"
          v-clipboard:copy="getUrl()"
          type="is-primary">
          <b-icon size="is-small" pack="fas" icon="link" />
        </b-button>
        <b-button
          v-if="destinationAddress"
          type="is-success"
          icon-left="money-bill"
          :loading="isLoading"
          @click="toast('Payment link copied to clipboard')"
          v-clipboard:copy="generatePaymentLink()"
          outlined>
          {{ $t('Copy Payment link') }}
        </b-button>
      </div>
      <div v-if="transactionValue && this.$route.query.donation">
        <div class="is-size-5">
          🎉 Congratulations for supporting
          <Identity ref="identity" :address="this.$route.query.target" inline />
        </div>
        <b-button
          type="is-info"
          class="tweetBtn"
          icon-left="share-square"
          outlined
          @click="shareInTweet">
          {{ $t('Tweet about your awesome donation') }}
        </b-button>
      </div>
    </div>
  </section>
</template>

<script lang="ts">
import { Component, mixins, Watch } from 'nuxt-property-decorator'
import Connector from '@kodadot1/sub-api'
import exec, { execResultValue, txCb } from '@/utils/transactionExecutor'
import { notificationTypes, showNotification } from '@/utils/notification'
import TransactionMixin from '@/utils/mixins/txMixin'
import AuthMixin from '@/utils/mixins/authMixin'
import shouldUpdate from '@/utils/shouldUpdate'
import ChainMixin from '@/utils/mixins/chainMixin'
import { DispatchError } from '@polkadot/types/interfaces'
import { calculateBalance } from '@/utils/formatBalance'
import correctFormat from '@/utils/ss58Format'
import { encodeAddress, isAddress } from '@polkadot/util-crypto'
import { urlBuilderTransaction } from '@/utils/explorerGuide'
import { calculateUsdFromKsm, calculateKsmFromUsd } from '@/utils/calculation'
import onApiConnect from '~/utils/api/general'
import type { ApiPromise } from '@polkadot/api'
import Query from '@/utils/api/Query'

@Component({
  components: {
    Auth: () => import('@/components/shared/Auth.vue'),
    BalanceInput: () => import('@/components/shared/BalanceInput.vue'),
    ReadOnlyBalanceInput: () =>
      import('@/components/shared/ReadOnlyBalanceInput.vue'),
    Identity: () => import('@/components/shared/format/Identity.vue'),
    Loader: () => import('@/components/shared/Loader.vue'),
    AddressInput: () => import('@/components/shared/AddressInput.vue'),
    Money: () => import('@/components/shared/format/Money.vue'),
    DisabledInput: () => import('@/components/shared/DisabledInput.vue'),
  },
})
export default class Transfer extends mixins(
  TransactionMixin,
  AuthMixin,
  ChainMixin
) {
  protected balance = '0'
  protected destinationAddress = ''
  protected transactionValue = ''
  protected price = 0
  protected usdValue = 0

  layout() {
    return 'centered-half-layout'
  }

  get disabled(): boolean {
    return !this.hasAddress || !this.price || !this.accountId
  }
  get ss58Format(): number {
    return this.chainProperties?.ss58Format
  }
  get hasAddress(): boolean {
    return isAddress(this.destinationAddress)
  }
  get correctAddress(): string {
    return this.hasAddress
      ? encodeAddress(this.destinationAddress, correctFormat(this.ss58Format))
      : ''
  }

  get isKSM(): boolean {
    return this.unit === 'KSM'
  }

  protected created() {
    this.$store.dispatch('fiat/fetchFiatPrice')
    this.checkQueryParams()
    onApiConnect(async (api) => {
      this.loadBalance(api)
    })
  }

  protected onAmountFieldChange() {
    /* calculating usd value on the basis of price entered */
    if (this.price) {
      this.usdValue = calculateUsdFromKsm(
        this.$store.getters['fiat/getCurrentKSMValue'],
        this.price
      )
    } else {
      this.usdValue = 0
    }
  }

  protected onUSDFieldChange() {
    /* calculating price value on the basis of usd entered */
    if (this.usdValue) {
      this.price = calculateKsmFromUsd(
        this.$store.getters['fiat/getCurrentKSMValue'],
        this.usdValue
      )
    } else {
      this.price = 0
    }
  }

  protected checkQueryParams() {
    const { query } = this.$route
    if (query.target) {
      const hasAddress = isAddress(query.target as string)
      if (hasAddress) {
        this.destinationAddress = query.target as string
      } else {
        showNotification('Unable to use target address', notificationTypes.warn)
      }
    }

    if (query.amount) {
      this.price = Number(query.amount)
    }

    if (query.usdamount) {
      this.usdValue = Number(query.usdamount)
      // getting ksm value from the usd value
      this.price = calculateKsmFromUsd(
        this.$store.getters['fiat/getCurrentKSMValue'],
        this.usdValue
      )
    }
  }

  public async submit(): Promise<void> {
    showNotification(
      `${this.$route.query.target ? 'Sent for Sign' : 'Dispatched'}`
    )
    this.initTransactionLoader()

    try {
      const { api } = Connector.getInstance()
      const cb = api.tx.balances.transfer
      const arg = [
        this.destinationAddress,
        calculateBalance(this.price, this.decimals),
      ]

      const tx = await exec(
        this.accountId,
        '',
        cb,
        arg,
        txCb(
          async (blockHash) => {
            this.transactionValue = execResultValue(tx)
            const header = await api.rpc.chain.getHeader(blockHash)
            const blockNumber = header.number.toString()

            showNotification(
              `[${this.unit}] Transfered ${this.price} ${this.unit} in block ${blockNumber}`,
              notificationTypes.success
            )

            this.destinationAddress = ''
            this.price = 0
            this.usdValue = 0
            if (this.$route.query && !this.$route.query.donation) {
              this.$router.push(this.$route.path)
            }

            this.isLoading = false
          },
          (dispatchError) => {
            execResultValue(tx)
            this.onTxError(dispatchError)
            this.isLoading = false
          },
          (res) => this.resolveStatus(res.status)
        )
      )
    } catch (e) {
      console.error('[ERR: TRANSFER SUBMIT]', e)
      if (e instanceof Error) {
        showNotification(e.message, notificationTypes.danger)
      }
    }
  }

  protected onTxError(dispatchError: DispatchError): void {
    const { api } = Connector.getInstance()
    if (dispatchError.isModule) {
      const decoded = api.registry.findMetaError(dispatchError.asModule)
      const { docs, name, section } = decoded
      showNotification(
        `[ERR] ${section}.${name}: ${docs.join(' ')}`,
        notificationTypes.danger
      )
    } else {
      showNotification(
        `[ERR] ${dispatchError.toString()}`,
        notificationTypes.danger
      )
    }

    this.isLoading = false
  }

  protected getUrl(): string {
    return urlBuilderTransaction(
      this.transactionValue,
      this.$store.getters['explorer/getCurrentChain'],
      'subscan'
    )
  }

  protected getExplorerUrl(): void {
    const url = this.getUrl()
    window.open(url, '_blank')
  }

  protected generatePaymentLink(): string {
    return `${window.location.origin}/transfer?target=${this.destinationAddress}&usdamount=${this.usdValue}&donation=true`
  }

  protected shareInTweet() {
    const text =
      'I have just helped a really cool creator by donating. Check my donation proof:'
    const url = `https://twitter.com/intent/tweet?text=${text}&via=KodaDot&url=${this.getUrl()}`
    window.open(url, '_blank')
  }

  @Watch('accountId', { immediate: true })
  hasAccount(value: string, oldVal: string): void {
    if (shouldUpdate(value, oldVal)) {
      this.loadBalance()
    }
  }

  @Watch('destinationAddress')
  destinationChanged(target: string): void {
    const { usdamount } = this.$route.query
    this.$router.replace({ query: { target, usdamount } }).catch(() => null) // null to further not throw navigation errors
  }

  @Watch('usdValue')
  usdValueChanged(usdamount: string): void {
    const { target } = this.$route.query
    this.$router.replace({ query: { target, usdamount } }).catch(() => null) // null to further not throw navigation errors
  }

  async loadBalance(
    api: ApiPromise = Connector.getInstance().api
  ): Promise<void> {
    if (!this.accountId || !this.unit || !api) {
      return
    }

    try {
      this.balance = await Query.getTokenBalance(api, this.accountId)
    } catch (e) {
      console.error('[ERR: BALANCE]', e)
    }
  }

  private toast(message: string): void {
    this.$buefy.toast.open(message)
  }
}
</script>

<style scoped lang="scss">
@import '@/styles/variables';
.tx {
  margin-left: 1rem;
}
.tweetBtn {
  margin-top: 0.5rem;
}
.box {
  &--container {
    display: flex;
    justify-content: space-between;
    @media screen and (max-width: 1023px) {
      flex-direction: column;
    }
  }
  &--target-info {
    margin-bottom: 0.8rem;
    &--url {
      font-weight: bold;
    }
  }
}
.linkartist {
  padding-left: 1.25rem;
  display: flex;
  align-items: center;
  &--icon {
    margin-right: 0.5rem;
  }
}
</style>
